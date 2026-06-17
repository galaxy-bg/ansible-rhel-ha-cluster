# Current Lab Implementation Notes

Bu dosya 2026-06-15 tarihinde hazirlanan mevcut lab degerlerini ve ilk implementation adimlarini takip etmek icindir.

## Node Bilgileri

| Host | Public/Mgmt | Corosync Private | iSCSI |
| --- | --- | --- | --- |
| sds-frs-n01 | 192.168.88.118/24 | 172.12.2.84/16 | 172.22.2.35/16 |
| sds-frs-n02 | 192.168.88.119/24 | 172.12.2.85/16 | 172.22.2.36/16 |

## Interface Mapping

| Interface | Network |
| --- | --- |
| ens34 | Public/Mgmt |
| ens35 | Corosync Private |
| ens36 | iSCSI |

Pacemaker node names public/mgmt network uzerinden cozulur; Corosync heartbeat icin `pcs cluster setup` komutunda node'lara ayrica `addr=<corosync_ip>` verilir. Bu sayede ring0 trafiği `172.12.2.x` private VLAN uzerinden akar.

## TrueNAS

| Alan | Deger |
| --- | --- |
| iSCSI VLAN/network | 172.22.0.0/16 |
| TrueNAS iSCSI IP | 172.22.2.33 |
| LUN size | 250GB |

## Eksik Degerler

Implementation oncesi netlesecek degerler:

```text
iscsi_target_iqn
zfs_disk_by_id
vCenter address
VMware fencing username
VMware fencing password, vault icinde
hacluster password, vault icinde
```

Secret degerler su path altinda tutulur:

```bash
inventory/group_vars/ha_cluster/vault.yml
```

Bu dosya `.gitignore` kapsamindadir. Ornek dosya:

```bash
inventory/group_vars/ha_cluster/vault.example.yml
```

`zfs_disk_by_id`, iSCSI login sonrasi su komutla dogrulanir:

```bash
ls -l /dev/disk/by-id/ | grep -Ei 'TrueNAS|iSCSI|scsi'
```

## Ansible Control Node

Ilk fazda Ansible laptop uzerinden calistirilacak. Bunun avantaji cluster node'larinin sadece target kalmasi ve tekrar kurulum/test akisinin daha temiz olmasidir.

## SSH Hazirligi

Ansible ping icin iki node'a SSH erisimi gerekir. Onerilen yontem SSH key:

```bash
ssh-copy-id root@192.168.88.118
ssh-copy-id root@192.168.88.119
ansible -i inventory/lab.ini ha_cluster -m ping
```

Alternatif olarak root parola ile calisilacaksa `--ask-pass` kullanilabilir veya parola Ansible Vault icinde tutulabilir.

## Ilk Kosum Sirasi

```bash
ansible -i inventory/lab.ini ha_cluster -m ping
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

## Pacemaker Resource Model

Ilk fazda ZFS pool ve VIP ayni Pacemaker resource group icinde yonetilir:

```text
frs-sds-group:
  zfs-pool
  vip
```

Group, kaynaklari ayni node'da tutar ve sirali baslatir. Start sirasi `zfs-pool -> vip`, stop sirasi `vip -> zfs-pool` olur.
