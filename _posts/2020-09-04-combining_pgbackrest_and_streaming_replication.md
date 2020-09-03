---
layout: post
title: Combining pgBackRest and Streaming Replication, PG13 update
draft: true
---

[pgBackRest](http://pgbackrest.org/) is a well-known powerful backup and 
restore tool. It offers a lot of possibilities.

While `pg_basebackup` is commonly used to setup the initial database copy for 
the Streaming Replication, it could be interesting to reuse a previous 
database backup (eg. taken with pgBackRest) to perform this initial copy.

This content updates one of my old posts, using PostgreSQL 13 and the latest 
pgBackRest version.

<!--MORE-->

-----

For the purpose of this post, we'll use 2 nodes called **primary** and 
**secondary**. Both are running on CentOS 7.

We'll cover some pgBackRest tips but won't go deeper in the PostgreSQL 
configuration, nor in the Streaming Replication best practices.

-----

# Installation

On both **primary** and **secondary** server, first configure the PGDG yum 
repositories:

```bash
$ sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/\
EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

Since PostgreSQL 13 is still in Beta version, we need to enable the `testing` 
repository:

```bash
sudo yum -y install yum-utils
sudo yum-config-manager --enable pgdg13-updates-testing
sudo yum search postgresql13
```

Then, install PostgreSQL and pgBackRest: 

```bash
$ sudo yum install -y postgresql13-server postgresql13-contrib
$ sudo yum install -y pgbackrest
```

Check that pgBackRest is correctly installed:

```bash
$ pgbackrest
pgBackRest 2.29 - General help

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
```

Create a basic PostgreSQL cluster on **primary**:

```bash
$ export PGSETUP_INITDB_OPTIONS="--data-checksums"
$ /usr/pgsql-13/bin/postgresql-13-setup initdb
$ sudo systemctl enable postgresql-13
$ sudo systemctl start postgresql-13
```

-----

## Setup a shared repository between the hosts

To be able to share the backups between the hosts, we'll here create a nfs 
export from **secondary** and mount it on **primary**.

Install and activate nfs server on **secondary**:

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
$ sudo sh -c 'echo "/mnt/backups primary(rw,sync,no_root_squash)" >> /etc/exports'
$ sudo exportfs -a
```

Install nfs client and mount the shared repository on **primary**:

```bash
$ sudo yum -y install nfs-utils
$ sudo mkdir /mnt/backups
$ sudo chown postgres: /mnt/backups/
$ sudo sh -c 'echo "secondary:/mnt/backups /mnt/backups nfs rw,sync,hard,intr 0 0" >> /etc/fstab'
$ sudo mount /mnt/backups/
```

The storage of your backups is completely up to you. The requirement here is 
to have that storage available on both servers. It could be a dedicated backup 
storage (internally or even remotely like S3,...) or even a specific pgBackRest 
backup host.

-----

## Configure pgBackRest to backup the local cluster

By default, the configuration file is `/etc/pgbackrest.conf`. Let's make a copy:

```bash
$ sudo cp /etc/pgbackrest.conf /etc/pgbackrest.conf.bck
```

Update the **primary** configuration:

```ini
[global]
repo1-path=/mnt/backups
repo1-retention-full=1
process-max=2
log-level-console=info
log-level-file=debug
start-fast=y
repo1-cipher-type=aes-256-cbc
repo1-cipher-pass=acbd

[mycluster]
pg1-path=/var/lib/pgsql/13/data
```

Configure archiving in the `postgresql.conf` file:

```
listen_addresses = '*'
archive_mode = on
archive_command = 'pgbackrest --stanza=mycluster archive-push %p'
```

The PostgreSQL cluster must be restarted after making these changes and 
before performing a backup.

```bash
$ sudo systemctl restart postgresql-13.service
```

Let's finally create the stanza and check the configuration:

```bash
$ sudo -iu postgres pgbackrest --stanza=mycluster stanza-create
...
P00   INFO: stanza-create command end: completed successfully

$ sudo -iu postgres pgbackrest --stanza=mycluster check
...
P00   INFO: check command end: completed successfully
```

-----

## Insert some test data in the database

Using pgbench, let's create some test data:

```bash
$ sudo -iu postgres createdb test
$ sudo -iu postgres /usr/pgsql-13/bin/pgbench -i -s 100 test
```

-----

# Prepare the servers for Streaming Replication

On **primary** server, create a specific user for the replication:

```bash
$ sudo -iu postgres psql
postgres=# CREATE ROLE replic_user WITH LOGIN REPLICATION PASSWORD 'mypwd';
```

Configure `pg_hba.conf`:

```
host	replication	replic_user	secondary		scram-sha-256
```

Reload configuration and allow the service in the firewall (if needed):

```bash
$ sudo systemctl reload postgresql-13.service
$ sudo firewall-cmd --permanent --add-service=postgresql
$ sudo firewall-cmd --reload
```

Configure `~postgres/.pgpass` on **secondary** servers:

```bash
$ echo "*:*:replication:replic_user:mypwd" >> ~postgres/.pgpass
$ chown postgres: ~postgres/.pgpass
$ chmod 0600 ~postgres/.pgpass
```

-----

# Perform a backup

Let's take our first backup on the **primary** server:

```bash
$ sudo -iu postgres pgbackrest --stanza=mycluster --type=full backup
P00   INFO: backup command begin 2.29: ...
P00   INFO: execute non-exclusive pg_start_backup(): backup begins after the requested immediate checkpoint completes
P00   INFO: backup start archive = 00000001000000000000009F, lsn = 0/9F000028
P00   INFO: full backup size = 1.5GB
P00   INFO: execute non-exclusive pg_stop_backup() and wait for all WAL segments to archive
P00   INFO: backup stop archive = 00000001000000000000009F, lsn = 0/9F000138
P00   INFO: check archive for segment(s) 00000001000000000000009F:00000001000000000000009F
P00   INFO: new backup label = 20200903-133206F
P00   INFO: backup command end: completed successfully
P00   INFO: expire command begin 2.29: ...
P00   INFO: expire command end: completed successfully
```

What's important to notice here is that just after a successful backup, the 
`expire` command is directly executed to remove old backups and archives 
according to the retention policy we configured. Add the 
[`expire-auto`](https://pgbackrest.org/configuration.html#section-backup/option-expire-auto) 
option allows to disable that default behavior.

The `info` command will give you some information about the backups and archives:

```bash
$ sudo -iu postgres pgbackrest --stanza=mycluster info
stanza: mycluster
    status: ok
    cipher: aes-256-cbc

    db (current)
        wal archive min/max (13-1): 00000001000000000000009F/00000001000000000000009F

        full backup: 20200903-133206F
            timestamp start/stop: 2020-09-03 13:32:06 / 2020-09-03 13:32:54
            wal start/stop: 00000001000000000000009F / 00000001000000000000009F
            database size: 1.5GB, backup size: 1.5GB
            repository size: 85MB, repository backup size: 85MB
```

-----

# Secondary setup

Configure `/etc/pgbackrest.conf`:

```ini
[global]
repo1-path=/mnt/backups
repo1-retention-full=1
process-max=2
log-level-console=info
log-level-file=debug
start-fast=y
repo1-cipher-type=aes-256-cbc
repo1-cipher-pass=acbd

[mycluster]
pg1-path=/var/lib/pgsql/13/data
recovery-option=primary_conninfo=host=primary user=replic_user
```

Make sure the configuration is correct by executing the `info` command. It 
should print you the same output as above on the **primary** server.

Restore the backup taken from the **primary** server:

```bash
$ sudo -iu postgres pgbackrest --stanza=mycluster --type=standby restore
P00   INFO: restore command begin 2.29: ...
P00   INFO: restore backup set 20200903-133206F
P00   INFO: write updated /var/lib/pgsql/13/data/postgresql.auto.conf
P00   INFO: restore global/pg_control (performed last to ensure aborted restores cannot be started)
P00   INFO: restore command end: completed successfully
```

The restore will add extra information to the `postgresql.auto.conf` file:

```ini
# Recovery settings generated by pgBackRest restore...
primary_conninfo = 'host=primary user=replic_user'
restore_command = 'pgbackrest --stanza=mycluster archive-get %f "%p"'
```

In addition to the `restore_command`, pgBackRest will add all the 
`recovery-option` parameters we configured in the `pgbackrest.conf` file. 
Target Recovery options (`recovery_target_time`, etc.) are generated 
automatically by pgBackRest and should not be set with this option.

The `--type=standby` option creates the `standby.signal` needed for PostgreSQL 
to start in standby mode. All we have to do now is to start the PostgreSQL 
cluster:

```bash
$ sudo systemctl enable postgresql-13
$ sudo systemctl start postgresql-13
```

If the replication setup is correct, you should see those processes on the 
**secondary** server:

```
# ps -ef |grep postgres
postgres 27974     1  ... /usr/pgsql-13/bin/postmaster -D /var/lib/pgsql/13/data/
postgres 27977 27974  ... postgres: startup recovering 0000000100000000000000A0
postgres 27984 27974  ... postgres: walreceiver streaming 0/A00007D0
```

We now have a 2-nodes cluster working with _Streaming Replication_ and archives 
recovery as safety net.

-----

## What if my PostgreSQL data directory is not empty?

In case your **secondary** server would not be synchronized with the 
**primary** server anymore (too much replication lag, WAL segments archives 
expired,...), rather than copying the entire data again, the 
[`--delta`](https://pgbackrest.org/configuration.html#section-general/option-delta) 
option would be incredibly helpful:

> Restore or backup using checksums.
> 
> During a restore, by default the PostgreSQL data and tablespace directories 
> are expected to be present but empty. 
> This option performs a delta restore using checksums.
> 
> During a backup, this option will use checksums instead of the timestamps 
> to determine if files will be copied.

Like the other parameters, it can be used with the `restore` command or 
directly set in the configuration file. 

-----

# Take backups from the **secondary** server

pgBackRest can perform backups on a standby server instead of the **primary**. 
Both the **primary** and **secondary** databases configuration are required, 
even if the majority of the files will be copied from the **secondary** to 
reduce load on the **primary**. 

**_Remark:_** To do so, you need to setup a trusted SSH communication between 
the hosts.

Adjust the `/etc/pgbackrest.conf` file on **secondary**:

```ini
[global]
repo1-path=/mnt/backups
repo1-retention-full=1
process-max=2
log-level-console=info
log-level-file=debug
start-fast=y
repo1-cipher-type=aes-256-cbc
repo1-cipher-pass=acbd
backup-standby=y
delta=y

[mycluster]
pg1-host=primary
pg1-path=/var/lib/pgsql/13/data
pg2-path=/var/lib/pgsql/13/data
recovery-option=primary_conninfo=host=primary user=replic_user
```

Options added are:

  * delta: to allow delta backup and restore without using `--delta`
  * backup-standby
  * pg1-host and pg1-path


Perform a backup from **secondary**:

```bash
$ sudo -iu postgres pgbackrest --stanza=mycluster --type=full backup
P00   INFO: backup command begin 2.29: ...
P00   INFO: execute non-exclusive pg_start_backup(): backup begins after the requested immediate checkpoint completes
P00   INFO: backup start archive = 0000000100000000000000A1, lsn = 0/A1000028
P00   INFO: wait for replay on the standby to reach 0/A1000028
P00   INFO: replay on the standby reached 0/A1000028
P00   INFO: full backup size = 1.5GB
P00   INFO: execute non-exclusive pg_stop_backup() and wait for all WAL segments to archive
P00   INFO: backup stop archive = 0000000100000000000000A1, lsn = 0/A1000138
P00   INFO: check archive for segment(s) 0000000100000000000000A1:0000000100000000000000A1
P00   INFO: new backup label = 20200903-144550F
P00   INFO: backup command end: completed successfully
P00   INFO: expire command begin 2.29: ...
P00   INFO: expire full backup 20200903-133206F
P00   INFO: remove expired backup 20200903-133206F
P00   INFO: expire command end: completed successfully
```

Even incremental backups can be taken:

```bash
$ sudo -iu postgres pgbackrest --stanza=mycluster --type=incr backup
P00   INFO: backup command begin 2.29: ...
P00   INFO: last backup label = 20200903-144550F, version = 2.29
P00   INFO: execute non-exclusive pg_start_backup(): backup begins after the requested immediate checkpoint completes
P00   INFO: backup start archive = 0000000100000000000000A3, lsn = 0/A3000028
P00   INFO: wait for replay on the standby to reach 0/A3000028
P00   INFO: replay on the standby reached 0/A3000028
P00   INFO: incr backup size = 1.5GB
P00   INFO: execute non-exclusive pg_stop_backup() and wait for all WAL segments to archive
P00   INFO: backup stop archive = 0000000100000000000000A3, lsn = 0/A3000100
P00   INFO: check archive for segment(s) 0000000100000000000000A3:0000000100000000000000A3
P00   INFO: new backup label = 20200903-144550F_20200903-145402I
P00   INFO: backup command end: completed successfully (8119ms)
P00   INFO: expire command begin 2.29: ...
P00   INFO: expire command end: completed successfully (20ms)

$ sudo -iu postgres pgbackrest --stanza=mycluster info
stanza: mycluster
    status: ok
    cipher: aes-256-cbc

    db (current)
        wal archive min/max (13-1): 0000000100000000000000A1/0000000100000000000000A3

        full backup: 20200903-144550F
            timestamp start/stop: 2020-09-03 14:45:50 / 2020-09-03 14:46:32
            wal start/stop: 0000000100000000000000A1 / 0000000100000000000000A1
            database size: 1.5GB, backup size: 1.5GB
            repository size: 85MB, repository backup size: 85MB

        incr backup: 20200903-144550F_20200903-145402I
            timestamp start/stop: 2020-09-03 14:54:02 / 2020-09-03 14:54:08
            wal start/stop: 0000000100000000000000A3 / 0000000100000000000000A3
            database size: 1.5GB, backup size: 90.0KB
            repository size: 85MB, repository backup size: 5.6KB
            backup reference list: 20200903-144550F
```

-----

# Conclusion

pgBackRest offers a lot of possibilities. 

While PostgreSQL 13 is still in Beta, pgBackRest latest version is already 
compatible with it. That's why using a tool supported by the community is 
often way better than using your own script.