---
layout: post
title: introduction to pgBackRest
---

**Remark:** _This post has been updated on 2019-07-19, with pgBackRest 2.15_

pgBackRest (http://pgbackrest.org/) aims to be a simple, reliable backup and 
restore system that can seamlessly scale up to the largest databases and 
workloads.

Instead of relying on traditional backup tools like tar and rsync, pgBackRest 
implements all backup features internally and uses a custom protocol to 
communicate with remote systems. Removing reliance on tar and rsync allows 
better solutions to database-specific backup challenges. The custom remote 
protocol also allows more flexibility and limits the types of connections that 
are required to perform a backup which increases security.

<!--MORE-->

-----

pgBackRest has some great features, like:

- Parallel backup and restore, utilizing multiple cores
- Local or remote operation
- Full, incremental and differential backups

This article will present how to perform some basic actions:

- Installation
- Local backup and restore
- Remote backup

-----

# [](#installation)Installation

pgBackRest is written in Perl and C. The target is to implement it completely 
in C. Some additional Perl modules must then also be installed but they are 
available as standard packages.

Packages for pgBackRest are available in the PGDG yum repositories.

Let's install it on a CentOS 7 server: 

```bash
$ sudo yum install -y https://download.postgresql.org/pub/repos/yum/11/redhat/\
rhel-7-x86_64/pgdg-centos11-11-2.noarch.rpm
$ sudo yum install -y postgresql11-server postgresql11-contrib
$ sudo yum install -y pgbackrest
```

Check that it's correctly installed:

```bash
$ sudo -iu postgres pgbackrest
pgBackRest 2.15 - General help

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

Create a basic PostgreSQL cluster to backup:

```bash
$ export PGSETUP_INITDB_OPTIONS="--data-checksums"
$ sudo /usr/pgsql-11/bin/postgresql-11-setup initdb
$ sudo systemctl enable postgresql-11
$ sudo systemctl start postgresql-11
```

> All the configurations and actions bellow are based on that cluster.

-----

## Configure pgBackRest to backup the local cluster

By default, the configuration file is `/etc/pgbackrest.conf`. Let's make a copy:

```bash
$ sudo cp /etc/pgbackrest.conf /etc/pgbackrest.conf.bck
```

Let's then configure it:

```ini
[global]
repo1-path=/var/lib/pgsql/11/backups
log-level-console=info
log-level-file=debug
start-fast=y

[my_stanza]
pg1-path=/var/lib/pgsql/11/data
```

The `global` section allows to specify the repository to stores the backups 
and WAL segments archives.

The `my_stanza` section is called a `stanza`. 

A stanza is the configuration for a PostgreSQL database cluster that defines 
where it is located, how it will be backed up, archiving options, etc. 
Most db servers will only have one PostgreSQL database cluster and therefore 
one stanza, whereas backup servers will have a stanza for every database 
cluster that needs to be backed up.

Configure archiving in the `postgresql.conf` file:

```ini
archive_mode = on
archive_command = 'pgbackrest --stanza=my_stanza archive-push %p'
```

pgBackRest provides the `archive-push` command to push a WAL segment to the 
archive.

The PostgreSQL cluster must be restarted after making these changes and before 
performing a backup.

Let's finally create the stanza and check the configuration:

```bash
$ sudo -iu postgres pgbackrest --stanza=my_stanza stanza-create
P00   INFO: stanza-create command begin 2.15: ...
P00   INFO: stanza-create command end: completed successfully

$ sudo -iu postgres pgbackrest --stanza=my_stanza check
P00   INFO: check command begin 2.15: ...
P00   INFO: WAL segment ... successfully stored in the archive at ...
P00   INFO: check command end: completed successfully
```

-----

# [](#perform-backup)Perform a backup

pgBackRest offers several types of backup.

The `full backup` copies the entire content of the database cluster. The first 
backup of the database cluster must always be independent and then be a 
`full backup`.

The `differential backup` only copies the database cluster files that have 
changed since the last `full backup`.

```bash
$ sudo -iu postgres pgbackrest --stanza=my_stanza backup
WARN: option repo1-retention-full is not set, the repository may run out of space
      HINT: to retain full backups indefinitely (without warning), set option 'repo1-retention-full' to the maximum.
P00   INFO: backup command begin 2.15: ...
WARN: no prior backup exists, incr backup has been changed to full
P00   INFO: execute non-exclusive pg_start_backup() with label 
        "pgBackRest backup started at ...": backup begins after the requested immediate checkpoint completes
P00   INFO: backup start archive = 000000010000000000000003, lsn = 0/3000028
P00   INFO: full backup size = 23.5MB
P00   INFO: execute non-exclusive pg_stop_backup() and wait for all WAL segments to archive
P00   INFO: backup stop archive = 000000010000000000000003, lsn = 0/3000130
P00   INFO: new backup label = 20190719-090152F
P00   INFO: backup command end: completed successfully
P00   INFO: expire command begin
P00   INFO: option 'repo1-retention-archive' is not set - archive logs will not be expired
P00   INFO: expire command end: completed successfully

```

By default pgBackRest will attempt to perform an `incremental backup` in order 
to only copy the database cluster files that have changed since the last backup, 
no matter what type it was.

If no previous backup exists, it runs a `full backup`.

pgBackRest expires backups based on retention options.

To configure it, edit `/etc/pgbackrest.conf`:

```ini
[global]
repo1-path=/var/lib/pgsql/11/backups
log-level-console=info
log-level-file=debug
start-fast=y

[my_stanza]
pg1-path=/var/lib/pgsql/11/data
repo1-retention-full=1
```

Launch another `full backup` to see the removal:

```bash
$ sudo -iu postgres pgbackrest --stanza=my_stanza --type=full backup
P00   INFO: backup command begin 2.15: ...
P00   INFO: execute non-exclusive pg_start_backup() with label 
        "pgBackRest backup started at ...": backup begins after the requested immediate checkpoint completes
P00   INFO: backup start archive = 000000010000000000000005, lsn = 0/5000028
P00   INFO: full backup size = 23.5MB
P00   INFO: execute non-exclusive pg_stop_backup() and wait for all WAL segments to archive
P00   INFO: backup stop archive = 000000010000000000000005, lsn = 0/5000130
P00   INFO: new backup label = 20190719-091209F
P00   INFO: backup command end: completed successfully
P00   INFO: expire command begin
P00   INFO: expire full backup 20190719-090152F
P00   INFO: remove expired backup 20190719-090152F
P00   INFO: expire command end: completed successfully
```

-----

# [](#backup-info)Show backup information

```bash
$ sudo -iu postgres pgbackrest info
stanza: my_stanza
    status: ok
    cipher: none

    db (current)
        wal archive min/max (11-1): 000000010000000000000005/000000010000000000000005

        full backup: 20190719-091209F
            timestamp start/stop: 2019-07-19 09:12:09 / 2019-07-19 09:12:21
            wal start/stop: 000000010000000000000005 / 000000010000000000000005
            database size: 23.5MB, backup size: 23.5MB
            repository size: 2.8MB, repository backup size: 2.8MB
```

Use the `--stanza` option to filter on a specific stanza name.

-----

# [](#perform-restore)Restore a backup

Stop the local PostgreSQL cluster and remove its data files:

```bash
$ sudo systemctl stop postgresql-11
$ sudo find /var/lib/pgsql/11/data -mindepth 1 -delete
```

Perform the restore:

```bash
$ sudo -iu postgres pgbackrest --stanza=my_stanza restore
P00   INFO: restore command begin 2.15: ...
P00   INFO: restore backup set 20190719-091209F
P00   INFO: write /var/lib/pgsql/11/data/recovery.conf
P00   INFO: restore global/pg_control (performed last to ensure aborted restores cannot be started)
P00   INFO: restore command end: completed successfully
```

Start the cluster:

```bash
$ sudo systemctl start postgresql-11
```

To avoid having to clean the data directory before the restore, it's possible 
to use the `--delta` option to automatically determine which files can be 
preserved and which ones need to be restored from the backup.

-----

# [](#tips)Tips

## [](#option-db-include)Restore a specific database

There may be cases where it is desirable to selectively restore specific 
databases from a cluster backup.

Let's create two test databases and perform an incremental backup:

```bash
$ sudo -iu postgres psql -c "create database test1;"
CREATE DATABASE
$ sudo -iu postgres psql -c "create database test2;"
CREATE DATABASE
$ sudo -iu postgres pgbackrest --stanza=my_stanza --type=incr backup
```

Create some data using pgbench:

```bash
$ sudo -iu postgres /usr/pgsql-11/bin/pgbench -i -s 10 -d test1
$ sudo -iu postgres /usr/pgsql-11/bin/pgbench -i -s 10 -d test2
```

Restore only test2:

```bash
$ sudo systemctl stop postgresql-11
$ sudo -iu postgres pgbackrest --stanza=my_stanza --delta --db-include=test2 restore
$ sudo systemctl start postgresql-11
```

Check the data content of test2:

```bash
$ sudo -iu postgres psql -c "select count(*) from pgbench_accounts;" test2
  count  
---------
 1000000
(1 row)
```

Since the test1 database is restored with sparse, zeroed files it will only 
require as much space as the amount of WAL that is written during recovery and 
will not be reachable:

```bash
$ sudo -iu postgres psql -c "select count(*) from pgbench_accounts;" test1
psql: FATAL:  relation mapping file "base/16384/pg_filenode.map" contains invalid data
```

It's then best to drop it:

```bash
$ sudo -iu postgres psql -c "drop database test1;"
DROP DATABASE
```

-----

## [](#parallel-backup-restore)Parallel backup and restore

pgBackRest offers parallel processing to improve performance of compression 
and transfer. The number of processes to be used for this feature is set 
using the `--process-max` option.

It may also be configured directly in the `/etc/pgbackrest.conf` file under 
the `global` section.

For very small backups the difference may not be very apparent, but as the 
size of the database increases so will time savings.

-----

# [](#backup-host)Configure a dedicated backup host

## Installation

On another CentOS 7 server, install pgBackRest:

```bash
$ sudo yum install -y https://download.postgresql.org/pub/repos/yum/11/redhat/\
rhel-7-x86_64/pgdg-centos11-11-2.noarch.rpm
$ sudo yum install -y pgbackrest
```

Create and configure a specific user:

```bash
$ sudo useradd --system --home-dir "/var/lib/pgbackrest" --comment "pgBackRest" backrest
$ sudo chmod 750 /var/lib/pgbackrest
$ sudo chown backrest:backrest /var/lib/pgbackrest
$ sudo chown backrest:backrest /var/log/pgbackrest
```

-----

## Setup a trusted SSH communication between the hosts

Create db-primary-host (the first server used) key pair and ensure the correct SELinux contexts:

```bash
$ sudo -u postgres ssh-keygen -N "" -t rsa -b 4096 -f /var/lib/pgsql/.ssh/id_rsa
$ sudo -u postgres restorecon -R /var/lib/pgsql/.ssh
```

Create backup-host (the second server) key pair and ensure the correct SELinux contexts:

```bash
$ sudo -u backrest ssh-keygen -N "" -t rsa -b 4096 -f /var/lib/pgbackrest/.ssh/id_rsa
$ sudo -u backrest restorecon -R /var/lib/pgbackrest/.ssh
```

On db-primary-host, copy backup-host public key and test the connexion:

```bash
$ sudo ssh root@backup-host cat /var/lib/pgbackrest/.ssh/id_rsa.pub | \
       sudo -u postgres tee -a /var/lib/pgsql/.ssh/authorized_keys

$ sudo -u postgres ssh backrest@backup-host
```

On backup-host, copy db-primary-host public key and test the connexion:

```bash
$ sudo ssh root@db-primary-host cat /var/lib/pgsql/.ssh/id_rsa.pub | \
       sudo -u backrest tee -a /var/lib/pgbackrest/.ssh/authorized_keys

$ sudo -u backrest ssh postgres@db-primary-host
```

-----

## Configuration

On db-primary-host, change `/etc/pgbackrest.conf`:

```ini
[global]
repo1-host=backup-host
repo1-host-user=backrest
process-max=2
log-level-console=info
log-level-file=debug

[db-primary]
pg1-path=/var/lib/pgsql/11/data
```

Change the `postgresql.conf` accordingly:

```ini
archive_command = 'pgbackrest --stanza=db-primary archive-push %p'
```

Reload PostgreSQL:

```bash
$ sudo systemctl reload postgresql-11
```

Remove the old backup repository:

```bash
$ sudo find /var/lib/pgsql/11/backups -mindepth 1 -delete
```

On the backup-host, configure `/etc/pgbackrest.conf`:

```ini
[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=1
process-max=2
log-level-console=info
log-level-file=debug
start-fast=y
stop-auto=y

[db-primary]
pg1-path=/var/lib/pgsql/11/data
pg1-host=db-primary-host
pg1-host-user=postgres
```

Create the stanza and check the configuration:

```bash
$ sudo -iu backrest pgbackrest --stanza=db-primary stanza-create
$ sudo -iu backrest pgbackrest --stanza=db-primary check
```

Finally, check the configuration on db-primary-host: 

```bash
$ sudo -iu postgres pgbackrest --stanza=db-primary check
```

-----

## Perform a backup

Run the backup command on backup-host:

```bash
$ sudo -iu backrest pgbackrest --stanza=db-primary backup
```

Finally, get the backup information:

```bash
$ sudo -iu backrest pgbackrest info
stanza: db-primary
    status: ok
    cipher: none

    db (current)
        wal archive min/max (11-1): 00000003000000000000001D/00000003000000000000001D

        full backup: 20190719-094116F
            timestamp start/stop: 2019-07-19 09:41:16 / 2019-07-19 09:41:33
            wal start/stop: 00000003000000000000001D / 00000003000000000000001D
            database size: 180.9MB, backup size: 180.9MB
            repository size: 11.7MB, repository backup size: 11.7MB
```

The restore command can then be used on the db-primary-host.

-----

In conclusion, pgBackRest offers a large amount of possibilities and use-cases.

It is quite simple to install, configure and use, simplifying Point-in-time 
recovery through WAL archiving.