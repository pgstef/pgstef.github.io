---
layout: post
title: Patroni and pgBackRest combined
date: 2022-07-12 09:00:00 +0200
---

I see more and more questions about pgBackRest in a Patroni cluster on community channels.
So, following yesterday's post about Patroni on pure Raft, we'll see in this post an example about how to setup pgBackRest in such cases.

To prepare this post, I followed most of the instructions given by Federico Campoli at PGDAY RUSSIA 2021 about [Protecting your data with Patroni and pgbackrest](https://pgday.ru/presentation/298/60f04d9ae4105.pdf). The video recording might even be found [here](https://youtu.be/UNM6if3uIUk).

<!--MORE-->

-----

# pgBackRest repository location

The first decision you'll have to take is to know where to store your pgBackRest repository. It can be any supported [repo-type](https://pgbackrest.org/configuration.html#section-repository/option-repo-type) with or without a dedicated backup server (aka. [repo-host](https://pgbackrest.org/user-guide-rhel.html#repo-host)). The most important thing to remember is that the repository should be equally reachable from all your PostgreSQL/Patroni nodes.

Based on that decision, you'll have to think about how to schedule the backup command. With a repo-host that's easy, because pgBackRest will be able to determine which PostgreSQL node is the primary (or a standby if you want to take the [backup from a standby node](https://pgbackrest.org/user-guide-rhel.html#standby-backup)). If you're using a directly attached storage (i.e. NFS mount point or S3 bucket), the easiest solution might be to test the return of `pg_is_in_recovery()` in the [cron](https://pgbackrest.org/user-guide-rhel.html#quickstart/schedule-backup) job.

For the purpose of this demo setup, I'll use a MinIO docker container with self-signed certificates and a bucket named `pgbackrest` already created.

![](../../../images/pgbackrest-patroni-minio.png){:width="800px"}


-----

# Installation

## pgBackRest installation and configuration

Install pgBackRest on all the PostgreSQL nodes:

```bash
$ sudo dnf install -y epel-release
$ sudo dnf install -y pgbackrest
$ pgbackrest version
pgBackRest 2.39
```

Then, configure `/etc/pgbackrest.conf`:

```bash
$ cat <<EOF | sudo tee /etc/pgbackrest.conf
[global]
repo1-type=s3
repo1-storage-verify-tls=n
repo1-s3-endpoint=192.168.121.1
repo1-s3-uri-style=path
repo1-s3-bucket=pgbackrest
repo1-s3-key=minioadmin
repo1-s3-key-secret=minioadmin
repo1-s3-region=eu-west-3

repo1-path=/repo1
repo1-retention-full=2
start-fast=y
log-level-console=info
log-level-file=debug
delta=y
process-max=2

[demo-cluster-1]
pg1-path=/var/lib/pgsql/14/data
pg1-port=5432
pg1-user=postgres
EOF
```

In the previous post, we defined the cluster name to use in the `scope` configuration of Patroni. We'll reuse the same name for the stanza name.

Let's initialize the stanza:

```bash
$ sudo -iu postgres pgbackrest --stanza=demo-cluster-1 stanza-create
INFO: stanza-create command begin 2.39: ...
INFO: stanza-create for stanza 'demo-cluster-1' on repo1
INFO: stanza-create command end: completed successfully (684ms)
```

## Configure Patroni to use pgBackRest

Let's adjust the `archive_command` in Patroni configuration:

```bash
$ sudo -iu postgres patronictl -c /etc/patroni.yml edit-config
## adjust the following lines
postgresql:
  parameters:
    archive_command: pgbackrest --stanza=demo-cluster-1 archive-push "%p"
    archive_mode: "on"

$ sudo -iu postgres patronictl -c /etc/patroni.yml reload demo-cluster-1
```

Check that the archiving system is working:

```bash
$ sudo -iu postgres pgbackrest --stanza=demo-cluster-1 check
INFO: check command begin 2.39: ...
INFO: check repo1 configuration (primary)
INFO: check repo1 archive for WAL (primary)
INFO: WAL segment 000000010000000000000004 successfully archived to '...' on repo1
INFO: check command end: completed successfully (1083ms)
```

## Take a first backup

Let's not take our first full backup:

```bash
$ sudo -iu postgres pgbackrest --stanza=demo-cluster-1 backup --type=full
P00   INFO: backup command begin 2.39: ...
P00   INFO: execute non-exclusive pg_start_backup():
  backup begins after the requested immediate checkpoint completes
P00   INFO: backup start archive = 000000010000000000000006, lsn = 0/6000028
P00   INFO: check archive for prior segment 000000010000000000000005
P00   INFO: execute non-exclusive pg_stop_backup() and wait for all WAL segments to archive
P00   INFO: backup stop archive = 000000010000000000000006, lsn = 0/6000100
P00   INFO: check archive for segment(s) 000000010000000000000006:000000010000000000000006
P00   INFO: new backup label = 20220711-075256F
P00   INFO: full backup size = 25.2MB, file total = 957
P00   INFO: backup command end: completed successfully
```

All Patroni nodes should now be able to see the content of the pgBackRest repository:

```bash
$ pgbackrest info
stanza: demo-cluster-1
    status: ok
    cipher: none

    db (current)
        wal archive min/max (14): 000000010000000000000006/000000010000000000000006

        full backup: 20220711-075256F
            timestamp start/stop: 2022-07-11 07:52:56 / 2022-07-11 07:53:02
            wal start/stop: 000000010000000000000006 / 000000010000000000000006
            database size: 25.2MB, database backup size: 25.2MB
            repo1: backup set size: 3.2MB, backup size: 3.2MB
```

-----

# The "restore" part

Now that the archiving system is working, we can configure the `restore_command`. Possibly, we could disable replication slots for Patroni too since we now have the archives as safety net for the replication.

Let's edit the bootstrap configuration part:

```bash
$ sudo -iu postgres patronictl -c /etc/patroni.yml edit-config
## adjust the following lines
loop_wait: 10
maximum_lag_on_failover: 1048576
postgresql:
  parameters:
    archive_command: pgbackrest --stanza=demo-cluster-1 archive-push "%p"
    archive_mode: 'on'
  recovery_conf:
    recovery_target_timeline: latest
    restore_command: pgbackrest --stanza=demo-cluster-1 archive-get %f "%p"
  use_pg_rewind: false
  use_slots: true
retry_timeout: 10
ttl: 30

$ sudo -iu postgres patronictl -c /etc/patroni.yml reload demo-cluster-1
```

To use pgBackRest for creating (or re-initializing) replicas, we need to adjust the Patroni configuration file.

On all your nodes, in `/etc/patroni.yml`, find the following part:

```yaml
postgresql:
  listen: "0.0.0.0:5432"
  connect_address: "$MY_IP:5432"
  data_dir: /var/lib/pgsql/14/data
  bin_dir: /usr/pgsql-14/bin
  pgpass: /tmp/pgpass0
  authentication:
    replication:
      username: replicator
      password: confidential
    superuser:
      username: postgres
      password: my-super-password
    rewind:
      username: rewind_user
      password: rewind_password
  parameters:
    unix_socket_directories: '/var/run/postgresql,/tmp'
```

and add:

```yaml
  create_replica_methods:
    - pgbackrest
    - basebackup
  pgbackrest:
    command: pgbackrest --stanza=demo-cluster-1 restore --type=none
    keep_data: True
    no_params: True
  basebackup:
    checkpoint: 'fast'
```

Don't forget to reload the configuration:

```bash
$ sudo systemctl reload patroni
```

## Create a replica using pgBackRest

Here is our current situation:

```bash
$ sudo -iu postgres patronictl -c /etc/patroni.yml list
+--------+-----------------+---------+---------+----+-----------+
| Member | Host            | Role    | State   | TL | Lag in MB |
+ Cluster: demo-cluster-1 (7119014497759128647) ----+-----------+
| srv1   | 192.168.121.2   | Replica | running |  1 |         0 |
| srv2   | 192.168.121.254 | Leader  | running |  1 |           |
| srv3   | 192.168.121.89  | Replica | running |  1 |         0 |
+--------+-----------------+---------+---------+----+-----------+
```

We already have 2 running replicas. So we'll need to stop Patroni on one node and remove its data directory to trigger a new replica creation:

```bash
$ sudo systemctl stop patroni
$ sudo rm -rf /var/lib/pgsql/14/data
$ sudo systemctl start patroni
$ sudo journalctl -u patroni.service -f
...
INFO: trying to bootstrap from leader 'srv2'
...
INFO: replica has been created using pgbackrest
INFO: bootstrapped from leader 'srv2'
...
INFO: no action. I am (srv3), a secondary, and following a leader (srv2)
```

As we can see from the logs above, the replica has successfully been created using pgBackRest:

```bash
$ sudo -iu postgres patronictl -c /etc/patroni.yml list
+--------+-----------------+---------+---------+----+-----------+
| Member | Host            | Role    | State   | TL | Lag in MB |
+ Cluster: demo-cluster-1 (7119014497759128647) ----+-----------+
| srv1   | 192.168.121.2   | Replica | running |  1 |         0 |
| srv2   | 192.168.121.254 | Leader  | running |  1 |           |
| srv3   | 192.168.121.89  | Replica | running |  1 |         0 |
+--------+-----------------+---------+---------+----+-----------+
```

Now, let's insert some data on srv2 and simulate an incident by suspending the VM network interface for a few seconds.

A failover will happen and the old primary will be completely out-of-sync and the replication will be lagging when adding new data on the primary:

```bash
$ sudo -iu postgres patronictl -c /etc/patroni.yml list
+--------+-----------------+---------+---------+----+-----------+
| Member | Host            | Role    | State   | TL | Lag in MB |
+ Cluster: demo-cluster-1 (7119014497759128647) ----+-----------+
| srv1   | 192.168.121.2   | Leader  | running |  2 |           |
| srv2   | 192.168.121.254 | Replica | running |  1 |       188 |
| srv3   | 192.168.121.89  | Replica | running |  2 |         0 |
+--------+-----------------+---------+---------+----+-----------+
```

Since we didn't configure `pg_rewind`, we'll need to re-initialize the failing node manually:

```bash
$ sudo -iu postgres patronictl -c /etc/patroni.yml reinit demo-cluster-1 srv2
+--------+-----------------+---------+---------+----+-----------+
| Member | Host            | Role    | State   | TL | Lag in MB |
+ Cluster: demo-cluster-1 (7119014497759128647) ----+-----------+
| srv1   | 192.168.121.2   | Leader  | running |  2 |           |
| srv2   | 192.168.121.254 | Replica | running |  1 |       188 |
| srv3   | 192.168.121.89  | Replica | running |  2 |         0 |
+--------+-----------------+---------+---------+----+-----------+
Are you sure you want to reinitialize members srv2? [y/N]: y
Success: reinitialize for member srv2

$ sudo -iu postgres patronictl -c /etc/patroni.yml list
+--------+-----------------+---------+---------+----+-----------+
| Member | Host            | Role    | State   | TL | Lag in MB |
+ Cluster: demo-cluster-1 (7119014497759128647) ----+-----------+
| srv1   | 192.168.121.2   | Leader  | running |  2 |           |
| srv2   | 192.168.121.254 | Replica | running |  2 |         0 |
| srv3   | 192.168.121.89  | Replica | running |  2 |         0 |
+--------+-----------------+---------+---------+----+-----------+
```

The following trace shows that pgBackRest has successfully been used to re-initialize the old primary:

```bash
INFO: replica has been created using pgbackrest
INFO: bootstrapped from leader 'srv1'
...
INFO: Lock owner: srv1; I am srv2
INFO: establishing a new patroni connection to the postgres cluster
INFO: no action. I am (srv2), a secondary, and following a leader (srv1)
```

-----

# Conclusion

It is very easy and convenient to configure Patroni to use pgBackRest. Obviously, having backups is a good thing but being able to use those backups as source for the replica creation or re-initialization is even better.

As usual, the hardest part is to clearly define where to store the pgBackRest repository.
