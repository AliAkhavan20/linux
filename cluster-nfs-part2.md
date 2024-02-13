# Part2:
## Configure ISCSI Target and Create a storage for fence device, refer to here.
Configure Storage Server with iSCSI.
Storage server with iSCSI on network is called iSCSI Target, Client Host that connects to iSCSI Target is called iSCSI Initiator.
This example is based on the environment like follows.
#### [1]	Install administration tool.
```
apt -y install targetcli-fb
```
#### [2]	Configure iSCSI Target.
For example, Create an disk-image under the [/var/lib/iscsi_disks] directory and set it as a SCSI device.
#### create a directory
```
mkdir /var/lib/iscsi_disks
```
#### enter the admin console
```
targetcli
```
Warning: Could not load preferences file /root/.targetcli/prefs.bin.
targetcli shell version 2.1.53
Copyright 2011-2013 by Datera, Inc and others.
For help on commands, type 'help'.
```
cd /backstores/fileio
```
#### create a disk-image with the name [disk01] on [/var/lib/iscsi_disks/disk01.img] with 10G
```
create disk01 /var/lib/iscsi_disks/disk01.img 10G
```
Created fileio disk01 with size 10737418240
```
cd /iscsi
```
### create a target
#### naming rule : [ iqn.(year)-(month).(reverse of domain name):(any name you like) ]
```
create iqn.2022-04.world.srv:dlp.target01
```
Created target iqn.2022-04.world.srv:dlp.target01.
Created TPG 1.
Global pref auto_add_default_portal=true
Created default portal listening on all IPs (0.0.0.0), port 3260.
```
cd iqn.2022-04.world.srv:dlp.target01/tpg1/luns
```
#### set LUN
```
/iscsi/iqn.20...t01/tpg1/luns> create /backstores/fileio/disk01
```
Created LUN 0.
```
/iscsi/iqn.20...t01/tpg1/luns> cd ../acls
```
#### set ACL (it's the IQN of an initiator you permit to connect)
/iscsi/iqn.20...t01/tpg1/acls> create iqn.2022-04.world.srv:node01.initiator01 
Created Node ACL for iqn.2022-04.world.srv:node01.initiator01
Created mapped LUN 0.
/iscsi/iqn.20...t01/tpg1/acls> cd iqn.2022-04.world.srv:node01.initiator01 

#### set UserID and Password for authentication
```
/iscsi/iqn.20...w.initiator01> set auth userid=username 
```
Parameter userid is now 'username'.
```
/iscsi/iqn.20...w.initiator01> set auth password=password
```
Parameter password is now 'password'.
```
/iscsi/iqn.20...w.initiator01> exit
```
Global pref auto_save_on_exit=true
Configuration saved to /etc/rtslib-fb-target/saveconfig.json

#### after configuration above, the target enters in listening like follows
```
root@dlp:~# ss -napt | grep 3260
```
LISTEN 0      256          0.0.0.0:3260       0.0.0.0:*
```
root@dlp:~# systemctl enable rtslib-fb-targetctl
```
## Login to iscsi on all nodes

Configure iSCSI Initiator to connect to iSCSI Target.
```
apt -y install open-iscsi
```
```
vi /etc/iscsi/initiatorname.iscsi
```
#### change to the same IQN you set on the iSCSI target server
InitiatorName=iqn.2022-04.world.srv:node01.initiator01
```
vi /etc/iscsi/iscsid.conf
```
#### line 58 : uncomment
node.session.auth.authmethod = CHAP
#### line 69,70: uncomment and specify the username and password you set on the iSCSI target server
node.session.auth.username = username
node.session.auth.password = password
```
systemctl restart iscsid open-iscsi
```
#### discover target
```
iscsiadm -m discovery -t sendtargets -p 10.0.0.30
```
10.0.0.30:3260,1 iqn.2022-04.world.srv:dlp.target01
#### confirm status after discovery
```
iscsiadm -m node -o show
```
#### BEGIN RECORD 2.1.5
```
node.name = iqn.2022-04.world.srv:dlp.target01
node.tpgt = 1
node.startup = manual
node.leading_login = No
iface.iscsi_ifacename = default
.....
.....
node.conn[0].iscsi.DataDigest = None
node.conn[0].iscsi.IFMarker = No
node.conn[0].iscsi.OFMarker = No
```
#### END RECORD

#### login to the target
```
iscsiadm -m node --login -p 10.0.0.30
```
Logging in to [iface: default, target: iqn.2022-04.world.srv:dlp.target01, portal: 10.0.0.30,3260]
Login to [iface: default, target: iqn.2022-04.world.srv:dlp.target01, portal: 10.0.0.30,3260] successful.

#### confirm the established session
```
iscsiadm -m session -o show
```
tcp: [1] 10.0.0.30:3260,1 iqn.2022-04.world.srv:dlp.target01 (non-flash)
#### if enable automatic login for system starting, set like follows
```
iscsiadm -m node -p 10.0.0.30 -o update -n node.startup -v automatic
```
#### confirm the partitions
```
cat /proc/partitions
```
major minor  #blocks  name

   7        0      81868 loop0
   7        1      63380 loop1
   7        2      45748 loop2
 252        0   31457280 sda
 252        1       1024 sda1
 252        2    2097152 sda2
 252        3   29357056 sda3
 253        0   28311552 dm-0
   8        0   10485760 sdb
#### added new device provided from the target server as [sdb]
[2]	After setting iSCSI device, configure on Initiator to use it like follows.
#### create label
```
parted --script /dev/sdb "mklabel gpt"
```
#### create partition
```
parted --script /dev/sdb "mkpart primary 0% 100%"
```
#### format with ext4
```
mkfs.ext4 /dev/sdb1
```
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 2617344 4k blocks and 655360 inodes
Filesystem UUID: 6eec40d4-a75c-4a6e-97eb-cfb545d696ee
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done
```
mount /dev/sdb1 /mnt
```
```
df -hT
```
Filesystem                        Type   Size  Used Avail Use% Mounted on
tmpfs                             tmpfs  393M  1.1M  392M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv ext4    27G  5.7G   20G  23% /
tmpfs                             tmpfs  2.0G     0  2.0G   0% /dev/shm
tmpfs                             tmpfs  5.0M     0  5.0M   0% /run/lock
/dev/sda2                         ext4   2.0G  125M  1.7G   7% /boot
tmpfs                             tmpfs  393M  4.0K  393M   1% /run/user/0
/dev/sdb1                         ext4   9.8G   24K  9.3G   1% /mnt


created ISCSI storage as IQN [iqn.2021-06.world.srv:dlp.target01] with [1M] size.

[3]	On all Cluster Nodes, Install SCSI Fence Agent.
```
apt -y install fence-agents-base
```
[4]	Configure Fencing on a Node.
[sda] of the example below is the storage from ISCSI target.
#### confirm disk ID
```
ll /dev/disk/by-id | grep sda | grep wwn
```
lrwxrwxrwx 1 root root   9 Sep 15 00:30 wwn-0x60014054a2b171ec9974ef5a736a642d -> ../../sda

#### set fencing
#### [scsi-shooter] : any name
#### [pcmk_host_list=***] : specify cluster nodes
#### [devices=***] : disk ID
```
pcs stonith create scsi-shooter fence_scsi pcmk_host_list="node01.srv.world node02.srv.world" devices=/dev/disk/by-id/wwn-0x60014054a2b171ec9974ef5a736a642d meta provides=unfencing
```
#### show config
```
pcs stonith config scsi-shooter
```
 Resource: scsi-shooter (class=stonith type=fence_scsi)
  Attributes: devices=/dev/disk/by-id/wwn-0x60014054a2b171ec9974ef5a736a642d pcmk_host_list="node01.srv.world node02.srv.world"
  Meta Attrs: provides=unfencing
  Operations: monitor interval=60s (scsi-shooter-monitor-interval-60s)

#### show status
#### OK if the status of fence device is [Started]
```
pcs status
```
Cluster name: ha_cluster
Cluster Summary:
  * Stack: corosync
  * Current DC: node01.srv.world (version 2.1.2-ada5c3b36e2) - partition with quorum
  * Last updated: Thu Sep 15 00:42:03 2022
  * Last change:  Thu Sep 15 00:41:27 2022 by root via cibadmin on node01.srv.world
  * 2 nodes configured
  * 1 resource instance configured

Node List:
  * Online: [ node01.srv.world node02.srv.world ]

Full List of Resources:
  * scsi-shooter        (stonith:fence_scsi):    Started node01.srv.world

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
#### [5]	Try to test fencing.
```
pcs status
```
Cluster name: ha_cluster
Cluster Summary:
  * Stack: corosync
  * Current DC: node01.srv.world (version 2.1.2-ada5c3b36e2) - partition with quorum
  * Last updated: Thu Sep 15 00:42:55 2022
  * Last change:  Thu Sep 15 00:41:27 2022 by root via cibadmin on node01.srv.world
  * 2 nodes configured
  * 1 resource instance configured

Node List:
  * Online: [ node01.srv.world node02.srv.world ]

Full List of Resources:
  * scsi-shooter        (stonith:fence_scsi):    Started node01.srv.world

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled

#### fencing
```
pcs stonith fence node01.srv.world
```
Node: node01.srv.world fenced
#### target node turns to [OFFLINE] and it will be restarted
```
pcs status
```
Cluster name: ha_cluster
Cluster Summary:
  * Stack: corosync
  * Current DC: node02.srv.world (version 2.1.2-ada5c3b36e2) - partition with quorum
  * Last updated: Thu Sep 15 00:43:48 2022
  * Last change:  Thu Sep 15 00:41:27 2022 by root via cibadmin on node01.srv.world
  * 2 nodes configured
  * 1 resource instance configured

Node List:
  * Online: [ node02.srv.world ]
  * OFFLINE: [ node01.srv.world ]

Full List of Resources:
  * scsi-shooter        (stonith:fence_scsi):    Started node02.srv.world

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled

#### after rebooting, if you manually start the node, do like follows
```
pcs cluster start node01.srv.world
```
