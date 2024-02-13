# On all Cluster Nodes, Change LVM System ID.
```
vi /etc/lvm/lvm.conf
```
#### line 1227 : uncomment and change
system_id_source = "uname"
[3]	On a Node in Cluster, Set LVM on shared storage.
[sdb] on the example below is shared storage from ISCSI Target.
#### current session
```
iscsiadm -m session -o show
```
tcp: [1] 10.0.0.30:3260,1 iqn.2022-01.world.srv:dlp.target01 (non-flash)
#### discover
```
iscsiadm -m discovery -t sendtargets -p 10.0.0.30
```
10.0.0.30:3260,1 iqn.2022-01.world.srv:dlp.target01
10.0.0.30:3260,1 iqn.2022-01.world.srv:dlp.target02
#### login
```
iscsiadm -m node --login --target iqn.2022-01.world.srv:dlp.target02
```
```
iscsiadm -m session -o show
```
tcp: [1] 10.0.0.30:3260,1 iqn.2022-01.world.srv:dlp.target01 (non-flash)
tcp: [2] 10.0.0.30:3260,1 iqn.2022-01.world.srv:dlp.target02 (non-flash)
#### set LVM
```
parted --script /dev/sdb "mklabel gpt"
```
```
parted --script /dev/sdb "mkpart primary 0% 100%"
```
```
parted --script /dev/sdb "set 1 lvm on"
```
#### create physical volume
```
pvcreate /dev/sdb1
```
  Physical volume "/dev/sdb1" successfully created.

#### create volume group
```
vgcreate vg_ha /dev/sdb1
```
  Volume group "vg_ha" successfully created with system ID node01.srv.world

#### confirm the value of [System ID] equals the value of [$ uname -n]
```
vgs -o+systemid
```
  VG              #PV #LV #SN Attr   VSize    VFree    System ID
  ubuntu-vg         1   1   0 wz--n-  <28.00g 1020.00m
  vg_ha             1   0   0 wz--n-   <9.98g   <9.98g node01.srv.world

#### create logical volume
```
lvcreate -l 100%FREE -n lv_ha vg_ha
```
  Logical volume "lv_ha" created.

#### format with ext4
```
mkfs.ext4 /dev/vg_ha/lv_ha
```
[4]	On other Nodes except a Node of [3], Scan LVM volumes to find new volume.
```
iscsiadm -m session -o show
```
tcp: [1] 10.0.0.30:3260,1 iqn.2022-01.world.srv:dlp.target01 (non-flash)
```
iscsiadm -m discovery -t sendtargets -p 10.0.0.30
```
10.0.0.30:3260,1 iqn.2022-01.world.srv:dlp.target01
10.0.0.30:3260,1 iqn.2022-01.world.srv:dlp.target02
```
iscsiadm -m node --login --target iqn.2022-01.world.srv:dlp.target02
```
```
iscsiadm -m session -o show
```
tcp: [1] 10.0.0.30:3260,1 iqn.2022-01.world.srv:dlp.target01 (non-flash)
tcp: [2] 10.0.0.30:3260,1 iqn.2022-01.world.srv:dlp.target02 (non-flash)
```
lvm pvscan --cache --activate ay
```
  pvscan[1150] PV /dev/sda1 ignore foreign VG.
  pvscan[1150] PV /dev/vda3 online, VG ubuntu-vg is complete.
  pvscan[1150] PV /dev/vdb1 online, VG ceph-5464aa8c-7249-47e7-a37b-d24e10457f0f is complete.
  pvscan[1150] VG ubuntu-vg run autoactivation.
[5]	On a Node of [3], Set shared storage as a Cluster resource.
#### [lvm_ha] : any name
#### [vgname=***] : volume group name
#### [--group] : any name
```
pcs resource create lvm_ha ocf:heartbeat:LVM-activate vgname=vg_ha vg_access_mode=system_id --group ha_group
```
#### confirm status
#### OK if LVM resource is [Started]
```
pcs status
```
Cluster name: ha_cluster
Cluster Summary:
  * Stack: corosync
  * Current DC: node01.srv.world (version 2.1.2-ada5c3b36e2) - partition with quorum
  * Last updated: Thu Sep 15 01:30:33 2022
  * Last change:  Thu Sep 15 01:30:21 2022 by root via cibadmin on node01.srv.world
  * 2 nodes configured
  * 2 resource instances configured

Node List:
  * Online: [ node01.srv.world node02.srv.world ]

Full List of Resources:
  * scsi-shooter        (stonith:fence_scsi):    Started node01.srv.world
  * Resource Group: ha_group:
    * lvm_ha    (ocf:heartbeat:LVM-activate):    Started node02.srv.world

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
