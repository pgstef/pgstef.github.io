---
layout: post
title: introduction to pgBackRest
---

pgBackRest (http://pgbackrest.org/) aims to be a simple, reliable backup and restore system that can seamlessly scale up to the largest databases and workloads.

Instead of relying on traditional backup tools like tar and rsync, pgBackRest implements all backup features internally and uses a custom protocol for communicating with remote systems. Removing reliance on tar and rsync allows for better solutions to database-specific backup challenges. The custom remote protocol allows for more flexibility and limits the types of connections that are required to perform a backup which increases security.

<!--MORE-->

-----

pgBackRest has some great features, like :

- Parallel backup and restore, utilizing multiple cores
- Local or remote operation
- Full, incremental and differential backups

# [](#installation)Installation

pgBackRest is written in Perl. Some additional modules must also be installed but they are available as standard packages.

Packages for pgBackRest are available in the postgresql.org repositories.

Let's install it on a CentOS 7 server : 

```bash
$ sudo yum install https://download.postgresql.org/pub/repos/yum/10/redhat/\
rhel-7-x86_64/pgdg-centos10-10-1.noarch.rpm
$ sudo yum install pgbackrest
```

Check that it's correctly installed :

```
$ sudo -u postgres pgbackrest
pgBackRest 1.27 - General help

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
    stanza-upgrade  Upgrade a stanza.
    start           Allow pgBackRest processes to run.
    stop            Stop pgBackRest processes from running.
    version         Get version.

Use 'pgbackrest help [command]' for more information.
```

Create a basic PostgreSQL cluster to backup :

```bash
$ sudo yum install postgresql10-server
$ sudo /usr/pgsql-10/bin/postgresql-10-setup initdb
$ sudo systemctl enable postgresql-10
$ sudo systemctl start postgresql-10
```

> All the configurations and actions bellow are based on that cluster.

-----

## Configure pgBackRest to backup the local cluster

By default, the configuration file is `/etc/pgbackrest.conf`. Let's make a copy :

```bash
$ sudo cp /etc/pgbackrest.conf /etc/pgbackrest.conf.bck
```

Let's then configure it :

```
[global]
repo-path=/var/lib/pgsql/10/backups

[localhost]
db-path=/var/lib/pgsql/10/data
```

The `global` section allows to specify the repository to stores the backups and WAL segments archives.

The `localhost` section is called a `stanza`. It's the name used by pgBackRest to define the configuration related to a PostgreSQL cluster, which allows to, among other parameters, specify where is located the data directory.

Configure archiving in the `postgresql.conf` file :

```
archive_mode = on
archive_command = 'pgbackrest --stanza=localhost archive-push %p'
```

pgBackRest provides the `archive-push` command to push a WAL segment to the archive.

The PostgreSQL cluster must be restarted after making these changes and before performing a backup.

Let's finally create the stanza and check the configuration :

```
$ sudo -u postgres pgbackrest --stanza=localhost --log-level-console=info stanza-create
P00   INFO: stanza-create command begin 1.27: --db1-path=/var/lib/pgsql/10/data --log-level-console=info --repo-path=/var/lib/pgsql/10/backups --stanza=localhost
P00   INFO: stanza-create command end: completed successfully

$ sudo -u postgres pgbackrest --stanza=localhost --log-level-console=info check
P00   INFO: check command begin 1.27: --db1-path=/var/lib/pgsql/10/data --log-level-console=info --repo-path=/var/lib/pgsql/10/backups --stanza=localhost
P00   INFO: WAL segment 000000010000000000000001 successfully stored in the archive at '/var/lib/pgsql/10/backups/archive/localhost/10-1/0000000100000000/000000010000000000000001-7097bb3b46550e1839accc378e3bc1f204ac85f4.gz'
P00   INFO: check command end: completed successfully
```

-----

# [](#perform-backup)Perform a backup

pgBackRest offers several types of backup.

The `full backup` copies the entire content of the database cluster. The first backup of the database cluster must always be independent and then be a `full backup`.

The `differential backup` only copies the database cluster files that have changed since the last `full backup`.

```
$ sudo -u postgres pgbackrest --stanza=localhost --log-level-console=info backup
WARN: option retention-full is not set, the repository may run out of space
HINT: to retain full backups indefinitely (without warning), set option 'retention-full' to the maximum.
P00   INFO: backup command begin 1.27: --db1-path=/var/lib/pgsql/10/data --log-level-console=info --repo-path=/var/lib/pgsql/10/backups --stanza=localhost
WARN: no prior backup exists, incr backup has been changed to full
P00   INFO: execute non-exclusive pg_start_backup() with label "pgBackRest backup started at 2018-01-04 09:23:31": backup begins after the next regular checkpoint completes
P00   INFO: backup start archive = 000000010000000000000003, lsn = 0/3000028
P00   INFO: full backup size = 23.2MB
P00   INFO: execute non-exclusive pg_stop_backup() and wait for all WAL segments to archive
P00   INFO: backup stop archive = 000000010000000000000003, lsn = 0/3000130
P00   INFO: new backup label = 20180104-092331F
P00   INFO: backup command end: completed successfully
P00   INFO: expire command begin 1.27: --log-level-console=info --repo-path=/var/lib/pgsql/10/backups --stanza=localhost
P00   INFO: option 'retention-archive' is not set - archive logs will not be expired
P00   INFO: expire command end: completed successfully
```

By default pgBackRest will attempt to perform an `incremental backup` in order to only copy the database cluster files that have changed since the last backup, no matter what type it was.

If no previous backup exists, it runs a `full backup`.

pgBackRest expires backups based on retention options.

To configure it, edit `/etc/pgbackrest.conf` :

```
[global]
repo-path=/var/lib/pgsql/10/backups

[localhost]
db-path=/var/lib/pgsql/10/data
retention-full=1
```

Launch another `full backup` to see the removal :

```
$ sudo -u postgres pgbackrest --stanza=localhost --log-level-console=info --type=full backup
P00   INFO: backup command begin 1.27: --db1-path=/var/lib/pgsql/10/data --log-level-console=info --repo-path=/var/lib/pgsql/10/backups --retention-full=1 --stanza=localhost --type=full
P00   INFO: execute non-exclusive pg_start_backup() with label "pgBackRest backup started at 2018-01-04 09:30:19": backup begins after the next regular checkpoint completes
P00   INFO: backup start archive = 000000010000000000000005, lsn = 0/5000028
P00   INFO: full backup size = 23.2MB
P00   INFO: execute non-exclusive pg_stop_backup() and wait for all WAL segments to archive
P00   INFO: backup stop archive = 000000010000000000000005, lsn = 0/50000F8
P00   INFO: new backup label = 20180104-093019F
P00   INFO: backup command end: completed successfully
P00   INFO: expire command begin 1.27: --log-level-console=info --repo-path=/var/lib/pgsql/10/backups --retention-archive=1 --retention-full=1 --stanza=localhost
P00   INFO: expire full backup 20180104-092331F
P00   INFO: remove expired backup 20180104-092331F
P00   INFO: expire command end: completed successfully
```

-----

# [](#backup-info)Show backup information

```
$ sudo -u postgres pgbackrest info
stanza: localhost
    status: ok

    db (current)
        wal archive min/max (10-1): 000000010000000000000005 / 000000010000000000000005

        full backup: 20180104-093019F
            timestamp start/stop: 2018-01-04 09:30:19 / 2018-01-04 09:30:28
            wal start/stop: 000000010000000000000005 / 000000010000000000000005
            database size: 23.2MB, backup size: 23.2MB
            repository size: 2.7MB, repository backup size: 2.7MB
```

Use the `--stanza` option to filter on a specific stanza name.

-----

# [](#perform-restore)Restore a backup

Stop the local PostgreSQL cluster and remove its data files :

```bash
$ sudo systemctl stop postgresql-10
$ sudo find /var/lib/pgsql/10/data -mindepth 1 -delete
```

Perform the restore :

```
$ sudo -u postgres pgbackrest --stanza=localhost --log-level-console=info restore
P00   INFO: restore command begin 1.27: --db1-path=/var/lib/pgsql/10/data --log-level-console=info --repo-path=/var/lib/pgsql/10/backups --stanza=localhost
P00   INFO: restore backup set 20180104-093019F
P00   INFO: write /var/lib/pgsql/10/data/recovery.conf
P00   INFO: restore global/pg_control (performed last to ensure aborted restores cannot be started)
P00   INFO: restore command end: completed successfully
```

Start the cluster :

```bash
$ sudo systemctl start postgresql-10
```

To avoid having to clean the data directory before the restore, it's possible to use the `--delta` option to automatically determine which files can be preserved and which ones need to be restored from the backup.

-----

# [](#tips)Tips

## [](#option-db-include)Restore a specific database

There may be cases where it is desirable to selectively restore specific databases from a cluster backup.

Let's create two test databases and perform an incremental backup :

```
$ sudo -u postgres psql -c "create database test1;"
CREATE DATABASE
$ sudo -iu postgres psql -c "create database test2;"
CREATE DATABASE
$ sudo -u postgres pgbackrest --stanza=localhost --type=incr backup
```

Create some data using pgbench :

```bash
$ sudo -u postgres /usr/pgsql-10/bin/pgbench -i -s 10 -d test1
$ sudo -u postgres /usr/pgsql-10/bin/pgbench -i -s 10 -d test2
```

Restore only test2 :

```bash
$ sudo systemctl stop postgresql-10
$ sudo -u postgres pgbackrest --stanza=localhost --delta --db-include=test2 restore
$ sudo systemctl start postgresql-10
```

Check the data content of test2 :

```
$ sudo -u postgres psql -c "select count(*) from pgbench_accounts;" test2
  count  
---------
 1000000
(1 row)
```

Since the test1 database is restored with sparse, zeroed files it will only require as much space as the amount of WAL that is written during recovery and will not be accessible :

```
$ sudo -u postgres psql -c "select count(*) from pgbench_accounts;" test1
psql: FATAL:  relation mapping file "base/16384/pg_filenode.map" contains invalid data
```

It's then best to drop it :

```
$ sudo -u postgres psql -c "drop database test1;"
DROP DATABASE
```

-----

## [](#parallel-backup-restore)Parallel backup and restore

pgBackRest offers parallel processing to improve performance of compression and transfer. The number of processes to be used for this feature is set using the `--process-max` option.

It may also be configured directly in the `/etc/pgbackrest.conf` file under the `global` section.

For very small backups the difference may not be very apparent, but as the size of the database increases so will time savings.

-----

# [](#backup-host)Configure a dedicated backup host

## Installation

On another CentOS 7 server, install pgBackRest :

```bash
$ sudo yum install https://download.postgresql.org/pub/repos/yum/10/redhat/\
rhel-7-x86_64/pgdg-centos10-10-1.noarch.rpm
$ sudo yum install pgbackrest
```

Create and configure a specific user :

```bash
$ sudo useradd --system --home-dir "/var/lib/pgbackrest" --comment "pgBackRest" backrest
$ sudo chmod 750 /var/lib/pgbackrest
$ sudo chown backrest:backrest /var/lib/pgbackrest
$ sudo chown backrest:backrest /var/log/pgbackrest
```

-----

## Setup a trusted SSH communication between the hosts

Create db-primary-host (the first server used) key pair and ensure the correct SELinux contexts :

```bash
$ sudo -u postgres ssh-keygen -N "" -t rsa -b 4096 -f /var/lib/pgsql/.ssh/id_rsa
$ sudo -u postgres restorecon -R /var/lib/pgsql/.ssh
```

Create backup-host (the second server) key pair and ensure the correct SELinux contexts :

```bash
$ sudo -u backrest ssh-keygen -N "" -t rsa -b 4096 -f /var/lib/pgbackrest/.ssh/id_rsa
$ sudo -u backrest restorecon -R /var/lib/pgbackrest/.ssh
```

On db-primary-host, copy backup-host public key and test the connexion :

```bash
$ sudo ssh root@backup-host cat /var/lib/pgbackrest/.ssh/id_rsa.pub | \
       sudo -u postgres tee -a /var/lib/pgsql/.ssh/authorized_keys

$ sudo -u postgres ssh backrest@backup-host
```

On backup-host, copy db-primary-host public key and test the connexion :

```bash
$ sudo ssh root@db-primary-host cat /var/lib/pgsql/.ssh/id_rsa.pub | \
       sudo -u backrest tee -a /var/lib/pgbackrest/.ssh/authorized_keys

$ sudo -u backrest ssh postgres@db-primary-host
```

-----

## Configuration

On db-primary-host, change `etc/pgbackrest.conf` :

```
[global]
backup-host=backup-host
backup-user=backrest
log-level-file=detail

[db-primary]
db-path=/var/lib/pgsql/10/data
```

Change the `postgresql.conf` accordingly :

```
archive_command = 'pgbackrest --stanza=db-primary archive-push %p'
```

Reload PostgreSQL :

```bash
$ sudo systemctl reload postgresql-10
```

Remove the old backup repository :

```bash
$ sudo find /var/lib/pgsql/10/backups -mindepth 1 -delete
```

On the backup-host, configure `/etc/pgbackrest.conf` :

```
[global]
repo-path=/var/lib/pgbackrest
retention-full=1

[db-primary]
db-host=db-primary-host
db-path=/var/lib/pgsql/10/data
db-user=postgres
```

Create the stanza and check the configuration :

```bash
$ sudo -u backrest pgbackrest --stanza=db-primary --log-level-console=info stanza-create
$ sudo -u backrest pgbackrest --stanza=db-primary --log-level-console=info check
```

Finally, check the configuration on db-primary-host : 

```bash
$ sudo -u postgres pgbackrest --stanza=db-primary --log-level-console=info check
```

-----

## Perform a backup

Run the backup command on backup-host :

```bash
$ sudo -u backrest pgbackrest --stanza=db-primary backup
```

Finally, get the backup information :

```
$ sudo -u backrest pgbackrest info
stanza: db-primary
    status: ok

    db (current)
        wal archive min/max (10-1): 00000003000000000000001E / 00000003000000000000001E

        full backup: 20180104-143735F
            timestamp start/stop: 2018-01-04 14:37:35 / 2018-01-04 14:37:55
            wal start/stop: 00000003000000000000001E / 00000003000000000000001E
            database size: 180.4MB, backup size: 180.4MB
            repository size: 11.6MB, repository backup size: 11.6MB
```

The restore command can then be used on the db-primary-host.

-----

In conclusion, pgBackRest offers a large amount of possibilities and use-cases.

It is quite simple to install, configure and use, simplifying Point-in-time recovery through WAL archiving.