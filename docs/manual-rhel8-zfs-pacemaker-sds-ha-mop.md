# MOP - RHEL 8.10 Üzerinde ZFS + Pacemaker Active/Passive SDS HA

## 1. Amaç

Bu doküman TrueNAS üzerinden sunulan paylaşımlı iSCSI LUN kullanılarak, RHEL 8.10 üzerinde çalışan iki SDS node arasında ZFS Pool failover sağlayan Pacemaker/Corosync tabanlı Active/Passive HA yapısının kurulumunu anlatır.

---

# 2. Mimari

## Node Bilgileri

| Host | IP |
|--------|--------|
| frs-sds-n1 | 172.12.2.77 |
| frs-sds-n2 | 172.12.2.78 |
| VIP | 172.12.2.100 |
| TrueNAS | 172.12.2.75 |

## iSCSI Target

```text
iqn.2005-10.org.freenas.ctl:frs-share
```

---

# 3. Ön Gereksinimler

- RHEL 8.10
- Secure Boot Disabled
- TrueNAS iSCSI Target
- DNS veya hosts çözümlemesi
- İki node arasında düşük gecikmeli ağ bağlantısı

---

# 4. Hostname ve Hosts

## Node1

```bash
hostnamectl set-hostname frs-sds-n1.kdx.local
```

## Node2

```bash
hostnamectl set-hostname frs-sds-n2.kdx.local
```

## /etc/hosts

```text
172.12.2.77 frs-sds-n1.kdx.local frs-sds-n1
172.12.2.78 frs-sds-n2.kdx.local frs-sds-n2
```

Kontrol:

```bash
hostname -f
ping frs-sds-n1
ping frs-sds-n2
```

---

# 5. Repo ve Paketler

```bash
subscription-manager repos --enable=rhel-8-for-x86_64-baseos-rpms
subscription-manager repos --enable=rhel-8-for_x86_64-appstream-rpms
subscription-manager repos --enable=codeready-builder-for-rhel-8-x86_64-rpms
subscription-manager repos --enable=rhel-8-for-x86_64-highavailability-rpms
```

EPEL:

```bash
dnf install -y epel-release
```

ZFS Repo:

```bash
dnf install -y https://zfsonlinux.org/epel/zfs-release-2-3.el8.noarch.rpm
```

Kurulum:

```bash
dnf install -y \
kernel-devel \
kernel-headers \
gcc \
make \
dkms \
zfs \
iscsi-initiator-utils \
device-mapper-multipath \
pcs \
pacemaker \
corosync \
resource-agents \
fence-agents-all
```

---

# 6. Secure Boot

Kontrol:

```bash
mokutil --sb-state
```

Beklenen:

```text
SecureBoot disabled
```

---

# 7. SSH Gecikme Problemi

İki node üzerinde:

```bash
echo "UseDNS no" >> /etc/ssh/sshd_config
systemctl restart sshd
```

---

# 8. iSCSI Discovery

```bash
systemctl enable --now iscsid
```

Discovery:

```bash
iscsiadm -m discovery -t sendtargets -p 172.12.2.75
```

Login:

```bash
iscsiadm -m node \
-T iqn.2005-10.org.freenas.ctl:frs-share \
-p 172.12.2.75:3260 \
--login
```

Automatic Startup:

```bash
iscsiadm -m node \
-T iqn.2005-10.org.freenas.ctl:frs-share \
-p 172.12.2.75:3260 \
--op update \
-n node.startup \
-v automatic
```

Kontrol:

```bash
iscsiadm -m session
lsblk
```

---

# 9. Disk Kontrolü

Disk by-id ile kullanılmalıdır.

```bash
ls -l /dev/disk/by-id/ | grep TrueNAS
```

Örnek:

```text
scsi-STrueNAS_iSCSI_Disk_64118f6612f5dc5
```

NOT:

Node1'de sdb, Node2'de sdc olabilir.
Kesinlikle by-id kullanılmalıdır.

---

# 10. Disk Temizleme

Node1 üzerinde:

```bash
wipefs -a /dev/disk/by-id/scsi-STrueNAS_iSCSI_Disk_64118f6612f5dc5
sgdisk --zap-all /dev/disk/by-id/scsi-STrueNAS_iSCSI_Disk_64118f6612f5dc5
partprobe
```

---

# 11. ZFS Pool Oluşturma

```bash
zpool create \
-f \
-o ashift=14 \
frs_pool \
/dev/disk/by-id/scsi-STrueNAS_iSCSI_Disk_64118f6612f5dc5
```

Kontrol:

```bash
zpool status
zpool get ashift frs_pool
```

Beklenen:

```text
ashift = 14
```

---

# 12. ZVOL Oluşturma

```bash
zfs create -V 50G frs_pool/testlun
```

Kontrol:

```bash
zfs list
zfs list -t volume
```

---

# 13. Manuel Failover Testleri

Node1:

```bash
zpool export frs_pool
```

Node2:

```bash
zpool import frs_pool
```

Kontrol:

```bash
zpool status
zfs list
```

Sonra ters yönde tekrarla.

NOT:

```text
zpool import -f kullanılmaz
zpool import -F kullanılmaz
```

---

# 14. Pacemaker Kurulumu

İki node:

```bash
systemctl enable --now pcsd
passwd hacluster
```

Auth:

```bash
pcs host auth frs-sds-n1 frs-sds-n2 -u hacluster
```

Cluster:

```bash
pcs cluster setup frs-cluster frs-sds-n1 frs-sds-n2 --force
```

Başlat:

```bash
pcs cluster start --all
pcs cluster enable --all
```

---

# 15. Cluster Properties

Lab ortamı:

```bash
pcs property set stonith-enabled=false
pcs property set no-quorum-policy=ignore
```

---

# 16. VIP Resource

```bash
pcs resource create vip \
ocf:heartbeat:IPaddr2 \
ip=172.12.2.100 \
cidr_netmask=24 \
op monitor interval=30s
```

---

# 17. OCF Agent

Dosya:

```text
/usr/lib/ocf/resource.d/kdx/zpool-ha
```

Amaç:

- Güvenli import
- Güvenli export
- Force import yapmamak
- Veri bütünlüğünü korumak

Metadata kontrol:

```bash
pcs resource describe ocf:kdx:zpool-ha
```

---

# 18. ZFS Resource

```bash
pcs resource create zfs-pool \
ocf:kdx:zpool-ha \
pool=frs_pool \
op start timeout=60s \
op stop timeout=120s \
op monitor interval=30s timeout=20s
```

---

# 19. Constraintler

Colocation:

```bash
pcs constraint colocation add vip with zfs-pool INFINITY
```

Order:

```bash
pcs constraint order start zfs-pool then vip
```

---

# 20. Failover Testi

Node1 standby:

```bash
pcs node standby frs-sds-n1
```

Beklenen:

```text
zfs-pool -> Node2
vip -> Node2
```

Kontrol:

```bash
pcs status --full
```

Node2:

```bash
zpool status
zfs list
```

Node1:

```bash
zpool status
```

Beklenen:

```text
no pools available
```

---

# 21. Failback

```bash
pcs node unstandby frs-sds-n1
```

Taşı:

```bash
pcs resource move zfs-pool frs-sds-n1
```

Temizle:

```bash
pcs resource clear zfs-pool
```

Kontrol:

```bash
pcs status
```

---

# 22. Backup

```bash
pcs config > /root/frs-cluster-working.conf
crm_mon -1Arf > /root/frs-cluster-status.txt
```

Agent Backup:

```bash
cp /usr/lib/ocf/resource.d/kdx/zpool-ha \
/root/zpool-ha-working.backup
```

---

# 23. Troubleshooting

## SSH Donması

Çözüm:

```bash
UseDNS no
```

## Pool Import Hatası

Kontrol:

```bash
zpool import
```

## Cluster

```bash
pcs status --full
```

## Corosync

```bash
corosync-cfgtool -s
```

## Pacemaker

```bash
journalctl -u pacemaker -n 100
```

---

# 24. Production Notları

Mutlaka:

```text
STONITH aktif edilmeli
```

Öneriler:

- HPE iLO
- IPMI
- Redfish
- VMware Fencing

---

# 25. Sonraki Faz

Eklenecek:

- targetcli
- LIO
- iSCSI Export
- Client Testleri

Kaynak sırası:

1. zfs-pool
2. iscsi-target
3. vip

Stop sırası:

1. vip
2. iscsi-target
3. zfs-pool
