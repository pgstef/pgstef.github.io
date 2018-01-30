---
layout: post
title: PAF cluster management
draft: true
---

“PostgreSQL Automatic Failover” (aka. PAF : http://clusterlabs.github.io/PAF/) is a resource agent providing service High Availability for PostgreSQL, based on Pacemaker and Corosync.

In the previous post, we saw how to quickly install it. Let's see now how to manage it.

<!--MORE-->

-----

# [](#cluster-management)Cluster management

The documentation provides cookbooks : http://clusterlabs.github.io/PAF/CentOS-7-admin-cookbook.html

Let's see how to perform some important actions.

-----

## [](#starting-or-stopping-the-cluster)Starting or stopping the cluster

It's possible to show the resources handled by the cluster :

```
# pcs resource show
 pgsql-master-ip	(ocf::heartbeat:IPaddr2):	Started server1
 Master/Slave Set: pgsql-ha [pgsqld]
     Masters: [ server1 ]
     Slaves: [ server2 ]
```

To completely stop the cluster (and its resources), first disable the `pgsql-ha` resource :

```bash
# pcs resource disable pgsql-ha
```

Then, stop the cluster on all nodes (one by one or at once) :

```bash
# pcs cluster stop --all
```

It's best to disable the resource before stopping the cluster to avoid some unexpected behavior.

To start the cluster :

```bash
# pcs cluster start --all
# pcs resource enable pgsql-ha
```

Prevent the resource id specified from running on the node (or on the current node it is running on if no node is specified) by creating a -INFINITY location constraint. 

To stop the cluster on a specific node, you can avoid the `pgsql-ha` resource to be running on a specific node with:

```bash
# pcs resource ban --wait pgsql-ha server2
```

You can then stop the cluster safely. To bring the resource back online, allows it:

```bash
# pcs resource clear pgsql-ha server2
```

-----

## [](#swapping-master-and-slave-roles-between-nodes)Swapping master and slave roles between nodes

You can easily perform a switch-over with :

```
# pcs resource move --wait --master pgsql-ha server2
Resource 'pgsql-ha' is master on node server2; slave on node server1.
# pcs resource clear pgsql-ha
```

To move the resource, pcs sets an INFINITY constraint location for the master on the given node. You must clear this constraint to avoid unexpected location behavior using the `pcs resource clear` command.

-----


## [](#maintenance-mode)Maintenance mode

To perform some maintenance task without stopping the resources, you just need to make sure the cluster manager do not decide to run any action.

The easiest way to do so is to put the whole cluster in maintenance mode : 

```bash
# pcs property set maintenance-mode=true
# pcs property set maintenance-mode=false
```

Once out of maintenance mode, the cluster manager will expect to find the resources in the same state they were before the maintenance.

-----

# [](#conclusion)Conclusion

If you wish to build service High Availability while keeping hands on your PostgreSQL configuration, PAF is definitively made for you !