# Baremetal DL380 Fencing

Bu dokuman HPE ProLiant DL380 baremetal sunucularda Pacemaker STONITH icin iLO tabanli fencing hazirligini anlatir.

## Network

Mevcut lab tasariminda fencing/BMC erisimi management network uzerinden yapilabilir:

```text
192.168.88.0/24 -> Ansible, client, VIP, vCenter veya iLO/BMC erisimi
172.12.2.0/16   -> Corosync private heartbeat
172.22.2.0/16   -> iSCSI storage
```

Baremetal test icin iLO portlari `192.168.88.0/24` tarafindan erisilebilir olmalidir.

Corosync VLAN fencing icin kullanilmaz.
iSCSI VLAN fencing icin kullanilmaz.

## Onerilen Agent

Ilk tercih:

```text
fence_ipmilan
```

Sebep:

- HPE iLO ile yaygin ve basit calisir.
- UDP 623/IPMI uzerinden power control yapar.
- DL380 lab testleri icin yeterli ve okunabilir.

Alternatif:

```text
fence_redfish
```

Redfish HTTPS 443 uzerinden calisir. iLO firmware ve Redfish URI farklarina gore ekstra ayar gerekebilir.

## HPE iLO IPMI Hazirligi

iLO tarafinda:

```text
1. Her sunucu icin iLO IP ver.
2. Cluster node'larindan iLO IP'lerine erisim sagla.
3. Fencing icin kullanici olustur.
4. Kullaniciya server power control yetkisi ver.
5. IPMI/DCMI over LAN aktif olsun.
```

Node'lardan erisim testi:

```bash
ansible -i inventory/lab.ini ha_cluster -m command -a "nc -zvu ILO_IP 623"
```

`nc` yoksa:

```bash
ansible -i inventory/lab.ini ha_cluster -m command -a "ping -c 2 ILO_IP"
```

## Degiskenler

VMware fencing'i kapatip HPE iLO IPMI profilini ac:

```yaml
vmware_fencing:
  enabled: false

hpe_ilo_fencing:
  enabled: true
  agent: fence_ipmilan
  username: Administrator
  password: "{{ vault_hpe_ilo_fencing_password }}"
  lanplus: true
  method: onoff
  privlvl: administrator
  nodes:
    sds-frs-n01:
      bmc_ip: 192.168.88.201
    sds-frs-n02:
      bmc_ip: 192.168.88.202
```

Vault:

```yaml
vault_hpe_ilo_fencing_password: "ChangeMe"
```

Sonra:

```bash
ansible-playbook -i inventory/lab.ini playbooks/40_fencing.yml
```

Kontrol:

```bash
ansible -i inventory/lab.ini sds-frs-n01 -m shell -a "pcs stonith config | sed -E 's/(password=)[^ ]+/\\1****/g'"
ansible -i inventory/lab.ini sds-frs-n01 -m command -a "pcs status --full"
```

## Redfish Alternatifi

Redfish kullanilacaksa:

```yaml
vmware_fencing:
  enabled: false

redfish_fencing:
  enabled: true
  agent: fence_redfish
  username: Administrator
  password: "{{ vault_redfish_fencing_password }}"
  ssl_insecure: true
  systems_uri: /redfish/v1/Systems/1
  nodes:
    sds-frs-n01:
      bmc_ip: 192.168.88.201
    sds-frs-n02:
      bmc_ip: 192.168.88.202
```

Vault:

```yaml
vault_redfish_fencing_password: "ChangeMe"
```

Redfish endpoint testi:

```bash
ansible -i inventory/lab.ini ha_cluster -m command -a "curl -k -I https://ILO_IP/redfish/v1"
```

## Dikkat Edilecekler

- Her node kendi karsilik gelen iLO/BMC IP'si ile eslenmelidir.
- `pcmk_host_list` node adini gostermelidir.
- Fencing testleri VM veya fiziksel sunucuyu reboot/power cycle edebilir.
- Shared storage aktifken STONITH olmadan failover test edilmemelidir.
- Gercek fence testi sadece bakim penceresinde yapilmalidir.

## Test Komutlari

Agent metadata:

```bash
ansible -i inventory/lab.ini sds-frs-n01 -m command -a "pcs stonith describe fence_ipmilan"
ansible -i inventory/lab.ini sds-frs-n01 -m command -a "pcs stonith describe fence_redfish"
```

STONITH status:

```bash
ansible -i inventory/lab.ini sds-frs-n01 -m command -a "pcs stonith status"
```

Manuel fence aksiyonu:

```bash
ansible -i inventory/lab.ini sds-frs-n01 -m command -a "stonith_admin --reboot sds-frs-n02"
```

Bu komut hedef node'u reboot eder.

Test sonrasi:

```bash
ansible -i inventory/lab.ini sds-frs-n01 -m command -a "pcs status --full"
ansible-playbook -i inventory/lab.ini playbooks/90_validation.yml
```

