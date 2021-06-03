---
layout: post
title: pgBackRest and SUSE
draft: true
---

I got the question the other day from a friend: "does pgBackRest work on SUSE?".
Having to admit I never really used SUSE, and not knowing what to answer, I decided to give it a try.
Let's see in this short post how far we can go.

<!--MORE-->

-----

# Vagrant box

The openSUSE project provides several [vagrant boxes](https://app.vagrantup.com/opensuse) including `Leap 15.2`:

```bash
$ vagrant init opensuse/Leap-15.2.x86_64
$ vagrant up
$ vagrant ssh

vagrant@localhost:~> cat /etc/os-release 
NAME="openSUSE Leap"
VERSION="15.2"
```

-----

# PostgreSQL

Thanks to the packaging team, we can use the [zypper](https://zypp.postgresql.org/howtozypp/) package manager to install PostgreSQL and/or other components.

In case you'd use SLES (aka. __SUSE Linux Enterprise Server__), some package dependencies requiring access to specific repositories not hosted at __postgresql.org__ might be needed.

Let's now install PostgreSQL 13 on the vagrant box:

```bash
root# zypper addrepo https://download.postgresql.org/pub/repos/zypp/repo/pgdg-sles-15-pg13.repo
root# zypper refresh
root# zypper install postgresql13-server
root# export PGSETUP_INITDB_OPTIONS="--data-checksums"
root# /usr/pgsql-13/bin/postgresql-13-setup initdb
root# systemctl enable postgresql-13
root# systemctl start postgresql-13
```

Check if the cluster is running:

```bash
postgres$ ps -o pid,cmd fx
  PID CMD
 5359 ps -o pid,cmd fx
 5194 /usr/pgsql-13/bin/postmaster -D /var/lib/pgsql/13/data/
 5195  \_ postgres: logger 
 5197  \_ postgres: checkpointer 
 5198  \_ postgres: background writer 
 5199  \_ postgres: walwriter 
 5200  \_ postgres: autovacuum launcher 
 5201  \_ postgres: stats collector 
 5202  \_ postgres: logical replication launcher
```

-----

# pgBackRest packages

The currently latest pgBackRest release is available in the PostgreSQL community repository:

```bash
root# zypper install pgbackrest
postgres$ pgbackrest version
pgBackRest 2.33
```

## Configuration

We'll now prepare the configuration for our `demo` stanza:

```ini
# /etc/pgbackrest.conf
[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=1
process-max=2
log-level-console=info
log-level-file=debug
start-fast=y
repo1-cipher-type=aes-256-cbc
repo1-cipher-pass=7jYdsTY9m2rweLMw3VydfsNPY25R7g0q

[demo]
pg1-path=/var/lib/pgsql/13/data
```

The `base64` cipher-pass has been randomly generated with:

```bash
$ openssl rand -base64 24
7jYdsTY9m2rweLMw3VydfsNPY25R7g0q
```

In production environment, it is **recommended** to store the backups on a remote storage (cifs, nfs, S3, Azure, GCS,...).
As per simplicity for this demo, I located the backup repository locally.

Configure archiving in the `postgresql.conf` file:

```
listen_addresses = '*'
archive_mode = on
archive_command = 'pgbackrest --stanza=demo archive-push %p'
```

The PostgreSQL cluster must be restarted after making these changes and before performing a backup.

```bash
root# systemctl restart postgresql-13.service
```

Let's finally create the stanza and check the configuration:

```bash
postgres$ pgbackrest --stanza=demo stanza-create
...
P00   INFO: stanza-create for stanza 'demo' on repo1
P00   INFO: stanza-create command end: completed successfully

postgres$ pgbackrest --stanza=demo check
...
P00   INFO: check repo1 configuration (primary)
P00   INFO: check repo1 archive for WAL (primary)
P00   INFO: WAL segment 000000010000000000000001 successfully archived to '...' on repo1
P00   INFO: check command end: completed successfully
```

Using `pgbench`, let's create some test data:

```bash
postgres$ createdb test
postgres$ /usr/pgsql-13/bin/pgbench -i -s 100 test
```

## Perform a backup

Let's take the first backup :

```bash
postgres$ pgbackrest --stanza=demo --type=full backup
P00   INFO: backup command begin 2.33: ...
P00   INFO: execute non-exclusive pg_start_backup(): backup begins after the requested immediate checkpoint completes
P00   INFO: backup start archive = 00000001000000000000009F, lsn = 0/9F000028
P00   INFO: full backup size = 1.5GB
P00   INFO: execute non-exclusive pg_stop_backup() and wait for all WAL segments to archive
P00   INFO: backup stop archive = 00000001000000000000009F, lsn = 0/9F000CA8
P00   INFO: check archive for segment(s) 00000001000000000000009F:00000001000000000000009F
P00   INFO: new backup label = 20210603-065339F
P00   INFO: backup command end: completed successfully
P00   INFO: expire command begin 2.33: ...
P00   INFO: expire command end: completed successfully
```

The `info` command will give you some information about the backups and archives:

```bash
postgres$ pgbackrest --stanza=demo info
stanza: demo
    status: ok
    cipher: aes-256-cbc

    db (current)
        wal archive min/max (13): 00000001000000000000009F/00000001000000000000009F

        full backup: 20210603-065339F
            timestamp start/stop: 2021-06-03 06:53:39 / 2021-06-03 06:54:30
            wal start/stop: 00000001000000000000009F / 00000001000000000000009F
            database size: 1.5GB, database backup size: 1.5GB
            repo1: backup set size: 85MB, backup size: 85MB
```

Everything is working as expected so far.

-----

# pgBackRest build from sources

In case you'd wish to build the latest sources, install prerequisites and adjust compilation flags if needed:

```bash
root# zypper install unzip make gcc openssl-devel libxml2-devel libbz2-devel
root# zypper install postgresql13-devel
root# curl -LJO https://github.com/pgbackrest/pgbackrest/archive/refs/heads/master.zip
root# export CPPFLAGS='-I /usr/pgsql-13/include'
root# export LDFLAGS='-L/usr/pgsql-13/lib'
root# cd pgbackrest-master/src
root# ./configure
root# make
root# mv pgbackrest /usr/bin/pgbackrest_build
root# pgbackrest_build version
pgBackRest 2.34dev
```

Finally, let's take a backup using the built package:

```bash
postgres$ pgbackrest_build --stanza=demo --type=incr backup
P00   INFO: backup command begin 2.34dev: ...
P00   INFO: last backup label = 20210603-065339F, version = 2.33
P00   INFO: execute non-exclusive pg_start_backup(): backup begins after the requested immediate checkpoint completes
P00   INFO: backup start archive = 0000000100000000000000A3, lsn = 0/A3000028
P00   INFO: incr backup size = 198.1KB
P00   INFO: execute non-exclusive pg_stop_backup() and wait for all WAL segments to archive
P00   INFO: backup stop archive = 0000000100000000000000A3, lsn = 0/A3000100
P00   INFO: check archive for segment(s) 0000000100000000000000A3:0000000100000000000000A3
P00   INFO: new backup label = 20210603-065339F_20210603-071650I
P00   INFO: backup command end: completed successfully
P00   INFO: expire command begin 2.34dev: ...
P00   INFO: repo1: 13-1 no archive to remove
P00   INFO: expire command end: completed successfully

postgres$ pgbackrest_build --stanza=demo info
stanza: demo
    status: ok
    cipher: aes-256-cbc

    db (current)
        wal archive min/max (13): 00000001000000000000009F/0000000100000000000000A3

        full backup: 20210603-065339F
            timestamp start/stop: 2021-06-03 06:53:39 / 2021-06-03 06:54:30
            wal start/stop: 00000001000000000000009F / 00000001000000000000009F
            database size: 1.5GB, database backup size: 1.5GB
            repo1: backup set size: 85MB, backup size: 85MB

        incr backup: 20210603-065339F_20210603-071650I
            timestamp start/stop: 2021-06-03 07:16:50 / 2021-06-03 07:16:52
            wal start/stop: 0000000100000000000000A3 / 0000000100000000000000A3
            database size: 1.5GB, database backup size: 198.4KB
            repo1: backup set size: 85MB, backup size: 19.7KB
            backup reference list: 20210603-065339F
```

-----

# Conclusion

Given this quick test, at least on openSUSE 15, pgBackRest seems to be working just fine.
Thanks again to the packaging team allowing us to install the tools we love so easily!