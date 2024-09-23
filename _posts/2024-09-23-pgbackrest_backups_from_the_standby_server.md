---
layout: post
title: pgBackRest backups from the standby server
date: 2024-09-23 10:15:00 +0200
---

Recently, we've received many questions about how to take backups from a standby server using pgBackRest. In this post, I'd like to clarify one of the most frequently asked questions and address a common misconception for new users.

First of all, it's important to understand that taking a backup exclusively from the standby server is not currently possible. When you trigger a backup from the standby, pgBackRest creates a standby backup that is identical to a backup performed on the primary. It does this by starting/stopping the backup on the primary, copying only files that are replicated from the standby, then copying the remaining few files from the primary.

For this setup to work, both the primary and standby servers must share a common backup repository. This can be any supported [repository type](https://pgbackrest.org/configuration.html#section-repository/option-repo-type).

Let's take an example, using an NFS mount point.

<!--MORE-->

-----

# Example setup for backups from the standby server

## Initial situation

Both the primary (`pg1`) and the standby (`pg2`) are seeing the same content of the mentioned NFS mount:

```bash
[postgres@pg1 ~]$ ls /shared/
[postgres@pg1 ~]$ touch /shared/test_write_from the primary
[postgres@pg1 ~]$ mkdir /shared/pgbackrest
```
```bash
[postgres@pg2 ~]$ ls /shared
pgbackrest  test_write_from
```

And we've got a working replication connection:

```bash
postgres=# SELECT * FROM pg_stat_replication;
-[ RECORD 1 ]----+------------------------------
pid              | 27773
usename          | replicator
application_name | pg2
state            | streaming
sent_lsn         | 0/500AD6C8
write_lsn        | 0/500AD6C8
flush_lsn        | 0/500AD6C8
replay_lsn       | 0/500AD6C8
sync_state       | async
```

## WAL archiving

Let's configure	pgBackRest on `pg1` and `pg2`:

```bash
$ pgbackrest version
pgBackRest 2.53.1

$ cat<<EOF | sudo tee "/etc/pgbackrest.conf"
[global]
repo1-path=/shared/pgbackrest
repo1-retention-full=4
repo1-bundle=y
repo1-block=y
start-fast=y
log-level-console=info
log-level-file=detail
delta=y
process-max=2
compress-type=zst

[mycluster]
pg1-path=/var/lib/pgsql/16/data
EOF
```

Enable archiving on the primary (`pg1`):

```bash
$ cat<<EOF | sudo tee -a "/var/lib/pgsql/16/data/postgresql.auto.conf"
archive_mode = on
archive_command = 'pgbackrest --stanza=mycluster archive-push %p'
EOF

$ sudo systemctl restart postgresql-16
```

Initialize the backup repository content running the `stanza-create` command on the primary (`pg1`):

```bash
$ sudo -iu postgres pgbackrest --stanza=mycluster stanza-create
INFO: stanza-create command begin 2.53.1: ...
INFO: stanza-create for stanza 'mycluster' on repo1
INFO: stanza-create command end: completed successfully
```

Since both the primary and the standby are sharing a common backup repository, you just need to run the above command only once!

Now, let's check that the archiving system works correctly by running the `check` command:

```bash
$ sudo -iu postgres pgbackrest --stanza=mycluster check
INFO: check command begin 2.53.1: ...
INFO: check repo1 configuration (primary)
INFO: check repo1 archive for WAL (primary)
INFO: WAL segment 000000010000000000000050 successfully archived to '...' on repo1
INFO: check command end: completed successfully
```

This will trigger a new archive on the primary and then check if pgBackRest, running from the server where the command is executed, can locate that archived file in the backup repository. It's helpful to run this `check` command on both the primary and the standby servers to confirm that the archiving system is functioning correctly and that both servers can access the same backup repository. But we'll do that later on the standby since we still need to adjust the pgBackRest configuration.

When using _Streaming Replication_, it's generally recommended to set up replication slots. This ensures that the primary doesn't remove WAL files needed by the standby, or rows that could cause a recovery conflict, even when the standby is disconnected. Since the primary archives those WAL segments before recycling them, and both nodes share a common backup repository, you can rely on those archives to help the standby (`pg2`) catch up. To do this, set the `restore_command` on the standby:

```bash
$ cat<<EOF | sudo tee -a "/var/lib/pgsql/16/data/postgresql.auto.conf"
restore_command = 'pgbackrest --stanza=mycluster archive-get %f "%p"'
EOF

$ sudo systemctl reload postgresql-16
```

To verify that replication and WAL archiving are working, you can rely on `pg_stat_replication` and `pg_stat_archiver`:

```sql
postgres=# SELECT * FROM pg_stat_replication;
-[ RECORD 1 ]----+------------------------------
pid              | 28865
usename          | replicator
application_name | pg2
state            | streaming
sent_lsn         | 0/520000D8
write_lsn        | 0/520000D8
flush_lsn        | 0/520000D8
replay_lsn       | 0/520000D8
sync_state       | async
...

postgres=# SELECT
            pg_walfile_name(pg_current_wal_lsn()),
            last_archived_wal FROM pg_stat_archiver;
-[ RECORD 1 ]------+------------------------------
pg_walfile_name    | 000000010000000000000052
last_archived_wal  | 000000010000000000000051
```

You can also use the pgBackRest `info` command to look at what's inside the repository:

```bash
$ sudo -iu postgres pgbackrest info --stanza=mycluster
...
db (current)
    wal archive min/max (16): 000000010000000000000050/000000010000000000000051
```

## SSH access

Now that replication and WAL archiving are working, and we've successfully verified that the standby server can access the shared backup repository, we can prepare the servers to allow backups to be taken from the standby. To achieve this, the pgBackRest process on the standby will need to trigger a local process on the primary server to establish a database connection via Unix sockets. This ensures that pgBackRest can interact with the primary for certain operations, even when the backup is initiated from the standby. For the communication between the servers, pgBackRest can be configured to use passwordless SSH.

The SSH setup is up to you, but usually it is as simple as creating SSH keys and authorize them on the other node. Example:

```bash
$ sudo -u postgres ssh-keygen -f /var/lib/pgsql/.ssh/id_rsa -t rsa -b 4096 -N ""
```

Then, copy the generated public keys to the other node:

```bash
# From pg1
[postgres@pg1 ~]$ ssh-copy-id postgres@pg2
[postgres@pg1 ~]$ ssh postgres@pg2 hostname

# From pg2
[postgres@pg2 ~]$ ssh-copy-id postgres@pg1
[postgres@pg2 ~]$ ssh postgres@pg1 hostname
```

Finally now, we'll need to adjust the pgBackRest configuration to add a connection to the primary (`pg2-path`, `pg2-host`, `pg2-host-user`) and specify that we want to take a backup from the standby server (`backup-standby=y`). The configuration on the standby should then look like this:

```ini
[global]
repo1-path=/shared/pgbackrest
repo1-retention-full=4
repo1-bundle=y
repo1-block=y
start-fast=y
log-level-console=info
log-level-file=detail
delta=y
process-max=2
compress-type=zst
backup-standby=y

[mycluster]
pg1-path=/var/lib/pgsql/16/data
pg2-path=/var/lib/pgsql/16/data
pg2-host=pg1
pg2-host-user=postgres
```

As I mentioned above, it is helpful to run the `check` command on the standby server once we've adjusted the configuration before taking a backup:

```bash
$ sudo -iu postgres pgbackrest --stanza=mycluster check
INFO: check command begin 2.53.1: ...
INFO: check repo1 (standby)
INFO: switch wal not performed because this is a standby
INFO: check repo1 configuration (primary)
INFO: check repo1 archive for WAL (primary)
INFO: WAL segment 000000010000000000000053 successfully archived to '...' on repo1
INFO: check command end: completed successfully
```

## Backups

Now, take a full backup:

```bash
$ sudo -iu postgres pgbackrest backup --stanza=mycluster --type=full
INFO: backup command begin 2.53.1: ...
INFO: execute non-exclusive backup start:
        backup begins after the requested immediate checkpoint completes
INFO: backup start archive = 000000010000000000000055, lsn = 0/55000028
INFO: wait for replay on the standby to reach 0/55000028
INFO: replay on the standby reached 0/55000028
INFO: check archive for prior segment 000000010000000000000054
INFO: execute non-exclusive backup stop and wait for all WAL segments to archive
INFO: backup stop archive = 000000010000000000000055, lsn = 0/55000138
INFO: check archive for segment(s) 000000010000000000000055:000000010000000000000055
INFO: new backup label = 20240920-144928F
2INFO: full backup size = 1.5GB, file total = 1280
INFO: backup command end: completed successfully
```

* `wait for replay on the standby to reach 0/55000028`: pgBackRest ensures that the standby server has fully caught up with the primary before allowing the backup to proceed. This prevents backing up a lagging standby.
* `check archive for prior segment 000000010000000000000054`: pgBackRest checks if the previous WAL archive has been successfully pushed to the repository by the primary within the `archive-timeout` period. If this check fails, it could indicate that the standby server lacks access to the shared repository, or that the archiving process on the primary is too slow.

If the archiving process on the primary is too slow, you might consider enabling [asynchronous archiving](https://pgbackrest.org/user-guide.html#async-archiving). The pgBackRest user guide highlights several [performance tuning](https://pgbackrest.org/user-guide.html#quickstart/performance-tuning) settings that can help resolve this issue: `archive-async`, `process-max`, and `compress-type=zst` are the recommended options to improve archiving speed.

Finally, look at the `info` command to know what's inside the repository:

```bash
$ sudo -iu postgres pgbackrest info --stanza=mycluster
stanza: mycluster
    status: ok
    cipher: none

    db (current)
        wal archive min/max (16): 000000010000000000000050/000000010000000000000055

        full backup: 20240920-144928F
            timestamp start/stop: 2024-09-20 14:49:28+00 / 2024-09-20 14:49:32+00
            wal start/stop: 000000010000000000000055 / 000000010000000000000055
            database size: 1.5GB, database backup size: 1.5GB
            repo1: backup size: 67.9MB
```

---

# Conclusion

Setting up backups from a standby server is an effective way to reduce the load on the primary server. While pgBackRest still relies on the primary server for certain tasks, using the standby for most of the backup process can greatly improve performance in a replicated environment. However, while the technical setup is relatively straightforward, the more challenging part of this architecture lies in deciding where, when, and how to trigger the backups.

In most cases, you would configure pgBackRest identically on both nodes and use a cron job to [schedule the backups](https://pgbackrest.org/user-guide.html#quickstart/schedule-backup). To avoid complications (and manual operations) when switching roles between the primary and standby, the easiest solution is to check the result of `pg_is_in_recovery()` in the cron job. This way, backups are automatically triggered on the correct node, regardless of which server is currently acting as the standby.

However, this approach is most efficient when you only have a single standby server. As your cluster grows, with multiple standbys and possibly automated failover (using tools like Patroni), it becomes more practical to set up a [dedicated repository host](https://pgbackrest.org/user-guide.html#repo-host) (also known as a dedicated backup server). This backup server can then trigger the backups and automatically select the appropriate node.
