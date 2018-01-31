---
layout: post
title: Introduction to PostgreSQL Automatic Failover
draft: true
---

“PostgreSQL Automatic Failover” (aka. PAF : http://clusterlabs.github.io/PAF/) is a Resource Agent providing service High Availability for PostgreSQL, based on Pacemaker and Corosync.

While the automatic failover influence the recovery time objective (RTO), the recovery point objective (RPO) is balanced by the PostgreSQL Streaming Replication.

Let's see in this post how to quickly install it.

<!--MORE-->

-----

# [](#introduction)Introduction

Pacemaker is nowadays the industry reference for High Availability. The Pacemaker + Corosync stack is able to detect failures on various services and automatically decide to failover the failing resource to another node when possible.

To be able to manage a specific service resource, Pacemaker interact with it through a so-called "Resource Agent". A Resource Agent is an external program that abstracts the service it provides and present a consistent view to the cluster.

PostgreSQL Automatic Failover (aka. PAF : http://clusterlabs.github.io/PAF/) is a Resource Agent dedicated to PostgreSQL. Its original wish is to keep a clear limit between the Pacemaker administration and the PostgreSQL one, to keep things simple, documented and yet powerful.

Once your PostgreSQL cluster built using internal Streaming Replication, PAF is able to expose to Pacemaker what is the current status of the PostgreSQL instance on each node: master, slave, stopped, catching up, etc. Should a failure occur on the master, Pacemaker will try to recover it by default. Should the failure be non-recoverable, PAF allows the slaves to be able to elect the best of them (the closest one to the old master) and promote it as the new master. All of this thanks to the robust, feature-full and most importantly experienced project: Pacemaker.

-----

# [](#fencing)Fencing

Fencing is one of the mandatory piece you need when building an highly available cluster for your database.

It's the ability to isolate a node from the cluster.

Should an issue happen where the master does not answer to the cluster, successful fencing is the only way to be sure what is its status: shutdown or not able to accept new work or touch data. It avoids countless situations where you end up with split brain scenarios or data corruption

The documentation provides good practices and examples : http://clusterlabs.github.io/PAF/fencing.html

-----

# [](#quick-start)Quick start

The documentation also provides a few quick starts : http://clusterlabs.github.io/PAF/documentation.html

We'll here focus on the Pacemaker administration part and assume the PostgreSQL Streaming Replication is working and correctly configured.

The resource agent requires the PostgreSQL instances to be already set up, ready to start and slaves ready to replicate. 

Make sure to setup your PostgreSQL master on your preferred node to host the master: during the very first startup of the cluster, PAF detects the master based on its shutdown status.

Moreover, it requires a `recovery.conf` template ready to use. 

You can create a `recovery.conf` file suitable to your needs, the only requirements are:

- have standby_mode = on
- have recovery_target_timeline = 'latest'
- a primary_conninfo with an application_name set to the node name

In case you configure a virtual IP (called pgsql-vip in this post) on the server hosting the master PostgreSQL instance, make sure each instance will not be able to replicate with itself (in the `pg_hba.conf`)! 

-----

## [](#initial-steps)Initial steps

For this article, I created 2 VMs (CentOS 7), using qemu-kvm through the virt-manager user interface.

```
# virsh list --all
 Id    Name                           State
----------------------------------------------------
 1     server1                        running
 2     server2                        running
```

I also installed (from the PGDG repository) and set up a PostgreSQL 10 cluster with Streaming Replication between those 2 servers.

The `recovery.conf.pcmk` template contains:

```
$ cat <<EOF > ~postgres/recovery.conf.pcmk
standby_mode = on
primary_conninfo = 'host=pgsql-vip application_name=$(hostname -s)'
recovery_target_timeline = 'latest'
EOF
```

-----

## [](#cluster-preparation)Pacemaker cluster preparation

Pacemaker installation:

```bash
# yum install -y pacemaker resource-agents pcs fence-agents-all fence-agents-virsh
```

We'll later create one fencing resource per node to fence. They are called fence_vm_xxx and use the fencing agent fence_virsh, allowing to power on or off a virtual machine using the virsh command through a ssh connexion to the hypervisor. You'll need to make sure your VMs are able to connect as root (it is possible to use a normal user with some more setup though) to your hypervisor.

Install the latest PAF version, directly from the PGDG repository:

```bash
# yum install -y resource-agents-paf
```

It is advised to keep Pacemaker off on server boot. It helps the administrator to investigate after a node fencing before Pacemaker starts and potentially enters in a death match with the other nodes. Make sure to disable Corosync as well to avoid unexpected behaviors. 

Run this on all nodes:

```bash
# systemctl disable corosync
# systemctl disable pacemaker
```

Let's use the cluster management tool `pcsd`, provided by RHEL, to ease the creation and setup of a cluster. 

It allows to create the cluster from command line, without editing configuration files or XML by hands.

`pcsd` uses the `hacluster` system user to work and communicate with other members of the cluster.

```bash
# passwd hacluster
# systemctl enable pcsd
# systemctl start pcsd
```

Now, authenticate each node to the other ones using the following command:

```
# pcs cluster auth server1 server2 -u hacluster
Password: 
server1: Authorized
server2: Authorized
```

Create and start the cluster: 

```
# pcs cluster setup --name cluster_pgsql server1 server2
Destroying cluster on nodes: server1, server2...
server1: Stopping Cluster (pacemaker)...
server2: Stopping Cluster (pacemaker)...
server1: Successfully destroyed cluster
server2: Successfully destroyed cluster

Sending 'pacemaker_remote authkey' to 'server1', 'server2'
server1: successful distribution of the file 'pacemaker_remote authkey'
server2: successful distribution of the file 'pacemaker_remote authkey'
Sending cluster config files to the nodes...
server1: Succeeded
server2: Succeeded

Synchronizing pcsd certificates on nodes server1, server2...
server1: Success
server2: Success
Restarting pcsd on the nodes in order to reload the certificates...
server1: Success
server2: Success

# pcs cluster start --all
server2: Starting Cluster...
server1: Starting Cluster...
```

Check the cluster status:

```
# pcs status
Cluster name: cluster_pgsql
WARNING: no stonith devices and stonith-enabled is not false
Stack: corosync
Current DC: server2 (version 1.1.16-12.el7_4.5-94ff4df) - partition with quorum
Last updated: ...
Last change: ... by hacluster via crmd on server2

2 nodes configured
0 resources configured

Online: [ server1 server2 ]

No resources

Daemon Status:
  corosync: active/disabled
  pacemaker: active/disabled
  pcsd: active/enabled
```

Now the cluster run, let’s start with some basic setup of the cluster. 

Run the following command from one node only (the cluster takes care of broadcasting the configuration on all nodes):

```bash
# pcs resource defaults migration-threshold=5
# pcs resource defaults resource-stickiness=10
```

This sets two default values for resources we create in the next chapter:

- migration-threshold : this controls how many time the cluster tries to recover a resource on the same node before moving it on another one.
- resource-stickiness : adds a sticky score for the resource on its current node. It helps avoiding a resource move back and forth between nodes where it has the same score.

-----

## [](#node-fencing)Node fencing configuration

We can now create one STONITH resource for each node. Each fencing resource will not be allowed to run on the node it is supposed to fence. 

Note that in the port argument of the following commands, server[1-2] are the names of the virtual machines as known by libvirtd side and 192.168.122.1 is the ip of the qemu-kvm hypervisor.

```bash
# pcs cluster cib cluster1.xml

# pcs -f cluster1.xml stonith create fence_vm_server1 fence_virsh \
    pcmk_host_check="static-list" pcmk_host_list="server1"        \
    ipaddr="192.168.122.1" login="root" port="server1"            \
    action="off" identity_file="/root/.ssh/id_rsa"

# pcs -f cluster1.xml stonith create fence_vm_server2 fence_virsh \
    pcmk_host_check="static-list" pcmk_host_list="server2"        \
    ipaddr="192.168.122.1" login="root" port="server2"            \
    action="off" identity_file="/root/.ssh/id_rsa"

# pcs -f cluster1.xml constraint location fence_vm_server1 avoids server1=INFINITY
# pcs -f cluster1.xml constraint location fence_vm_server2 avoids server2=INFINITY
# pcs cluster cib-push cluster1.xml
```

Check the cluster status:

```
# pcs status
Cluster name: cluster_pgsql
Stack: corosync
Current DC: server1 (version 1.1.16-12.el7_4.5-94ff4df) - partition with quorum
Last updated: ...
Last change: ... by root via cibadmin on server1

2 nodes configured
2 resources configured

Online: [ server1 server2 ]

Full list of resources:

 fence_vm_server1	(stonith:fence_virsh):	Started server2
 fence_vm_server2	(stonith:fence_virsh):	Started server1

Daemon Status:
  corosync: active/disabled
  pacemaker: active/disabled
  pcsd: active/enabled
```

-----

## [](#cluster-resources)Cluster resources creation

Now the fencing is working, we can add all other resources and constraints all together in the same time. 

Create a new offline CIB:

```
# pcs cluster cib cluster1.xml
```

We'll create three resources : `pgsqld`, `pgsql-ha`, and `pgsql-master-ip`.

The `pgsqld` defines the properties of a PostgreSQL instance: where it is located, where are its binaries, its configuration files, how to monitor it, and so on.

```bash
# pcs -f cluster1.xml resource create pgsqld ocf:heartbeat:pgsqlms \
    bindir=/usr/pgsql-10/bin pgdata=/var/lib/pgsql/10/data         \
    recovery_template=/var/lib/pgsql/recovery.conf.pcmk            \
    op start timeout=60s                                           \
    op stop timeout=60s                                            \
    op promote timeout=30s                                         \
    op demote timeout=120s                                         \
    op monitor interval=15s timeout=10s role="Master"              \
    op monitor interval=16s timeout=10s role="Slave"               \
    op notify timeout=60s
```

The `pgsql-ha` resource controls all the `pgsqld` PostgreSQL instances in your cluster, decides where the primary is promoted and where the standby is started.

```bash
# pcs -f cluster1.xml resource master pgsql-ha pgsqld notify=true
```

The `pgsql-master-ip` resource controls the pgsql-vip (192.168.122.50) IP address. It is started on the node hosting the PostgreSQL master resource.

```bash
# pcs -f cluster1.xml resource create pgsql-master-ip ocf:heartbeat:IPaddr2 \
    ip=192.168.122.50 cidr_netmask=24 op monitor interval=10s
```

We now define the collocation between `pgsql-ha` and `pgsql-master-ip`. The start/stop and promote/demote order for these resources must be asymmetrical: we MUST keep the master IP on the master during its demote process so the standby receive everything during the master shutdown.

```bash
# pcs -f cluster1.xml constraint colocation add pgsql-master-ip with master pgsql-ha INFINITY
# pcs -f cluster1.xml constraint order promote pgsql-ha then start pgsql-master-ip symmetrical=false kind=Mandatory
# pcs -f cluster1.xml constraint order demote pgsql-ha then stop pgsql-master-ip symmetrical=false kind=Mandatory
```

We can now push our Cluster Information Base (aka. CIB) to the cluster, which will start all the magic stuff:

```bash
# pcs cluster cib-push cluster1.xml
```

Check the cluster status:

```bash
# pcs status
Cluster name: cluster_pgsql
Stack: corosync
Current DC: server1 (version 1.1.16-12.el7_4.5-94ff4df) - partition with quorum
Last updated: ...
Last change: ... by root via crm_attribute on server1

2 nodes configured
5 resources configured

Online: [ server1 server2 ]

Full list of resources:

 fence_vm_server1	(stonith:fence_virsh):	Started server2
 fence_vm_server2	(stonith:fence_virsh):	Started server1
 pgsql-master-ip	(ocf::heartbeat:IPaddr2):	Started server1
 Master/Slave Set: pgsql-ha [pgsqld]
     Masters: [ server1 ]
     Slaves: [ server2 ]

Daemon Status:
  corosync: active/disabled
  pacemaker: active/disabled
  pcsd: active/enabled
```

And finally, try to connect:

```
$ psql -h pgsql-vip
psql (10.1)
Type "help" for help.

postgres=# SELECT * FROM pg_stat_replication;
-[ RECORD 1 ]----+----------------------------
pid              | 1830
usesysid         | 10
usename          | postgres
application_name | server2
client_addr      | 192.168.122.77
client_hostname  | server2
client_port      | 42380
backend_start    | ...
backend_xmin     | 555
state            | streaming
sent_lsn         | 0/50002C0
write_lsn        | 0/50002C0
flush_lsn        | 0/50002C0
replay_lsn       | 0/50002C0
write_lag        | 
flush_lag        | 
replay_lag       | 
sync_priority    | 0
sync_state       | async

postgres=# SELECT pg_is_in_recovery();
 pg_is_in_recovery 
-------------------
 f
(1 row)
```

-----

# [](#conclusion)Conclusion

The quick guides provided with the PAF project are quite clear and complete. 

However, if your PostgreSQL instance is managed by Pacemaker, you should proceed to administration tasks with care.

Pacemaker only uses `pg_ctl`, and as other tools behave differently, using them could lead to some unpredictable behavior, like an init script reporting that the instance is stopped when it is not.

We'll see in a future article how to correctly manage the cluster.