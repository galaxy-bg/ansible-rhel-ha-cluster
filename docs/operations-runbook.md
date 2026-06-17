# Operations Runbook

Bu runbook kurulum sonrasi gunluk kontrol, troubleshooting ve kontrollu failover testleri icin Ansible ad-hoc komutlarini toplar.

Komutlar laptop/control node uzerinden calistirilir:

```bash
cd /Users/bilginozturk/Desktop/KronosDX-Projeler/ansible-rhel-ha-cluster
```

## Hizli Saglik Kontrolu

Ansible erisimi:

```bash
ansible -i inventory/lab.ini ha_cluster -m ping
```

Cluster status:

```bash
ansible -i inventory/lab.ini sds-frs-n01 -m command -a "pcs status --full"
```

Corosync ring durumu:

```bash
ansible -i inventory/lab.ini ha_cluster -m command -a "corosync-cfgtool -s"
```

Cluster membership:

```bash
ansible -i inventory/lab.ini ha_cluster -m command -a "crm_node -l"
```

Servisler:

```bash
ansible -i inventory/lab.ini ha_cluster -m command -a "systemctl is-active corosync pacemaker pcsd iscsid multipathd"
```

Validation playbook:

```bash
ansible-playbook -i inventory/lab.ini playbooks/90_validation.yml
```

## Resource ve STONITH Kontrolu

Resource group:

```bash
ansible -i inventory/lab.ini sds-frs-n01 -m command -a "pcs resource config"
```

STONITH config:

```bash
ansible -i inventory/lab.ini sds-frs-n01 -m shell -a "pcs stonith config | sed -E 's/(password=)[^ ]+/\\1****/g'"
```

Cluster property:

```bash
ansible -i inventory/lab.ini sds-frs-n01 -m command -a "pcs property config"
```

Beklenenler:

```text
stonith-enabled: true
frs-sds-group:
  zfs-pool
  vip
```

## Storage Kontrolu

iSCSI session:

```bash
ansible -i inventory/lab.ini ha_cluster -m command -a "iscsiadm -m session"
```

Disk by-id:

```bash
ansible -i inventory/lab.ini ha_cluster -m shell -a "ls -l /dev/disk/by-id/ | grep -Ei 'TrueNAS|iSCSI|scsi-STrueNAS'"
```

Aktif pool hangi node'da:

```bash
ansible -i inventory/lab.ini ha_cluster -m command -a "zpool list -H -o name"
```

Pool status:

```bash
ansible -i inventory/lab.ini ha_cluster -m command -a "zpool status frs_pool"
```

Pasif node'da `zpool status frs_pool` hata donebilir; normaldir. Pool sadece aktif node'da import edilmelidir.

Import edilebilir pool kontrolu:

```bash
ansible -i inventory/lab.ini ha_cluster -m command -a "zpool import"
```

## VIP Kontrolu

VIP hangi node'da:

```bash
ansible -i inventory/lab.ini ha_cluster -m command -a "ip -4 addr show dev ens34"
```

Sadece aktif resource node'unda `192.168.88.120` gorulmelidir.

Client erisimi:

```bash
ping 192.168.88.120
```

## Kontrollu Failover Testi

Mevcut resource yerlesimi:

```bash
ansible -i inventory/lab.ini sds-frs-n01 -m command -a "pcs status --full"
```

Aktif node `sds-frs-n01` ise standby:

```bash
ansible -i inventory/lab.ini sds-frs-n01 -m command -a "pcs node standby sds-frs-n01"
```

Resource group gecisini izle:

```bash
ansible -i inventory/lab.ini sds-frs-n01 -m command -a "pcs status --full"
```

Beklenen:

```text
frs-sds-group Started sds-frs-n02
zfs-pool Started sds-frs-n02
vip Started sds-frs-n02
```

ZFS ve VIP dogrulama:

```bash
ansible -i inventory/lab.ini ha_cluster -m command -a "zpool list -H -o name"
ansible -i inventory/lab.ini ha_cluster -m command -a "ip -4 addr show dev ens34"
```

Aktif node'u geri al:

```bash
ansible -i inventory/lab.ini sds-frs-n01 -m command -a "pcs node unstandby sds-frs-n01"
```

Not: Resource stickiness nedeniyle group otomatik geri donmeyebilir. Geri tasimak istenirse:

```bash
ansible -i inventory/lab.ini sds-frs-n01 -m command -a "pcs resource move frs-sds-group sds-frs-n01"
ansible -i inventory/lab.ini sds-frs-n01 -m command -a "pcs resource clear frs-sds-group"
```

Son kontrol:

```bash
ansible-playbook -i inventory/lab.ini playbooks/90_validation.yml
```

## Kontrollu Failback Testi

Group su anda `sds-frs-n02` uzerindeyse ve `sds-frs-n01`'e almak istiyorsak:

```bash
ansible -i inventory/lab.ini sds-frs-n01 -m command -a "pcs resource move frs-sds-group sds-frs-n01"
ansible -i inventory/lab.ini sds-frs-n01 -m command -a "pcs status --full"
ansible -i inventory/lab.ini sds-frs-n01 -m command -a "pcs resource clear frs-sds-group"
```

`pcs resource clear` unutulmamali; aksi halde gecici location constraint kalabilir.

## Fencing Testi

Dikkat: Fencing testi VM reset/power action uretir. Sadece bakimli lab ortaminda yapilmalidir.

Fence resource status:

```bash
ansible -i inventory/lab.ini sds-frs-n01 -m command -a "pcs stonith status"
```

VMware REST agent metadata:

```bash
ansible -i inventory/lab.ini sds-frs-n01 -m command -a "pcs stonith describe fence_vmware_rest"
```

Manuel fence testi yapmadan once resource group'u ve storage durumunu kontrol et:

```bash
ansible-playbook -i inventory/lab.ini playbooks/90_validation.yml
```

Gercek fence aksiyonu icin ornek:

```bash
ansible -i inventory/lab.ini sds-frs-n01 -m command -a "stonith_admin --reboot sds-frs-n02"
```

Bu komut hedef VM'i reboot eder. Test sonrasi:

```bash
ansible -i inventory/lab.ini sds-frs-n01 -m command -a "pcs status --full"
ansible-playbook -i inventory/lab.ini playbooks/90_validation.yml
```

## Troubleshooting

Cluster hatalari:

```bash
ansible -i inventory/lab.ini sds-frs-n01 -m command -a "pcs status --full"
ansible -i inventory/lab.ini sds-frs-n01 -m command -a "pcs resource failcount show"
ansible -i inventory/lab.ini sds-frs-n01 -m command -a "pcs resource cleanup frs-sds-group"
```

Pacemaker log:

```bash
ansible -i inventory/lab.ini ha_cluster -m command -a "journalctl -u pacemaker -n 100 --no-pager"
```

Corosync log:

```bash
ansible -i inventory/lab.ini ha_cluster -m command -a "journalctl -u corosync -n 100 --no-pager"
```

OCF agent debug:

```bash
ansible -i inventory/lab.ini ha_cluster -m command -a "ls -l /usr/lib/ocf/resource.d/kdx/zpool-ha"
ansible -i inventory/lab.ini sds-frs-n01 -m command -a "pcs resource describe ocf:kdx:zpool-ha"
```

iSCSI discovery tekrar:

```bash
ansible -i inventory/lab.ini ha_cluster -m command -a "iscsiadm -m discovery -t sendtargets -p 172.22.2.33"
```

iSCSI logout/login dikkatli kullanilmalidir. Resource group aktifken logout yapilmaz.

ZFS import hatasi:

```bash
ansible -i inventory/lab.ini ha_cluster -m command -a "zpool import"
ansible -i inventory/lab.ini ha_cluster -m command -a "zpool status"
```

Force import kullanilmaz:

```text
zpool import -f kullanma
zpool import -F kullanma
```

## Backup Komutlari

Cluster config backup:

```bash
ansible -i inventory/lab.ini sds-frs-n01 -m shell -a "pcs config > /root/frs-cluster-working.conf"
ansible -i inventory/lab.ini sds-frs-n01 -m shell -a "crm_mon -1Arf > /root/frs-cluster-status.txt"
```

OCF agent backup:

```bash
ansible -i inventory/lab.ini ha_cluster -m shell -a "cp /usr/lib/ocf/resource.d/kdx/zpool-ha /root/zpool-ha-working.backup"
```
