 Create an LVM logical volume for the new share:

crm cluster run "lvcreate -n share2 -L 20G nfs"

Update the file /etc/drbd.d/nfs.res to add the new volume under the existing volumes:

   volume 2 {
      device           /dev/drbd2;
      disk             /dev/nfs/share2;
      meta-disk        internal;
   }

Copy the updated file to the other nodes:

csync2 -xv

Initialize the metadata storage for the new volume:

crm cluster run "drbdadm create-md nfs/2 --force"

Update the nfs configuration to create the new device:

crm cluster run "drbdadm adjust nfs"

Skip the initial synchronization for the new device:

drbdadm new-current-uuid --clear-bitmap nfs/2

The NFS cluster resources might have moved to another node since they were created. Check the DRBD status with drbdadm status nfs, and make a note of which node is in the Primary role.

On the node that is in the Primary role, create an ext4 file system on /dev/drbd2:

mkfs.ext4 /dev/drbd2

Start the crm interactive shell:

crm configure

Create a primitive for the file system to be exported on /dev/drbd2:

primitive fs-nfs-share2 Filesystem \
  params device="/dev/drbd2" directory="/srv/nfs/share2" fstype=ext4

Add the new file system resource to the g-nfs group before the nfsserver resource:

modgroup g-nfs add fs-nfs-share2 before nfsserver

Create a primitive for NFS exports from the new share:

primitive exportfs-nfs2 exportfs \
  params directory="/srv/nfs/share2" \
  options="rw,mountpoint" clientspec="192.168.1.0/24" fsid=102 \
  op monitor interval=30s timeout=90s

Add the new NFS export resource to the g-nfs group before the vip-nfs resource:

modgroup g-nfs add exportfs-nfs2 before vip-nfs

Commit this configuration:

commit

Leave the crm interactive shell:

quit

Check the status of the cluster. The resources in the g-nfs group should appear in the following order:

crm status
[...]
Full List of Resources
  [...]
  * Resource Group: g-nfs:
    * fs-nfs-state    (ocf:heartbeat:Filesystem):   Started alice
    * fs-nfs-share    (ocf:heartbeat:Filesystem):   Started alice
    * fs-nfs-share2   (ocf:heartbeat:Filesystem):   Started alice
    * nfsserver       (ocf:heartbeat:nfsserver):    Started alice
    * exportfs-nfs    (ocf:heartbeat:exportfs):     Started alice
    * exportfs-nfs2   (ocf:heartbeat:exportfs):     Started alice
    * vip-nfs         (ocf:heartbeat:IPaddr2):      Started alice

Confirm that the NFS exports are set up properly:

exportfs -v
/srv/nfs/share   IP_ADDRESS_OF_CLIENT(OPTIONS)
/srv/nfs/share2  IP_ADDRESS_OF_CLIENT(OPTIONS)

