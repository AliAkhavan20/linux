# On all Cluster Nodes, Install NFS tools.
```
apt -y install nfs-kernel-server nfs-common
```
[2]	On a Node that LVM shared storage is active in Cluster, Add NFS resource.
[/dev/vg_ha/lv_ha] on the example below is LVM shared storage.
#### current status
```
pcs status
```
Cluster name: ha_cluster
Cluster Summary:
  * Stack: corosync
  * Current DC: node01.srv.world (version 2.1.2-ada5c3b36e2) - partition with quorum
  * Last updated: Thu Sep 15 01:52:09 2022
  * Last change:  Thu Sep 15 01:52:02 2022 by root via cibadmin on node01.srv.world
  * 2 nodes configured
  * 2 resource instances configured

Node List:
  * Online: [ node01.srv.world node02.srv.world ]

Full List of Resources:
  * scsi-shooter        (stonith:fence_scsi):    Started node01.srv.world
  * Resource Group: ha_group:
    * lvm_ha    (ocf:heartbeat:LVM-activate):    Started node01.srv.world

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled

#### create a directory for NFS filesystem
```
mkdir /home/nfs-share
```
#### set Filesystem resource
#### [nfs_share] : any name
#### [device=***] : shared storage
#### [directory=***] : mount point
#### [--group ***] : set in the same group with shared storage
```
pcs resource create nfs_share ocf:heartbeat:Filesystem device=/dev/vg_ha/lv_ha directory=/home/nfs-share fstype=ext4 --group ha_group
```
```
pcs status
```
Cluster name: ha_cluster
Cluster Summary:
  * Stack: corosync
  * Current DC: node02.srv.world (version 2.1.2-ada5c3b36e2) - partition with quorum
  * Last updated: Thu Sep 15 04:12:28 2022
  * Last change:  Thu Sep 15 04:12:23 2022 by root via cibadmin on node01.srv.world
  * 2 nodes configured
  * 3 resource instances configured

Node List:
  * Online: [ node01.srv.world node02.srv.world ]

Full List of Resources:
  * scsi-shooter        (stonith:fence_scsi):    Started node01.srv.world
  * Resource Group: ha_group:
    * lvm_ha    (ocf:heartbeat:LVM-activate):    Started node01.srv.world
    * nfs_share (ocf:heartbeat:Filesystem):      Started node01.srv.world

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled

#### mount automatically on a node that resources started
```
df -hT /home/nfs-share
```
Filesystem              Type  Size  Used Avail Use% Mounted on
/dev/mapper/vg_ha-lv_ha ext4  9.8G   24K  9.3G   1% /home/nfs-share

#### set nfsserver resource
#### [nfs_daemon] : any name
#### [nfs_shared_infodir=***] : specify a directory that NFS server related files are located
```
pcs resource create nfs_daemon ocf:heartbeat:nfsserver nfs_shared_infodir=/home/nfs-share/nfsinfo nfs_no_notify=true --group ha_group
```
#### set IPaddr2 resource
#### virtual IP address clients access to NFS service
```
pcs resource create nfs_vip ocf:heartbeat:IPaddr2 ip=10.0.0.60 cidr_netmask=24 --group ha_group
```
```
pcs status
```
Cluster name: ha_cluster
Cluster Summary:
  * Stack: corosync
  * Current DC: node02.srv.world (version 2.1.2-ada5c3b36e2) - partition with quorum
  * Last updated: Thu Sep 15 04:13:57 2022
  * Last change:  Thu Sep 15 04:13:45 2022 by root via cibadmin on node01.srv.world
  * 2 nodes configured
  * 5 resource instances configured

Node List:
  * Online: [ node01.srv.world node02.srv.world ]

Full List of Resources:
  * scsi-shooter        (stonith:fence_scsi):    Started node01.srv.world
  * Resource Group: ha_group:
    * lvm_ha    (ocf:heartbeat:LVM-activate):    Started node01.srv.world
    * nfs_share (ocf:heartbeat:Filesystem):      Started node01.srv.world
    * nfs_daemon        (ocf:heartbeat:nfsserver):       Started node01.srv.world
    * nfs_vip   (ocf:heartbeat:IPaddr2):         Started node01.srv.world

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
[3]	On an active Node NFS filesystem mounted, set exportfs setting.
#### create a directory for exportfs
```
mkdir -p /home/nfs-share/nfs-root/share01
```
#### set exportfs resource
#### [nfs_root] : any name
#### [clientspec=*** options=*** directory=***] : exports setting
#### [fsid=0] : root point on NFSv4
```
pcs resource create nfs_root ocf:heartbeat:exportfs clientspec=10.0.0.0/255.255.255.0 options=rw,sync,no_root_squash directory=/home/nfs-share/nfs-root fsid=0 --group ha_group
```
#### set exportfs resource
```
pcs resource create nfs_share01 ocf:heartbeat:exportfs clientspec=10.0.0.0/255.255.255.0 options=rw,sync,no_root_squash directory=/home/nfs-share/nfs-root/share01 fsid=1 --group ha_group
```
```
pcs status
```
Cluster name: ha_cluster
Cluster Summary:
  * Stack: corosync
  * Current DC: node02.srv.world (version 2.1.2-ada5c3b36e2) - partition with quorum
  * Last updated: Thu Sep 15 04:15:02 2022
  * Last change:  Thu Sep 15 04:14:55 2022 by root via cibadmin on node02.srv.world
  * 2 nodes configured
  * 7 resource instances configured

Node List:
  * Online: [ node01.srv.world node02.srv.world ]

Full List of Resources:
  * scsi-shooter        (stonith:fence_scsi):    Started node01.srv.world
  * Resource Group: ha_group:
    * lvm_ha    (ocf:heartbeat:LVM-activate):    Started node01.srv.world
    * nfs_share (ocf:heartbeat:Filesystem):      Started node01.srv.world
    * nfs_daemon        (ocf:heartbeat:nfsserver):       Started node01.srv.world
    * nfs_vip   (ocf:heartbeat:IPaddr2):         Started node01.srv.world
    * nfs_root  (ocf:heartbeat:exportfs):        Started node01.srv.world
    * nfs_share01       (ocf:heartbeat:exportfs):        Started node01.srv.world

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
```
showmount -e
```
Export list for node01.srv.world:
/home/nfs-share/nfs-root         10.0.0.0/255.255.255.0
/home/nfs-share/nfs-root/share01 10.0.0.0/255.255.255.0
[4]	Verify settings to access to virtual IP address with NFS from any client computer.
```
mount -t nfs4 10.0.0.60:share01 /mnt
```
```
df -hT /mnt
```
Filesystem         Type  Size  Used Avail Use% Mounted on
10.0.0.60:/share01 nfs4  9.8G  512K  9.3G   1% /mnt
