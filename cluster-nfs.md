
# Set NFS Cluster Resource and Configure Active/Passive NFS Server

### Install and Configure Linux High Availability System, Pacemaker
> [1]	On all Nodes, Install Pacemaker and configure some settings.
```
apt -y install pacemaker pcs resource-agents
systemctl stop pacemaker corosync
systemctl enable --now pcsd
```
#### set cluster admin password
```
passwd hacluster
```
#### Changing password for user hacluster.
New password:
Retype new password:
passwd: all authentication tokens updated successfully.

> [2]	On a Node, Configure basic Cluster settings.
### authorize among nodes
```
pcs host auth node01.srv.world node02.srv.world
```
Username: hacluster
Password:
node01.srv.world: Authorized
node02.srv.world: Authorized
### configure cluster
```
pcs cluster setup ha_cluster node01.srv.world node02.srv.world
```
> No addresses specified for host 'node01.srv.world', using 'node01.srv.world'
No addresses specified for host 'node02.srv.world', using 'node02.srv.world'
Destroying cluster on hosts: 'node01.srv.world', 'node02.srv.world'...
node02.srv.world: Successfully destroyed cluster
node01.srv.world: Successfully destroyed cluster
Requesting remove 'pcsd settings' from 'node01.srv.world', 'node02.srv.world'
node01.srv.world: successful removal of the file 'pcsd settings'
node02.srv.world: successful removal of the file 'pcsd settings'
Sending 'corosync authkey', 'pacemaker authkey' to 'node01.srv.world', 'node02.srv.world'
node01.srv.world: successful distribution of the file 'corosync authkey'
node01.srv.world: successful distribution of the file 'pacemaker authkey'
node02.srv.world: successful distribution of the file 'corosync authkey'
node02.srv.world: successful distribution of the file 'pacemaker authkey'
Sending 'corosync.conf' to 'node01.srv.world', 'node02.srv.world'
node01.srv.world: successful distribution of the file 'corosync.conf'
node02.srv.world: successful distribution of the file 'corosync.conf'
Cluster has been successfully set up.

#### start services for cluster
```
pcs cluster start --all
```
> node01.srv.world: Starting Cluster...
> node02.srv.world: Starting Cluster...
#### set auto-start
```
pcs cluster enable --all
```
> node01.srv.world: Cluster Enabled
> node02.srv.world: Cluster Enabled
#### show status
```
pcs cluster status
```
Cluster Status:
 Cluster Summary:
   * Stack: corosync
   * Current DC: node01.srv.world (version 2.1.2-ada5c3b36e2) - partition with quorum
   * Last updated: Thu Sep 15 00:17:19 2022
   * Last change:  Thu Sep 15 00:17:13 2022 by hacluster via crmd on node01.srv.world
   * 2 nodes configured
   * 0 resource instances configured
 Node List:
   * Online: [ node01.srv.world node02.srv.world ]

PCSD Status:
  node01.srv.world: Online
  node02.srv.world: Online
```
pcs status corosync
```
Membership information
----------------------
    Nodeid      Votes Name
         1          1 node01.srv.world (local)
         2          1 node02.srv.world
