# ansible-rhel-ha-cluster

RHEL 8.10 uzerinde iki node'lu Pacemaker/Corosync HA cluster kurulumu icin Ansible projesi.

Ilk faz kapsam:

- 2 node active/passive HA cluster
- 3 network tasarimi
- TrueNAS iSCSI shared LUN
- ZFS pool failover
- VMware ESXi/vCenter fencing
- VIP + ZFS resource order/colocation

## Mimari

| Bilesen | Deger |
| --- | --- |
| OS | Red Hat Enterprise Linux 8.10 |
| Cluster | Pacemaker + Corosync + pcs |
| Storage | TrueNAS iSCSI shared LUN |
| Filesystem | OpenZFS |
| HA modeli | Active/Passive |
| Fencing | VMware REST API, `fence_vmware_rest` |

## 3 Network Tasarimi

Ilk fazda her VM icin 3 NIC kullanilir.

| Network | Ornek Interface | Kullanim |
| --- | --- | --- |
| Public / Management | `ens192` | SSH, Ansible, pcs, VIP, client erisimi, vCenter erisimi |
| Cluster / Corosync | `ens224` | Node-to-node heartbeat |
| Storage / iSCSI | `ens256` | TrueNAS iSCSI portal erisimi |

Ornek IP plani:

| Host | Public/Mgmt | Corosync | iSCSI |
| --- | --- | --- | --- |
| frs-sds-n1 | 172.12.2.77 | 10.10.10.77 | 10.20.20.77 |
| frs-sds-n2 | 172.12.2.78 | 10.10.10.78 | 10.20.20.78 |
| VIP | 172.12.2.100 | - | - |
| TrueNAS | - | - | 10.20.20.75 |

Sonraki fazda iSCSI icin ikinci path eklenip multipath tasarimi genisletilebilir.

## VMware Hazirliklari

Best practice'e yakin lab icin:

- `frs-sds-n1` ve `frs-sds-n2` VM'leri ayni ESXi host uzerinde tutulmamali.
- vSphere DRS varsa anti-affinity rule eklenmeli.
- Pacemaker fencing icin ayri servis hesabi acilmali.
- Servis hesabina sadece ilgili iki VM icin power/status/reset yetkileri verilmeli.
- Node'lardan vCenter/ESXi API adresine Public/Mgmt network uzerinden erisim olmali.
- VM adlari sabit tutulmali veya `plug` alaninda VM UUID kullanilmali.

Detayli hazirlik listesi: [docs/vmware-network-storage-plan.md](docs/vmware-network-storage-plan.md)

## TrueNAS Hazirliklari

- iSCSI portal Storage network uzerinde olmalidir.
- Iki RHEL node'un initiator IQN degerleri TrueNAS tarafinda allow list'e eklenmelidir.
- Shared LUN iki node'a da gorunmelidir.
- ZFS pool ayni anda yalnizca aktif node uzerinde import edilmelidir.

## Kullanim

Degiskenleri once duzenleyin:

```bash
vim inventory/lab.ini
vim inventory/group_vars/ha_cluster.yml
```

Vault dosyasi olusturun:

```bash
ansible-vault create inventory/group_vars/vault.yml
```

Ornek icerik:

```yaml
vault_hacluster_password: "ChangeMe"
vault_vmware_fencing_password: "ChangeMe"
```

Playbook sirasi:

```bash
ansible-playbook -i inventory/lab.ini playbooks/00_common.yml --ask-vault-pass
ansible-playbook -i inventory/lab.ini playbooks/10_cluster_install.yml --ask-vault-pass
ansible-playbook -i inventory/lab.ini playbooks/20_cluster_setup.yml --ask-vault-pass
ansible-playbook -i inventory/lab.ini playbooks/30_cluster_properties.yml --ask-vault-pass
ansible-playbook -i inventory/lab.ini playbooks/40_fencing.yml --ask-vault-pass
ansible-playbook -i inventory/lab.ini playbooks/50_storage.yml --ask-vault-pass
ansible-playbook -i inventory/lab.ini playbooks/60_resources.yml --ask-vault-pass
ansible-playbook -i inventory/lab.ini playbooks/70_constraints.yml --ask-vault-pass
ansible-playbook -i inventory/lab.ini playbooks/90_validation.yml --ask-vault-pass
```

Tum akisi tek komutla calistirmak icin:

```bash
ansible-playbook -i inventory/lab.ini site.yml --ask-vault-pass
```

## Kritik Notlar

- Shared ZFS pool icin disk yolu mutlaka `/dev/disk/by-id/...` olmalidir.
- `zpool import -f` ve `zpool import -F` kullanilmaz.
- Production benzeri senaryoda `stonith-enabled=true` olmalidir.
- Lab kolayligi icin STONITH kapatilabilir, fakat shared storage varken onerilmez.
- Cluster resource baslama sirasi: `zfs-pool -> vip`
- Cluster resource durma sirasi: `vip -> zfs-pool`

## Failover Testleri

```bash
pcs status --full
pcs node standby frs-sds-n1
pcs status --full
zpool status
pcs node unstandby frs-sds-n1
pcs resource move zfs-pool frs-sds-n1
pcs resource clear zfs-pool
```

Fencing testi once kontrollu ve bakimli lab ortaminda yapilmalidir.
