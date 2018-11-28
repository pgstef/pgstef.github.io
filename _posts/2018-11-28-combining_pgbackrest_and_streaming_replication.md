---
layout: post
title: Combining pgBackRest and Streaming Replication
---

[pgBackRest](http://pgbackrest.org/) is a well-known powerful backup and 
restore tool. It offers a lot of possibilities.

While pg_basebackup is commonly used to setup the initial database copy for 
the Streaming Replication, it could be interesting to reuse a previous 
database backup (eg. taken with pgBackRest) to perform this initial copy.

Furthermore, the `--delta` option provided by pgBackRest can help us to 
re-synchronize an old secondary server without having to rebuild it from 
scratch.

To reduce the load on the primary server during a backup, pgBackRest even 
allows to take backups from a standby server.

We'll see in this blog post how to do that.

<!--MORE-->

-----

For the purpose of this post, we'll use 2 nodes called primary and secondary. Both 
are running on CentOS 7.

We'll cover some pgBackRest tips but won't go deeper in the PostgreSQL 
configuration, nor in the Streaming Replication best practices.

-----

# [](#installation)Installation

On both primary and secondary server, install PostgreSQL and pgBackRest 
packages directly from the PGDG yum repositories:

```bash
$ sudo yum install -y https://download.postgresql.org/pub/repos/yum/11/redhat/\
rhel-7-x86_64/pgdg-centos11-11-2.noarch.rpm
$ sudo yum install -y postgresql11-server postgresql11-contrib pgbackrest
```

Check that pgBackRest is correctly installed:

```bash
$ pgbackrest
pgBackRest 2.07 - General help

Usage:
    pgbackrest [options] [command]

Commands:
    archive-get     Get a WAL segment from the archive.
    archive-push    Push a WAL segment to the archive.
    backup          Backup a database cluster.
    check           Check the configuration.
    expire          Expire backups that exceed retention.
    help            Get help.
    info            Retrieve information about backups.
    restore         Restore a database cluster.
    stanza-create   Create the required stanza data.
    stanza-delete   Delete a stanza.
    stanza-upgrade  Upgrade a stanza.
    start           Allow pgBackRest processes to run.
    stop            Stop pgBackRest processes from running.
    version         Get version.

Use 'pgbackrest help [command]' for more information.
```

Create a basic PostgreSQL cluster on primary:

```bash
$ sudo /usr/pgsql-11/bin/postgresql-11-setup initdb
$ sudo systemctl enable postgresql-11
$ sudo systemctl start postgresql-11
```

-----

## [](#nfs-share)Setup a shared repository between the hosts

To be able to share the backups between the hosts, we'll here create a nfs 
export from secondary and mount it on primary.

Install and activate nfs server on secondary:

```bash
$ sudo yum -y install nfs-utils
$ sudo systemctl enable nfs-server.service 
$ sudo systemctl start nfs-server.service 
$ sudo firewall-cmd --permanent --add-service=nfs
$ sudo firewall-cmd --reload
```

Create the backup repository and export it:

```bash
$ sudo mkdir /mnt/backups
$ sudo chown postgres: /mnt/backups/
$ sudo echo "/mnt/backups primary(rw,sync,no_root_squash)" >> /etc/exports
$ sudo exportfs -a
```

Install nfs client and mount the shared repository on primary:

```bash
$ sudo yum -y install nfs-utils
$ sudo echo "secondary:/mnt/backups /mnt/backups nfs rw,sync,hard,intr 0 0" >> /etc/fstab
$ sudo mount /mnt/backups/
```

The storage of your backups is completely up to you. The requirement here is 
to have that storage available on both servers. 

If needed, you might even encrypt your repository. To do that, follow the 
[documentation](https://pgbackrest.org/user-guide.html#quickstart/configure-encryption).

-----

## Configure pgBackRest to backup the local cluster

By default, the configuration file is `/etc/pgbackrest.conf`. Let's make a copy:

```bash
$ sudo cp /etc/pgbackrest.conf /etc/pgbackrest.conf.bck
```

Update the primary configuration:

```ini
[global]
repo1-path=/mnt/backups
repo1-retention-full=1
process-max=2
log-level-console=info
log-level-file=debug

[mycluster]
pg1-path=/var/lib/pgsql/11/data
```

Configure archiving in the `postgresql.conf` file:

```
archive_mode = on
archive_command = 'pgbackrest --stanza=mycluster archive-push %p'
```

The PostgreSQL cluster must be restarted after making these changes and 
before performing a backup.

Let's finally create the stanza and check the configuration:

```bash
$ sudo -u postgres pgbackrest --stanza=mycluster stanza-create
P00   INFO: stanza-create command begin 2.07: 
			--log-level-console=info --log-level-file=debug  
			--pg1-path=/var/lib/pgsql/11/data --repo1-path=/mnt/backups 
			--stanza=mycluster
P00   INFO: stanza-create command end: completed successfully

$ sudo -u postgres pgbackrest --stanza=mycluster check
P00   INFO: check command begin 2.07: -
			-log-level-console=info --log-level-file=debug  
			--pg1-path=/var/lib/pgsql/11/data --repo1-path=/mnt/backups 
			--stanza=mycluster
P00   INFO: WAL segment 000000010000000000000001 successfully stored in the 
	archive at '/mnt/backups/archive/mycluster/11-1/0000000100000000/
	000000010000000000000001-ee7d07fc95b699231dac05d3b5c9f4b1dda22488.gz'
P00   INFO: check command end: completed successfully
```

-----

## Insert some test data in the database

Using pgbench, let's create some test data:

```bash
$ sudo -iu postgres createdb test
$ sudo -iu postgres /usr/pgsql-11/bin/pgbench -i -s 100 test
```

-----

# Prepare the servers for Streaming Replication

On primary server, add to `postgresql.conf`:

```
listen_addresses = '*'
```

Create a specific user for the replication:

```bash
$ sudo -iu postgres psql
postgres=# CREATE ROLE replic_user WITH LOGIN REPLICATION PASSWORD 'mypwd';
```

Configure `pg_hba.conf`:

```
host replication replic_user secondary md5
```

Restart the cluster and allow the service in the firewall (if needed):

```bash
$ sudo systemctl restart postgresql-11.service
$ sudo firewall-cmd --permanent --add-service=postgresql
$ sudo firewall-cmd --reload
```

Configure `~postgres/.pgpass` on secondary servers:

```bash
$ echo "*:*:replication:replic_user:mypwd" >> ~postgres/.pgpass
$ chown postgres: ~postgres/.pgpass
$ chmod 0600 ~postgres/.pgpass
```

-----

# [](#perform-backup)Perform a backup

Let's take our first backup on the primary server:

```bash
$ sudo -u postgres pgbackrest --stanza=mycluster --type=full backup
P00   INFO: backup command begin 2.07: 
		--log-level-console=info --log-level-file=debug 
		--pg1-path=/var/lib/pgsql/11/data --process-max=2 
		--repo1-path=/mnt/backups --repo1-retention-full=1 
		--stanza=mycluster --type=full
P00   INFO: execute non-exclusive pg_start_backup() with label "...": 
		backup begins after the next regular checkpoint completes
P00   INFO: backup start archive = 000000010000000000000057, lsn = 0/57000028
...
P00   INFO: full backup size = 1.4GB
P00   INFO: execute non-exclusive pg_stop_backup() and wait for all WAL segments 
        to archive
P00   INFO: backup stop archive = 000000010000000000000057, lsn = 0/57000168
P00   INFO: new backup label = 20181127-152908F
P00   INFO: backup command end: completed successfully
P00   INFO: expire command begin
P00   INFO: expire command end: completed successfully
```

-----

# [](#secondary-setup)Secondary setup

Configure `/etc/pgbackrest.conf`:

```ini
[global]
repo1-path=/mnt/backups
repo1-retention-full=1
process-max=2
log-level-console=info
log-level-file=debug

[mycluster]
pg1-path=/var/lib/pgsql/11/data
recovery-option=standby_mode=on
recovery-option=primary_conninfo=host=primary user=replic_user
recovery-option=recovery_target_timeline=latest
```

Usage of the `--delta` option:

> Restore or backup using checksums.
> 
> During a restore, by default the PostgreSQL data and tablespace directories 
> are expected to be present but empty. 
> This option performs a delta restore using checksums.
> 
> During a backup, this option will use checksums instead of the timestamps 
> to determine if files will be copied.

We use this option here to avoid having to clean the data directory of the 
secondary server, which can be very helpful in case of huge volumes. 
If you kept several full backups for example, it could be interesting to use 
this option for the backups too. Like the other parameters, it can also be set 
in the configuration files.

Restore the backup taken from the primary server:

```bash
$ sudo -u postgres pgbackrest --stanza=mycluster --delta restore
P00   INFO: restore command begin 2.07: 
		--delta --log-level-console=info --log-level-file=debug 
		--pg1-path=/var/lib/pgsql/11/data --process-max=2 
		--recovery-option=standby_mode=on 
		--recovery-option="primary_conninfo=host=primary user=replic_user" 
		--recovery-option=recovery_target_timeline=latest 
		--repo1-path=/mnt/backups --stanza=mycluster
P00   INFO: restore backup set 20181127-152908F
P00   INFO: remove invalid files/paths/links from /var/lib/pgsql/11/data
...
P00   INFO: write /var/lib/pgsql/11/data/recovery.conf
P00   INFO: restore global/pg_control 
		(performed last to ensure aborted restores cannot be started)
P00   INFO: restore command end: completed successfully
```

Actually, the `recovery-option` parameters allow pgBackRest to configure the
`recovery.conf` file:

```bash
$ cat /var/lib/pgsql/11/data/recovery.conf 
primary_conninfo = 'host=primary user=replic_user'
recovery_target_timeline = 'latest'
standby_mode = 'on'
restore_command = 'pgbackrest --stanza=mycluster archive-get %f "%p"'
```

All we have to do now is to start the PostgreSQL cluster:

```bash
$ sudo systemctl enable postgresql-11
$ sudo systemctl start postgresql-11
```

If the replication setup is correct, you should see those processes on the 
secondary server:

```
# ps -ef |grep postgres
postgres 19610     1  ... /usr/pgsql-11/bin/postmaster -D /var/lib/pgsql/11/data/
postgres 19614 19610  ... postgres: startup   recovering 000000010000000000000058
postgres 19621 19610  ... postgres: walreceiver   streaming 0/58000140
```

We now have a 2-nodes cluster working with Streaming Replication and archives 
recovery as safety net.

-----

# [](#backup-standby)Take backups from the secondary server

pgBackRest can perform backups on a standby server instead of the primary. 
Both the primary and secondary databases configuration are required, even if 
the majority of the files will be copied from the secondary to reduce load 
on the primary. 

To do so, adjust the `/etc/pgbackrest.conf` file on secondary:

```ini
[global]
repo1-path=/mnt/backups
repo1-retention-full=1
process-max=2
log-level-console=info
log-level-file=debug
backup-standby=y
delta=y

[mycluster]
pg1-host=primary
pg1-path=/var/lib/pgsql/11/data
pg2-path=/var/lib/pgsql/11/data
recovery-option=standby_mode=on
recovery-option=primary_conninfo=host=primary user=replic_user
recovery-option=recovery_target_timeline=latest
```

Options added are:

  * delta: to allow delta backup and restore without using `--delta`
  * backup-standby
  * pg1-host and pg1-path


Perform a backup from secondary:

```bash
$ sudo -u postgres pgbackrest --stanza=mycluster --type=full backup
P00   INFO: backup command begin 2.07: 
		--backup-standby --delta --log-level-console=info 
        --log-level-file=debug 
		--pg1-host=primary --pg1-path=/var/lib/pgsql/11/data 
		--pg2-path=/var/lib/pgsql/11/data --process-max=2 
		--repo1-path=/mnt/backups --repo1-retention-full=1 
		--stanza=mycluster --type=full
P00   INFO: execute non-exclusive pg_start_backup() with label "...": 
		backup begins after the next regular checkpoint completes
P00   INFO: backup start archive = 00000001000000000000005B, lsn = 0/5B000028
P00   INFO: wait for replay on the standby to reach 0/5B000028
P00   INFO: replay on the standby reached 0/5B0000D0, checkpoint 0/5B000060
...
P00   INFO: full backup size = 1.4GB
P00   INFO: execute non-exclusive pg_stop_backup() and wait for all WAL segments 
        to archive
P00   INFO: backup stop archive = 00000001000000000000005B, lsn = 0/5B000130
P00   INFO: new backup label = 20181127-164924F
P00   INFO: backup command end: completed successfully
P00   INFO: expire command begin
...
P00   INFO: expire command end: completed successfully
```

Even incremental backups can be taken:

```bash
$ sudo -u postgres pgbackrest --stanza=mycluster --type=incr backup
P00   INFO: backup command begin 2.07: 
		--backup-standby --delta --log-level-console=info 
        --log-level-file=debug 
		--pg1-host=primary --pg1-path=/var/lib/pgsql/11/data 
		--pg2-path=/var/lib/pgsql/11/data --process-max=2 
		--repo1-path=/mnt/backups --repo1-retention-full=1 
		--stanza=mycluster --type=incr
P00   INFO: last backup label = 20181127-164924F, version = 2.07
P00   INFO: execute non-exclusive pg_start_backup() with label "...": 
		backup begins after the next regular checkpoint completes
P00   INFO: backup start archive = 00000001000000000000005D, lsn = 0/5D000028
P00   INFO: wait for replay on the standby to reach 0/5D000028
P00   INFO: replay on the standby reached 0/5D0000D0, checkpoint 0/5D000060
...
P00   INFO: incr backup size = 1.4GB
P00   INFO: execute non-exclusive pg_stop_backup() and wait for all WAL segments 
        to archive
P00   INFO: backup stop archive = 00000001000000000000005D, lsn = 0/5D000130
P00   INFO: new backup label = 20181127-164924F_20181127-165743I
P00   INFO: backup command end: completed successfully
P00   INFO: expire command begin
P00   INFO: expire command end: completed successfully

$ sudo -u postgres pgbackrest --stanza=mycluster info
stanza: mycluster
    status: ok
    cipher: none

    db (current)
        wal archive min/max (11-1): 
            00000001000000000000005B / 00000001000000000000005D

        full backup: 20181127-164924F
            timestamp start/stop: 2018-11-27 16:49:24 / 2018-11-27 16:50:10
            wal start/stop: 00000001000000000000005B / 00000001000000000000005B
            database size: 1.4GB, backup size: 1.4GB
            repository size: 83.7MB, repository backup size: 83.7MB

        incr backup: 20181127-164924F_20181127-165743I
            timestamp start/stop: 2018-11-27 16:57:43 / 2018-11-27 16:57:54
            wal start/stop: 00000001000000000000005D / 00000001000000000000005D
            database size: 1.4GB, backup size: 50.5KB
            repository size: 83.7MB, repository backup size: 3.9KB
            backup reference list: 20181127-164924F
```

-----

# [](#conclusion)Conclusion

pgBackRest offers a lot of possibilities. We've seen in this post some tips to 
use in addition to Streaming Replication.

We've also seen in previous posts that changes can sometimes happen in 
PostgreSQL itself (eg. integrate recovery.conf into postgresql.conf). 

Using a tool supported by the community rather than your own script will also 
help you keep compatibility with those changes.