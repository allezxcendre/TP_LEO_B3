---
title: TP2_LEO_stockage

---

# II. SAN network



🌞 **Configurer des RAID**
```
[alexandre@Sto1 ~]$ sudo mdadm --create /dev/md0 --level=5 --raid-devices=3 /dev/sdb /dev/s
dic/dev/sdd

[alexandre@Sto1 ~]$ sudo mdadm --create /dev/md1 --level=5 --raid-devices=3 /dev/sde /dev/s
df /dev/sdg

[alexandre@Sto1 ~]$ sudo mdadm --create /dev/md2 --level=5 --raid-devices=3 /dev/sdh /dev/s
di /dev/sdj
```
🌞 **Prouvez que vous avez 3 volumes RAID prêts à l'emploi**

```
[alexandre@Sto1 ~]$ lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda           8:0    0   80G  0 disk
├─sda1        8:1    0    1G  0 part  /boot
└─sda2        8:2    0   79G  0 part
  ├─rl-root 253:0    0 51.7G  0 lvm   /
  ├─rl-swap 253:1    0    2G  0 lvm   [SWAP]
  └─rl-home 253:2    0 25.2G  0 lvm   /home
sdb           8:16   0   20G  0 disk
└─md0         9:0    0   40G  0 raid5
sdc           8:32   0   20G  0 disk
└─md0         9:0    0   40G  0 raid5
sdd           8:48   0   20G  0 disk
└─md0         9:0    0   40G  0 raid5
sde           8:64   0   20G  0 disk
└─md1         9:1    0   40G  0 raid5
sdf           8:80   0   20G  0 disk
└─md1         9:1    0   40G  0 raid5
sdg           8:96   0   20G  0 disk
└─md1         9:1    0   40G  0 raid5
sdh           8:112  0   20G  0 disk
└─md2         9:2    0   40G  0 raid5
sdi           8:128  0   20G  0 disk
└─md2         9:2    0   40G  0 raid5
sdj           8:144  0   20G  0 disk
└─md2         9:2    0   40G  0 raid5
sr0          11:0    1 1024M  0 rom
```

### B. iSCSI target



🌞 **Installer `target`**
````
[alexandre@Sto2 ~]$ sudo dnf install targetcli -y
````
🌞 **Démarrer le service `target`**
````
[alexandre@Sto2 ~]$ sudo systemctl start target
[alexandre@Sto2 ~]$ sudo systemctl enable target
Created symlink /etc/systemd/system/multi-user.target.wants/target.service → /usr/lib/systemd/system/target.service.
````

🌞 **Configurer les *targets iSCSI***

```bash
/backstores/fileio create name=data-chunk1 file_or_dev=/dev/md0
/backstores/fileio create name=data-chunk2 file_or_dev=/dev/md1
/backstores/fileio create name=data-chunk3 file_or_dev=/dev/md2

/iscsi create iqn.2024-12.tp2.b3:data-chunk1
/iscsi create iqn.2024-12.tp2.b3:data-chunk2
/iscsi create iqn.2024-12.tp2.b3:data-chunk3

/iscsi/iqn.2024-12.tp2.b3:data-chunk1/tpg1/acls create iqn.2024-12.tp2.b3:data-chunk1:chunk1-initiator
/iscsi/iqn.2024-12.tp2.b3:data-chunk2/tpg1/acls create iqn.2024-12.tp2.b3:data-chunk2:chunk2-initiator
/iscsi/iqn.2024-12.tp2.b3:data-chunk3/tpg1/acls create iqn.2024-12.tp2.b3:data-chunk3:chunk3-initiator

/iscsi/iqn.2024-12.tp2.b3:data-chunk1/tpg1/luns/ create /backstores/fileio/data-chunk1
/iscsi/iqn.2024-12.tp2.b3:data-chunk2/tpg1/luns/ create /backstores/fileio/data-chunk2
/iscsi/iqn.2024-12.tp2.b3:data-chunk3/tpg1/luns/ create /backstores/fileio/data-chunk3

saveconfig
```

## 2. Chunks machine


### A. Simple iSCSI

🌞 **Installer les tools iSCSI sur `chunk1.tp2.b3`**
````
[alexandre@Chunk1 ~]$ sudo dnf install iscsi-initiator-utils -y
````
🌞 **Configurer un iSCSI initiator**

```
sudo iscsiadm -m discoverydb -t st -p 10.3.1.1:3260 --discover
sudo iscsiadm -m discoverydb -t st -p 10.3.1.2:3260 --discover
sudo iscsiadm -m discoverydb -t st -p 10.3.2.1:3260 --discover
sudo iscsiadm -m discoverydb -t st -p 10.3.2.2:3260 --discover

[alexandre@Chunk1 ~]$ sudo iscsiadm -m node
10.3.1.1:3260,1 iqn.2024-12.tp2.b3:data-chunk1
10.3.1.2:3260,1 iqn.2024-12.tp2.b3:data-chunk1
10.3.2.1:3260,1 iqn.2024-12.tp2.b3:data-chunk1
10.3.2.2:3260,1 iqn.2024-12.tp2.b3:data-chunk1
10.3.1.1:3260,1 iqn.2024-12.tp2.b3:data-chunk2
10.3.1.2:3260,1 iqn.2024-12.tp2.b3:data-chunk2
10.3.2.1:3260,1 iqn.2024-12.tp2.b3:data-chunk2
10.3.2.2:3260,1 iqn.2024-12.tp2.b3:data-chunk2
10.3.1.1:3260,1 iqn.2024-12.tp2.b3:data-chunk3
10.3.1.2:3260,1 iqn.2024-12.tp2.b3:data-chunk3
10.3.2.1:3260,1 iqn.2024-12.tp2.b3:data-chunk3
10.3.2.2:3260,1 iqn.2024-12.tp2.b3:data-chunk3

[alexandre@Chunk1 ~]$ sudo iscsiadm -m node --targetname iqn.2024-12.tp2.b3:data-chunk1 --portal 10.3.1.1:3260 --login

[alexandre@Chunk1 ~]$ sudo iscsiadm -m session
tcp: [10] 10.3.1.2:3260,1 iqn.2024-12.tp2.b3:data-chunk1 (non-flash)
tcp: [11] 10.3.2.2:3260,1 iqn.2024-12.tp2.b3:data-chunk1 (non-flash)
tcp: [16] 10.3.2.1:3260,1 iqn.2024-12.tp2.b3:data-chunk1 (non-flash)
tcp: [2] 10.3.1.1:3260,1 iqn.2024-12.tp2.b3:data-chunk1 (non-flash)
```

🌞 **Modifier la configuration du démon iSCSI**
```
[alexandre@Chunk1 ~]$ sudo nano /etc/iscsi/iscsid.conf
[alexandre@Chunk1 ~]$ sudo systemctl restart iscsi
[alexandre@Chunk1 ~]$ sudo systemctl restart iscsid
```

🌞 **Prouvez que la configuration est prête**
```
[alexandre@Chunk1 ~]$ lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda           8:0    0   80G  0 disk
├─sda1        8:1    0    1G  0 part /boot
└─sda2        8:2    0   79G  0 part
  ├─rl-root 253:0    0 51.7G  0 lvm  /
  ├─rl-swap 253:1    0    2G  0 lvm  [SWAP]
  └─rl-home 253:2    0 25.2G  0 lvm  /home
sdb           8:16   0   40G  0 disk
sdc           8:32   0   40G  0 disk
sdd           8:48   0   40G  0 disk
sde           8:64   0   40G  0 disk
sr0          11:0    1 1024M  0 rom

[alexandre@Chunk1 ~]$ sudo iscsiadm -m session -P 3
iSCSI Transport Class version 2.0-870
version 6.2.1.9
Target: iqn.2024-12.tp2.b3:data-chunk1 (non-flash)
        Current Portal: 10.3.1.2:3260,1
        Persistent Portal: 10.3.1.2:3260,1
                **********
                Interface:
                **********
                Iface Name: default
                Iface Transport: tcp
                Iface Initiatorname: iqn.2024-12.tp2.b3:data-chunk1:chunk1-initiator
                Iface IPaddress: 10.3.1.101
                Iface HWaddress: default
                Iface Netdev: default
                SID: 10
                iSCSI Connection State: LOGGED IN
                iSCSI Session State: LOGGED_IN
                Internal iscsid Session State: NO CHANGE
                *********
                Timeouts:
                *********
                Recovery Timeout: 120
                Target Reset Timeout: 30
                LUN Reset Timeout: 30
                Abort Timeout: 15
                *****
                CHAP:
                *****
                username: <empty>
                password: ********
                username_in: <empty>
                password_in: ********
                ************************
                Negotiated iSCSI params:
                ************************
                HeaderDigest: None
                DataDigest: None
                MaxRecvDataSegmentLength: 262144
                MaxXmitDataSegmentLength: 262144
                FirstBurstLength: 65536
                MaxBurstLength: 262144
                ImmediateData: Yes
                InitialR2T: Yes
                MaxOutstandingR2T: 1
                ************************
                Attached SCSI devices:
                ************************
                Host Number: 3  State: running
                scsi3 Channel 00 Id 0 Lun: 0
                        Attached scsi disk sdc          State: running
        Current Portal: 10.3.2.2:3260,1
        Persistent Portal: 10.3.2.2:3260,1
                **********
                Interface:
                **********
                Iface Name: default
                Iface Transport: tcp
                Iface Initiatorname: iqn.2024-12.tp2.b3:data-chunk1:chunk1-initiator
                Iface IPaddress: 10.3.2.101
                Iface HWaddress: default
                Iface Netdev: default
                SID: 11
                iSCSI Connection State: LOGGED IN
                iSCSI Session State: LOGGED_IN
                Internal iscsid Session State: NO CHANGE
                *********
                Timeouts:
                *********
                Recovery Timeout: 120
                Target Reset Timeout: 30
                LUN Reset Timeout: 30
                Abort Timeout: 15
                *****
                CHAP:
                *****
                username: <empty>
                password: ********
                username_in: <empty>
                password_in: ********
                ************************
                Negotiated iSCSI params:
                ************************
                HeaderDigest: None
                DataDigest: None
                MaxRecvDataSegmentLength: 262144
                MaxXmitDataSegmentLength: 262144
                FirstBurstLength: 65536
                MaxBurstLength: 262144
                ImmediateData: Yes
                InitialR2T: Yes
                MaxOutstandingR2T: 1
                ************************
                Attached SCSI devices:
                ************************
                Host Number: 5  State: running
                scsi5 Channel 00 Id 0 Lun: 0
                        Attached scsi disk sdd          State: running
        Current Portal: 10.3.2.1:3260,1
        Persistent Portal: 10.3.2.1:3260,1
                **********
                Interface:
                **********
                Iface Name: default
                Iface Transport: tcp
                Iface Initiatorname: iqn.2024-12.tp2.b3:data-chunk1:chunk1-initiator
                Iface IPaddress: 10.3.2.101
                Iface HWaddress: default
                Iface Netdev: default
                SID: 16
                iSCSI Connection State: LOGGED IN
                iSCSI Session State: LOGGED_IN
                Internal iscsid Session State: NO CHANGE
                *********
                Timeouts:
                *********
                Recovery Timeout: 120
                Target Reset Timeout: 30
                LUN Reset Timeout: 30
                Abort Timeout: 15
                *****
                CHAP:
                *****
                username: <empty>
                password: ********
                username_in: <empty>
                password_in: ********
                ************************
                Negotiated iSCSI params:
                ************************
                HeaderDigest: None
                DataDigest: None
                MaxRecvDataSegmentLength: 262144
                MaxXmitDataSegmentLength: 262144
                FirstBurstLength: 65536
                MaxBurstLength: 262144
                ImmediateData: Yes
                InitialR2T: Yes
                MaxOutstandingR2T: 1
                ************************
                Attached SCSI devices:
                ************************
                Host Number: 9  State: running
                scsi9 Channel 00 Id 0 Lun: 0
                        Attached scsi disk sde          State: running
        Current Portal: 10.3.1.1:3260,1
        Persistent Portal: 10.3.1.1:3260,1
                **********
                Interface:
                **********
                Iface Name: default
                Iface Transport: tcp
                Iface Initiatorname: iqn.2024-12.tp2.b3:data-chunk1:chunk1-initiator
                Iface IPaddress: 10.3.1.101
                Iface HWaddress: default
                Iface Netdev: default
                SID: 2
                iSCSI Connection State: LOGGED IN
                iSCSI Session State: LOGGED_IN
                Internal iscsid Session State: NO CHANGE
                *********
                Timeouts:
                *********
                Recovery Timeout: 120
                Target Reset Timeout: 30
                LUN Reset Timeout: 30
                Abort Timeout: 15
                *****
                CHAP:
                *****
                username: <empty>
                password: ********
                username_in: <empty>
                password_in: ********
                ************************
                Negotiated iSCSI params:
                ************************
                HeaderDigest: None
                DataDigest: None
                MaxRecvDataSegmentLength: 262144
                MaxXmitDataSegmentLength: 262144
                FirstBurstLength: 65536
                MaxBurstLength: 262144
                ImmediateData: Yes
                InitialR2T: Yes
                MaxOutstandingR2T: 1
                ************************
                Attached SCSI devices:
                ************************
                Host Number: 4  State: running
                scsi4 Channel 00 Id 0 Lun: 0
                        Attached scsi disk sdb          State: running
```
### B. Multipathing

> Toujouuurs sur `chunk1.tp2.b3`.

Le *multipathing* va terminer le setup iSCSI. Actuellement, notre machine `chunk1` voit 4 volumes, alors qu'il en existe que 2 en réalité.

On va configurer le *multipathing* pour que la machine `chunk1` ne voit que deux volumes, chacun ayant deux chemins d'accès possibles : **on ajoute un niveau de redondance pour tolérer les pannes réseau.**

🌞 **Installer les outils multipath sur `chunk1.tp2.b3`**
```
[alexandre@Chunk1 ~]$ sudo dnf install device-mapper-multipath
```
🌞 **Configurer le fichier `/etc/multipath.conf`**

```conf
[alexandre@Chunk1 /]$ sudo cat usr/share/doc/device-mapper-multipath/multipath.conf
[alexandre@Chunk1 /]$ sudo nano /etc/multipath.conf

defaults {
  user_friendly_names yes
  find_multipaths yes
  path_grouping_policy failover
  features "1 queue_if_no_path"
  no_path_retry 100
}


```

🌞 **Démarrer le service `multipathd`**
```
[alexandre@Chunk1 /]$ sudo systemctl restart multipathd
```
🌞 **Et euh c'est tout, il est smart enough**
```
[alexandre@Chunk1 /]$ lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda           8:0    0   80G  0 disk
├─sda1        8:1    0    1G  0 part  /boot
└─sda2        8:2    0   79G  0 part
  ├─rl-root 253:0    0 51.7G  0 lvm   /
  ├─rl-swap 253:1    0    2G  0 lvm   [SWAP]
  └─rl-home 253:2    0 25.2G  0 lvm   /home
sdb           8:16   0   40G  0 disk
└─mpathb    253:4    0   40G  0 mpath
sdc           8:32   0   40G  0 disk
└─mpatha    253:3    0   40G  0 mpath
sdd           8:48   0   40G  0 disk
└─mpatha    253:3    0   40G  0 mpath
sde           8:64   0   40G  0 disk
└─mpathb    253:4    0   40G  0 mpath
sr0          11:0    1 1024M  0 rom
[alexandre@Chunk1 /]$ multipath -ll
need to be root
[alexandre@Chunk1 /]$ sudo multipath -ll
mpatha (360014058f0d8c6e00fa4ca095ade73df) dm-3 LIO-ORG,data-chunk1
size=40G features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 3:0:0:0 sdc 8:32 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 5:0:0:0 sdd 8:48 active ready running
mpathb (36001405c428a49690dc477d8676ab6ff) dm-4 LIO-ORG,data-chunk1
size=40G features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 4:0:0:0 sdb 8:16 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 9:0:0:0 sde 8:64 active ready running
```

## 3. Formatage et montage

🌞 **Créez une partition sur les devices `mpatha` et `mpathb`**
```
[alexandre@Chunk1 /]$ sudo fdisk /dev/mapper/mpatha
[alexandre@Chunk1 /]$ sudo fdisk /dev/mapper/mpathb
```
🌞 **Formatez en `xfs` les partitions**

```
[alexandre@Chunk1 /]$ sudo mkfs.xfs /dev/mapper/mpatha1
meta-data=/dev/mapper/mpatha1    isize=512    agcount=4, agsize=2618752 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=0
data     =                       bsize=4096   blocks=10475008, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[alexandre@Chunk1 /]$ sudo mkfs.xfs /dev/mapper/mpathb1
meta-data=/dev/mapper/mpathb1    isize=512    agcount=4, agsize=2618752 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=0
data     =                       bsize=4096   blocks=10475008, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```

🌞 **Point de montage `/mnt/data_chunk1`**

```
[alexandre@Chunk1 /]$ sudo mkdir -p /mnt/data_chunk1
[alexandre@Chunk1 /]$ sudo mkdir -p /mnt/data_chunk2

sudo mount /dev/mapper/mpatha1 /mnt/data_chunk1
sudo mount /dev/mapper/mpathb1 /mnt/data_chunk2

[alexandre@Chunk1 /]$ df -h
Filesystem           Size  Used Avail Use% Mounted on
devtmpfs             4.0M     0  4.0M   0% /dev
tmpfs                888M     0  888M   0% /dev/shm
tmpfs                355M  5.1M  350M   2% /run
/dev/mapper/rl-root   52G  1.6G   51G   3% /
/dev/sda1            960M  230M  731M  24% /boot
/dev/mapper/rl-home   26G  213M   25G   1% /home
tmpfs                178M     0  178M   0% /run/user/1000
/dev/mapper/mpatha1   40G  318M   40G   1% /mnt/data_chunk1
/dev/mapper/mpathb1   40G  318M   40G   1% /mnt/data_chunk2


```

🌞 **Point de montage `/mnt/data_chunk2`**``````

- pareil, pour `mpathb`

Ca donne ça :

```bash
[it4@chunk1 ~]$ lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda           8:0    0    8G  0 disk  
├─sda1        8:1    0    1G  0 part  /boot
└─sda2        8:2    0    7G  0 part  
  ├─rl-root 253:0    0  6.2G  0 lvm   /
  └─rl-swap 253:1    0  820M  0 lvm   [SWAP]
sdb           8:16   0  511M  0 disk  
└─mpathb    253:4    0  511M  0 mpath 
  └─mpathb1 253:5    0  503M  0 part  
sdc           8:32   0  511M  0 disk  
└─mpatha    253:2    0  511M  0 mpath 
  └─mpatha1 253:3    0  503M  0 part  
sdd           8:48   0  511M  0 disk  
└─mpatha    253:2    0  511M  0 mpath 
  └─mpatha1 253:3    0  503M  0 part  
sde           8:64   0  511M  0 disk  
└─mpathb    253:4    0  511M  0 mpath 
  └─mpathb1 253:5    0  503M  0 part

[it4@chunk1 ~]$ df -h 
Filesystem           Size  Used Avail Use% Mounted on
devtmpfs             4.0M     0  4.0M   0% /dev
tmpfs                229M     0  229M   0% /dev/shm
tmpfs                 92M  2.5M   89M   3% /run
/dev/mapper/rl-root  6.2G  2.5G  3.7G  41% /
/dev/sda1            960M  280M  681M  30% /boot
tmpfs                 46M  4.0K   46M   1% /run/user/1000
/dev/mapper/mpatha1  439M  149M  291M  34% /mnt/data_chunk1
/dev/mapper/mpathb1  439M  206M  234M  47% /mnt/data_chunk2

sudo nano /etc/systemd/system/mnt-data_chunk1.mount

[Unit]
Description=Mount mpatha to /mnt/data_chunk1
After=network-online.target
Wants=network-online.target

[Mount]
What=/dev/mapper/mpatha1
Where=/mnt/data_chunk1
Type=xfs
Options=defaults,_netdev

[Install]
WantedBy=multi-user.target

[alexandre@Chunk1 /]$ sudo systemctl status mnt-data_chunk1.mount
● mnt-data_chunk1.mount - /mnt/data_chunk1
     Loaded: loaded (/proc/self/mountinfo)
     Active: active (mounted) since Fri 2024-12-06 10:28:52 CET; 5min ago
      Until: Fri 2024-12-06 10:28:52 CET; 5min ago
      Where: /mnt/data_chunk1
       What: /dev/mapper/mpatha1
```

## 4. Tests



### A. Simulation de panne

🌞 **Simuler une coupure réseau**
```
[alexandre@Chunk1 ~]$ sudo watch -n 0.5 multipath -ll
Every 0.5s: multipath -ll                                                               Chunk1: Fri Dec  6 10:49:19 2024
mpatha (360014058f0d8c6e00fa4ca095ade73df) dm-3 LIO-ORG,data-chunk1
size=40G features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 3:0:0:0 sdc 8:32 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 5:0:0:0 sdd 8:48 active ready running
mpathb (36001405c428a49690dc477d8676ab6ff) dm-4 LIO-ORG,data-chunk1
size=40G features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 4:0:0:0 sdb 8:16 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 9:0:0:0 sde 8:64 active ready running
  
[alexandre@Chunk1 ~]$ sudo journalctl -f
      Dec 06 10:49:15 Chunk1 sudo[48948]: pam_unix(sudo:session): session opened for user root(uid=0) by alexandre(uid=1000)
Dec 06 10:49:21 Chunk1 sudo[48948]: pam_unix(sudo:session): session closed for user root
Dec 06 10:49:48 Chunk1 sudo[48979]: alexandre : TTY=pts/0 ; PWD=/home/alexandre ; USER=root ; COMMAND=/bin/journalctl -f
Dec 06 10:49:48 Chunk1 sudo[48979]: pam_unix(sudo:session): session opened for user root(uid=0) by alexandre(uid=1000)
Dec 06 10:50:38 Chunk1 kernel: e1000: enp0s9 NIC Link is Down
Dec 06 10:50:38 Chunk1 kernel: e1000 0000:00:09.0 enp0s9: Reset adapter
Dec 06 10:50:43 Chunk1 kernel:  connection2:0: ping timeout of 5 secs expired, recv timeout 5, last rx 4297654658, last ping 4297659904, now 4297665025

Dec 06 10:50:48 Chunk1 kernel:  session2: session recovery timed out after 5 secs
Dec 06 10:50:49 Chunk1 multipathd[1709]: checker failed path 8:16 in map mpathb
Dec 06 10:50:49 Chunk1 multipathd[1709]: mpathb: remaining active paths: 1
Dec 06 10:50:49 Chunk1 kernel: device-mapper: multipath: 253:4: Failing path 8:16.
Dec 06 10:50:52 Chunk1 kernel:  session10: session recovery timed out after 5 secs
Dec 06 10:50:53 Chunk1 multipathd[1709]: checker failed path 8:32 in map mpatha
Dec 06 10:50:53 Chunk1 multipathd[1709]: mpatha: remaining active paths: 1
Dec 06 10:50:53 Chunk1 kernel: device-mapper: multipath: 253:3: Failing path 8:32.
Dec 06 10:50:54 Chunk1 systemd[1]: NetworkManager-dispatcher.service: Deactivated successfully.

> disconnected (reason 'carrier-changed', sys-iface-state: 'managed')
Dec 06 10:51:33 Chunk1 NetworkManager[696]: <info>  [1733478693.8944] policy: auto-activating connection 'System enp0s9' (93d13955-e9e2-a6bd-df73-12e3c747f122)
Dec 06 10:51:33 Chunk1 NetworkManager[696]: <info>  [1733478693.8948] device (enp0s9): Activation: starting connection 'System enp0s9' (93d13955-e9e2-a6bd-df73-12e3c747f122)
Dec 06 10:51:33 Chunk1 NetworkManager[696]: <info>  [1733478693.8950] device (enp0s9): state change: disconnected -> prepare (reason 'none', sys-iface-state: 'managed')
Dec 06 10:51:33 Chunk1 NetworkManager[696]: <info>  [1733478693.8954] device (enp0s9): state change: prepare -> config (reason 'none', sys-iface-state: 'managed')
Dec 06 10:51:33 Chunk1 NetworkManager[696]: <info>  [1733478693.9118] device (enp0s9): state change: config -> ip-config (reason 'none', sys-iface-state: 'managed')
Dec 06 10:51:33 Chunk1 NetworkManager[696]: <info>  [1733478693.9556] device (enp0s9): state change: ip-config -> ip-check (reason 'none', sys-iface-state: 'managed')

[alexandre@Chunk1 ~]$ sudo watch -n 0.5 multipath -ll
Every 0.5s: multipath -ll                                                         Chunk1: Fri Dec  6 10:53:39 2024
mpatha (360014058f0d8c6e00fa4ca095ade73df) dm-3 LIO-ORG,data-chunk1
size=40G features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=enabled
| `- 3:0:0:0 sdc 8:32 active ready running
`-+- policy='service-time 0' prio=50 status=active
  `- 5:0:0:0 sdd 8:48 active ready running
mpathb (36001405c428a49690dc477d8676ab6ff) dm-4 LIO-ORG,data-chunk1
size=40G features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=enabled
| `- 4:0:0:0 sdb 8:16 active ready running
`-+- policy='service-time 0' prio=50 status=active
  `- 9:0:0:0 sde 8:64 active ready running
  
```

### B. Jouer avec les paramètres

Pour améliorer la rapidité de la bascule (attention si le réseau est instable, ça peut coûter cher), on peut ajouter un peu de conf et déterminer des temps plus courts pour les timeouts par exemple.

Je vous laisse aller lire le détails des options, je vous recommande de tester avec ces paramètres dans `/etc/multipath.conf` :

```conf
defaults {
 user_friendly_names yes
 find_multipaths yes
 path_grouping_policy failover
 no_path_retry 100
 features "1 queue_if_no_path"
 polling_interval 2
 max_polling_interval 8
 fast_io_fail_tmo 3
}
```

🌞 **Resimuler une panne**

```
c 06 11:24:06 Chunk1 iscsid[794]: iscsid: Could not set session22 priority. READ/WRITE throughout and latency could be affected.
Dec 06 11:24:06 Chunk1 iscsid[794]: iscsid: connection20:0 login rejected: initiator failed authorization with target
Dec 06 11:24:06 Chunk1 iscsid[794]: iscsid: Connection20:0 to [target: iqn.2024-12.tp2.b3:data-chunk3, portal: 10.3.2.1,3260] through [iface: default] is shutdown.
Dec 06 11:24:06 Chunk1 iscsid[794]: iscsid: connection20:0 IPC qtask write failed: Broken pipe
Dec 06 11:24:06 Chunk1 iscsid[794]: iscsid: connection21:0 login rejected: initiator failed authorization with target
Dec 06 11:24:06 Chunk1 iscsid[794]: iscsid: Connection21:0 to [target: iqn.2024-12.tp2.b3:data-chunk3, portal: 10.3.2.2,3260] through [iface: default] is shutdown.
Dec 06 11:24:06 Chunk1 iscsid[794]: iscsid: connection21:0 IPC qtask write failed: Broken pipe
Dec 06 11:24:06 Chunk1 iscsid[794]: iscsid: connection17:0 login rejected: initiator failed authorization with target
Dec 06 11:24:06 Chunk1 iscsid[794]: iscsid: Connection17:0 to [target: iqn.2024-12.tp2.b3:data-chunk2, portal: 10.3.1.2,3260] through [iface: default] is shutdown.
Dec 06 11:24:06 Chunk1 iscsid[794]: iscsid: connection17:0 IPC qtask write failed: Broken pipe
Dec 06 11:24:06 Chunk1 iscsid[794]: iscsid: connection19:0 login rejected: initiator failed authorization with target
Dec 06 11:24:06 Chunk1 iscsid[794]: iscsid: Connection19:0 to [target: iqn.2024-12.tp2.b3:data-chunk2, portal: 10.3.2.2,3260] through [iface: default] is shutdown.
Dec 06 11:24:06 Chunk1 iscsid[794]: iscsid: connection19:0 IPC qtask write failed: Broken pipe
Dec 06 11:24:06 Chunk1 iscsid[794]: iscsid: connection22:0 login rejected: initiator failed authorization with target
Dec 06 11:24:06 Chunk1 kernel:  connection18:0: detected conn error (1020)
Dec 06 11:24:06 Chunk1 iscsid[794]: iscsid: Connection22:0 to [target: iqn.2024-12.tp2.b3:data-chunk2, portal: 10.3.1.1,3260] through [iface: default] is shutdown.
Dec 06 11:24:06 Chunk1 iscsid[794]: iscsid: connection22:0 IPC qtask write failed: Broken pipe
Dec 06 11:24:06 Chunk1 iscsid[794]: iscsid: connection18:0 login rejected: initiator failed authorization with target
Dec 06 11:24:06 Chunk1 iscsid[794]: iscsid: Connection18:0 to [target: iqn.2024-12.tp2.b3:data-chunk2, portal: 10.3.2.1,3260] through [iface: default] is shutdown.
Dec 06 11:24:06 Chunk1 iscsid[794]: iscsid: connection18:0 IPC qtask write failed: Broken pipe
Dec 06 11:24:06 Chunk1 iscsid[794]: iscsid: connection13:0 is operational after recovery (6 attempts)
Dec 06 11:24:06 Chunk1 iscsid[794]: iscsid: connection3:0 is operational after recovery (6 attempts)
```

## 5. Replicate

➜ **Répliquer le setup de `chunk1` sur les machines `chunk2` et `chunk3`**

- chaque machine ne doit accéder qu'aux targets iSCSI qui lui sont destinés
  - chaque machine "chunk" a un IQN qui lui est dédié
  - la machine `chunk1`, on lui a donné accès qu'aux IQN `iqn.2024-12.tp2.b3:chunk1` (sur chaque IP)
  - il faut donc faire pareil la machine `chunk2` et sur `chunk3`
- même setup :
  - `iscsiadm` pour les 4 targets iSCSI
  - puis multipath pour en avoir plus que 2
  - vous formatez en `xfs` les deux volumes `mpatha` et `mpathb` et montez sur `/mnt/data_chunk1` et `/mnt/data_chunk2`

🌞 **Preuve du setup**
```
[alexandre@chunck2 ~]$ lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda           8:0    0   80G  0 disk
├─sda1        8:1    0    1G  0 part  /boot
└─sda2        8:2    0   79G  0 part
  ├─rl-root 253:0    0 51.7G  0 lvm   /
  ├─rl-swap 253:1    0    2G  0 lvm   [SWAP]
  └─rl-home 253:2    0 25.2G  0 lvm   /home
sdb           8:16   0   40G  0 disk
└─mpathb    253:3    0   40G  0 mpath
  └─mpathb1 253:5    0   40G  0 part  /mnt/data_chunk2
sdc           8:32   0   40G  0 disk
└─mpatha    253:4    0   40G  0 mpath
  └─mpatha1 253:6    0   40G  0 part  /mnt/data_chunk1
sdd           8:48   0   40G  0 disk
└─mpathb    253:3    0   40G  0 mpath
  └─mpathb1 253:5    0   40G  0 part  /mnt/data_chunk2
sde           8:64   0   40G  0 disk
└─mpatha    253:4    0   40G  0 mpath
  └─mpatha1 253:6    0   40G  0 part  /mnt/data_chunk1
sr0          11:0    1 1024M  0 rom

[alexandre@chunck3 ~]$ lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda           8:0    0   80G  0 disk
├─sda1        8:1    0    1G  0 part  /boot
└─sda2        8:2    0   79G  0 part
  ├─rl-root 253:0    0 51.7G  0 lvm   /
  ├─rl-swap 253:1    0    2G  0 lvm   [SWAP]
  └─rl-home 253:2    0 25.2G  0 lvm   /home
sdb           8:16   0   40G  0 disk
└─mpatha    253:3    0   40G  0 mpath
  └─mpatha1 253:5    0   40G  0 part  /mnt/data_chunk1
sdc           8:32   0   40G  0 disk
└─mpatha    253:3    0   40G  0 mpath
  └─mpatha1 253:5    0   40G  0 part  /mnt/data_chunk1
sdd           8:48   0   40G  0 disk
└─mpathb    253:4    0   40G  0 mpath
  └─mpathb1 253:6    0   40G  0 part  /mnt/data_chunk2
sde           8:64   0   40G  0 disk
└─mpathb    253:4    0   40G  0 mpath
  └─mpathb1 253:6    0   40G  0 part  /mnt/data_chunk2
sr0          11:0    1 1024M  0 rom

[alexandre@chunck2 ~]$ df -h
Filesystem           Size  Used Avail Use% Mounted on
devtmpfs             4.0M     0  4.0M   0% /dev
tmpfs                888M     0  888M   0% /dev/shm
tmpfs                355M  5.1M  350M   2% /run
/dev/mapper/rl-root   52G  1.6G   51G   3% /
/dev/mapper/rl-home   26G  213M   25G   1% /home
/dev/sda1            960M  230M  731M  24% /boot
tmpfs                178M     0  178M   0% /run/user/1000
/dev/mapper/mpatha1   40G  318M   40G   1% /mnt/data_chunk1
/dev/mapper/mpathb1   40G  318M   40G   1% /mnt/data_chunk2

[alexandre@chunck3 ~]$ df -h
Filesystem           Size  Used Avail Use% Mounted on
devtmpfs             4.0M     0  4.0M   0% /dev
tmpfs                888M     0  888M   0% /dev/shm
tmpfs                355M  9.5M  346M   3% /run
/dev/mapper/rl-root   52G  1.6G   51G   3% /
/dev/sda1            960M  230M  731M  24% /boot
/dev/mapper/rl-home   26G  213M   25G   1% /home
tmpfs                178M     0  178M   0% /run/user/1000
/dev/mapper/mpathb1   40G  318M   40G   1% /mnt/data_chunk2
/dev/mapper/mpatha1   40G  318M   40G   1% /mnt/data_chunk1

               Interface:
                **********
                Iface Name: default
                Iface Transport: tcp
                Iface Initiatorname: iqn.2024-12.tp2.b3:data-chunk2:chunk2-initiator
                Iface IPaddress: 10.3.2.102
                Iface HWaddress: default
                Iface Netdev: default
                SID: 9
                iSCSI Connection State: LOGGED IN
                iSCSI Session State: LOGGED_IN
                Internal iscsid Session State: NO CHANGE
                *********
                Timeouts:
                *********
                Recovery Timeout: 3
                Target Reset Timeout: 30
                LUN Reset Timeout: 30
                Abort Timeout: 15
                *****
                CHAP:
                *****
                username: <empty>
                password: ********
                username_in: <empty>
                password_in: ********
                ************************
                Negotiated iSCSI params:
                ************************
                HeaderDigest: None
                DataDigest: None
                MaxRecvDataSegmentLength: 262144
                MaxXmitDataSegmentLength: 262144
                FirstBurstLength: 65536
                MaxBurstLength: 262144
                ImmediateData: Yes
                InitialR2T: Yes
                MaxOutstandingR2T: 1
                ************************
                Attached SCSI devices:
                ************************
                Host Number: 7  State: running
                scsi7 Channel 00 Id 0 Lun: 0
                        Attached scsi disk sde          State: running

                **********
                Interface:
                **********
                Iface Name: default
                Iface Transport: tcp
                Iface Initiatorname: iqn.2024-12.tp2.b3:data-chunk3:chunk3-initiator
                Iface IPaddress: 10.3.2.103
                Iface HWaddress: default
                Iface Netdev: default
                SID: 8
                iSCSI Connection State: LOGGED IN
                iSCSI Session State: LOGGED_IN
                Internal iscsid Session State: NO CHANGE
                *********
                Timeouts:
                *********
                Recovery Timeout: 3
                Target Reset Timeout: 30
                LUN Reset Timeout: 30
                Abort Timeout: 15
                *****
                CHAP:
                *****
                username: <empty>
                password: ********
                username_in: <empty>
                password_in: ********
                ************************
                Negotiated iSCSI params:
                ************************
                HeaderDigest: None
                DataDigest: None
                MaxRecvDataSegmentLength: 262144
                MaxXmitDataSegmentLength: 262144
                FirstBurstLength: 65536
                MaxBurstLength: 262144
                ImmediateData: Yes
                InitialR2T: Yes
                MaxOutstandingR2T: 1
                ************************
                Attached SCSI devices:
                ************************
                Host Number: 6  State: running
                scsi6 Channel 00 Id 0 Lun: 0
                        Attached scsi disk sde          State: running

[alexandre@chunck3 ~]$ sudo multipath -ll
mpatha (36001405f75f0cac1edf4159bfe711adb) dm-3 LIO-ORG,data-chunk3
size=40G features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 3:0:0:0 sdb 8:16 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 4:0:0:0 sdc 8:32 active ready running
mpathb (36001405eca4934daed348bda31b8d48a) dm-4 LIO-ORG,data-chunk3
size=40G features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 5:0:0:0 sdd 8:48 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 6:0:0:0 sde 8:64 active ready running


[alexandre@chunck2 ~]$ sudo multipath -ll
mpatha (36001405f034f40f22544da3a4bb5f41d) dm-4 LIO-ORG,data-chunk2
size=40G features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 7:0:0:0 sde 8:64 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 5:0:0:0 sdc 8:32 active ready running
mpathb (3600140522ca7af2d3bc4760b47317411) dm-3 LIO-ORG,data-chunk2
size=40G features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 6:0:0:0 sdb 8:16 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 4:0:0:0 sdd 8:48 active ready running

```
# III. Distributed filesystem

🌞 **Installer les paquets nécessaires pour avoir un Moose Master**
```
curl "http://repository.moosefs.com/MooseFS-3-el9.repo" > /etc/yum.repos.d/MooseFS.repo
[root@Master ~]# yum install moosefs-master moosefs-cgi moosefs-cgiserv moosefs-cli
```
🌞 **Démarrez les services du Moose Master**
```
[root@Master ~]# sudo systemctl start moosefs-master
[root@Master ~]# sudo systemctl enable moosefs-master
```
🌞 **Ouvrez les ports firewall**
```
[root@Master ~]# sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --permanent --add-port=9425/tcp
sudo firewall-cmd --permanent --add-port=9419/tcp
sudo firewall-cmd --permanent --add-port=9420/tcp
sudo firewall-cmd --permanent --add-port=9421/tcp
sudo firewall-cmd --reload
```
➜ **WebUI moche dispo sur le port `9425/tcp`**

## 2. Chunk servers

> ***Sur les 3 machines "chunk" :  `chunk1.tp2.b3`, `chunk2.tp2.b3` et `chunk3.tp2.b3`.***

🌞 **Installer les paquets nécessaires pour avoir un Moose Chunk Server**

```
[root@Chunk1 ~]# curl "https://repository.moosefs.com/RPM-GPG-KEY-MooseFS" > /etc/pki/rpm-gpg/RPM-GPG-KEY-MooseFS

[root@Chunk1 ~]# curl "http://repository.moosefs.com/MooseFS-3-el9.repo" > /etc/yum.repos.d/MooseFS.repo

[root@Chunk1 ~]# yum install moosefs-chunkserver

```

🌞 **Modifier la conf du Chunk Server**
```
[root@Chunk1 ~]# sudo nano /etc/mfs/mfschunkserver.cfg
MASTER_HOST = master.local
```
🌞 **Faire appartenir les partitions à partager à l'utilisateur `mfs`**

```
[root@Chunk1 ~]# sudo chown -R mfs:mfs /mnt/data_chunk1
sudo chown -R mfs:mfs /mnt/data_chunk2
```
🌞 **Modifier la conf des disques du Chunk Server**
```
[root@Chunk1 ~]# sudo nano /etc/mfs/mfshdd.cfg
/mnt/data_chunk1
/mnt/data_chunk2
```
🌞 **Démarrez les services du Moose Chunk Server**

```
sudo systemctl start moosefs-chunkserver
```

## 3. Consume

Dernière machine : celui qui consomme le stockage à la fin, notre `web.tp2.b3`.

Sur cette machine :

- on va installer les outils Moose pour pouvoir accéder à la partition proposé par le Master
- tout fichier stocké sur cette partition sera répliqué
- déploiement fast d'un ptit serveur web NGINX qui stocke ses données sur le volume répliqué (un bête `index.html`)

## 1. Monter la partition Moose

> ***On est sur `web.tp2.b3`.***

🌞 **Installer les paquets nécessaires pour avoir un Moose Client**


```
[alexandre@web ~]$ curl "https://repository.moosefs.com/RPM-GPG-KEY-MooseFS" > /etc/pki/rpm-gpg/RPM-GPG-KEY-MooseFS

[root@web ~]# curl "http://repository.moosefs.com/MooseFS-3-el9.repo" > /etc/yum.repos.d/MooseFS.repo

[root@web ~]# yum install moosefs-client
```

🌞 **Monter la partition Moose**
```
[root@web ~]# sudo mkdir -p /mnt/www
[root@web ~]# ls -ld /mnt/www
drwxr-xr-x. 2 root root 6 Dec  6 15:25 /mnt/www
[root@web ~]# sudo mfsmount /mnt/www -H 10.3.250.1
mfsmaster accepted connection with parameters: read-write,restricted_ip,admin ; root mapped to root:root


```

🌞 **Preuve et test d'écriture**
```
[root@web ~]# lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda           8:0    0   80G  0 disk
├─sda1        8:1    0    1G  0 part /boot
└─sda2        8:2    0   79G  0 part
  ├─rl-root 253:0    0 51.7G  0 lvm  /
  ├─rl-swap 253:1    0    2G  0 lvm  [SWAP]
  └─rl-home 253:2    0 25.2G  0 lvm  /home
sr0          11:0    1 1024M  0 rom

lsblk ne trouve pas le /mnt/www/
j'ai eu des soucis au niveau de l'installation sur master
```
## 2. NGINX

🌞 **Installer et configurer NGINX sur la machine `web.tp2.b3`**

- paquet `nginx`
- ouvrez le port `80/tcp` dans le firewall
- créez un fichier `index.html` dans `/mnt/www/`
- créez un fichier de conf minimal qui permet de servir ce site web
- je veux le fichier de conf dans le compte-rendu

🌞 **Démarrer le service NGINX**

🌞 **Prouvez avec un `curl` que le site est actif**

**Well that's a highly-available little storage setup !** :d

![haha](./img/ha.png)