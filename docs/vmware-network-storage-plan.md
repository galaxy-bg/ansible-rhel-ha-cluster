# VMware, Network ve Storage Hazirlik Plani

Bu dokuman ilk faz 3 network tasarimina gore VM, vCenter ve TrueNAS hazirliklarini listeler.

## VM NIC Plani

Her RHEL 8.10 node icin:

| NIC | Network | Ornek Interface | Kullanim |
| --- | --- | --- | --- |
| vNIC1 | Public/Mgmt | `ens192` | SSH, Ansible, VIP, client, vCenter API |
| vNIC2 | Corosync | `ens224` | Pacemaker/Corosync heartbeat |
| vNIC3 | iSCSI | `ens256` | TrueNAS iSCSI LUN |

## vSphere Ayarlari

- VM isimleri inventory ile ayni tutulur: `frs-sds-n1`, `frs-sds-n2`.
- Mumkunse iki VM farkli ESXi hostlarda calistirilir.
- DRS varsa anti-affinity rule eklenir.
- VMware Tools kurulu ve calisir durumda olmalidir.
- Fencing servis hesabi sadece ilgili VM'leri reset/power/status yapacak yetkide olmalidir.

## Fencing Servis Hesabi

Ornek:

```text
svc_pacemaker_fence@vsphere.local
```

Gerekli operasyonlar:

- VM power status okuma
- VM reset/reboot
- VM power off
- VM power on

Sifre `group_vars/vault.yml` icinde saklanir.

## TrueNAS Ayarlari

- iSCSI portal Storage network IP'sinden yayinlanir.
- Iki node'un initiator IQN bilgisi TrueNAS tarafinda izinli olmalidir.
- Tek shared LUN iki node'a da sunulur.
- LUN uzerindeki ZFS pool sadece Pacemaker tarafindan aktif node'da import edilir.

## Sonraki Faz

Multipath icin ikinci storage network eklenir:

| Network | Kullanim |
| --- | --- |
| iSCSI-A | Path 1 |
| iSCSI-B | Path 2 |

Bu fazda `multipath.conf`, iki portal discovery ve path failover testleri eklenecek.

