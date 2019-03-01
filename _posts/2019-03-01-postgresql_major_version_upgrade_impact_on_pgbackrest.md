---
layout: post
title: PostgreSQL major version upgrade impact on pgBackRest
draft: true
---

[pgBackRest](http://pgbackrest.org/) is a well-known powerful backup and 
restore tool. 

While it works with a really simple configuration, a major version upgrade of 
PostgreSQL has some impact on it.

<!--MORE-->

-----

For the purpose of this post, I'll use a fresh CentOS 7 install.

We'll talk about the `stanza-upgrade` command of pgBackRest but won't go 
deeper in the PostgreSQL configuration, nor in the PostgreSQL major version 
upgrade best practices.

-----

# [](#installation)Installation

First of all, install PostgreSQL and pgBackRest packages directly from the 
PGDG yum repositories:

```bash
$ sudo yum install -y https://download.postgresql.org/pub/repos/yum/10/redhat/\
rhel-7-x86_64/pgdg-centos10-10-2.noarch.rpm
$ sudo yum install -y postgresql10-server postgresql10-contrib
$ sudo yum install -y pgbackrest
```

Check that pgBackRest is correctly installed:

```bash
$ pgbackrest
pgBackRest 2.10 - General help

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

Create a basic PostgreSQL cluster with some data in it:

```bash
$ sudo /usr/pgsql-10/bin/postgresql-10-setup initdb
$ sudo systemctl start postgresql-10
$ sudo -iu postgres createdb bench
$ sudo -iu postgres /usr/pgsql-10/bin/pgbench -i -s 100 bench
```

-----

## Configure pgBackRest to backup the local cluster

By default, the configuration file is `/etc/pgbackrest.conf`. 
Let's make a copy:

```bash
$ sudo cp /etc/pgbackrest.conf /etc/pgbackrest.conf.bck
```

Update the configuration:

```ini
[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=1
process-max=2
log-level-console=info
log-level-file=debug

[some_cool_stanza_name]
pg1-path=/var/lib/pgsql/10/data
```

Make sure that the postgres user can write in `/var/lib/pgbackrest`.

Configure archiving in the `postgresql.conf` file:

```
archive_mode = on
archive_command = 'pgbackrest --stanza=some_cool_stanza_name archive-push %p'
```

The PostgreSQL cluster must be restarted after making these changes and 
before performing a backup.

Let's finally create the stanza and check the configuration:

```bash
$ sudo -iu postgres pgbackrest --stanza=some_cool_stanza_name stanza-create
P00   INFO: stanza-create command end: completed successfully

$ sudo -iu postgres pgbackrest --stanza=some_cool_stanza_name check
P00   INFO: WAL segment 00000001000000000000004E successfully stored in the 
    archive at '/var/lib/pgbackrest/archive/some_cool_stanza_name/
    10-1/0000000100000000/
    00000001000000000000004E-201c08f0d6be79ba6c9b08c7011bfdab156c4638.gz'
P00   INFO: check command end: completed successfully
```

-----

## Perform a backup and simulate some activity

Let's take our first backup:

```bash
$ sudo -iu postgres pgbackrest --stanza=some_cool_stanza_name --type=full backup
...
P00   INFO: new backup label = 20190301-102816F
P00   INFO: backup command end: completed successfully
...
```

Then, let's use `pgbench` to simulate some activity (over a 600s period):

```bash
$ sudo -iu postgres /usr/pgsql-10/bin/pgbench -c 10 -T 600 bench
```

-----

# [](#upgrade)PostgreSQL upgrade

The following instructions are not meant to be a full guide over upgrading 
PostgreSQL. I'll here use `pg_upgrade`.

Before running it you must:
* create a new database cluster (using the new version of initdb)
* shutdown the postmaster servicing the old cluster
* shutdown the postmaster servicing the new cluster

```bash
$ sudo yum install -y https://download.postgresql.org/pub/repos/yum/11/redhat/\
rhel-7-x86_64/pgdg-centos11-11-2.noarch.rpm
$ sudo yum install -y postgresql11-server postgresql11-contrib
$ sudo /usr/pgsql-11/bin/postgresql-11-setup initdb
$ sudo systemctl stop postgresql-10
$ sudo systemctl stop postgresql-11
```

When you run pg_upgrade, you must provide the following information:
* the data directory for the old cluster
* the data directory for the new cluster
* the "bin" directory for the old version
* the "bin" directory for the new version

```bash
$ sudo -iu postgres /usr/pgsql-11/bin/pg_upgrade \
--old-datadir=/var/lib/pgsql/10/data/ \
--new-datadir=/var/lib/pgsql/11/data/ \
--old-bindir=/usr/pgsql-10/bin \
--new-bindir=/usr/pgsql-11/bin \
--check 

Performing Consistency Checks
-----------------------------
Checking cluster versions                                   ok
Checking database user is the install user                  ok
Checking database connection settings                       ok
Checking for prepared transactions                          ok
Checking for reg* data types in user tables                 ok
Checking for contrib/isn with bigint-passing mismatch       ok
Checking for presence of required libraries                 ok
Checking database user is the install user                  ok
Checking for prepared transactions                          ok

*Clusters are compatible*
```

Don't forget to configure correctly your new cluster. Here, report the 
previous `postgresql.conf` configuration about `archive_mode` and 
`archive_command`.

Then, run `pg_upgrade` without the `--check` option. You should get a result 
like that:

```bash
Upgrade Complete
----------------
Optimizer statistics are not transferred by pg_upgrade so,
once you start the new server, consider running:
    ./analyze_new_cluster.sh

Running this script will delete the old cluster's data files:
    ./delete_old_cluster.sh
```

Once PostgreSQL updated, update the pgBackRest configuration 
(`/etc/pgbackrest.conf`) to point to the new cluster:

```ini
pg1-path=/var/lib/pgsql/11/data
```

Before starting the new PostgreSQL cluster, the `stanza-upgrade` command must 
be run:

```bash
$ sudo -iu postgres pgbackrest --stanza=some_cool_stanza_name --no-online stanza-upgrade
P00   INFO: stanza-upgrade command end: completed successfully
```

Start the new cluster, confirm it is successfully installed and test the configuration:

```bash
$ sudo systemctl start postgresql-11
$ sudo -iu postgres pgbackrest --stanza=some_cool_stanza_name check
P00   INFO: WAL segment 000000010000000100000040 successfully stored in the 
    archive at '/var/lib/pgbackrest/archive/some_cool_stanza_name/
    11-2/0000000100000001/
    000000010000000100000040-cb9963f36306e7a5410af9f29a37bd30a2d79b1c.gz'
P00   INFO: check command end: completed successfully
```

Refresh the optimizer statistics and remove the old cluster:

```bash
$ sudo -iu postgres ./analyze_new_cluster.sh
This script will generate minimal optimizer statistics rapidly
so your system is usable, and then gather statistics twice more
with increasing accuracy.  When it is done, your system will
have the default level of optimizer statistics.

If you have used ALTER TABLE to modify the statistics target for
any tables, you might want to remove them and restore them after
running this script because they will delay fast statistics generation.

If you would like default statistics as quickly as possible, cancel
this script and run:
    "/usr/pgsql-11/bin/vacuumdb" --all --analyze-only

vacuumdb: processing database "bench": Generating minimal optimizer statistics (1 target)
vacuumdb: processing database "postgres": Generating minimal optimizer statistics (1 target)
vacuumdb: processing database "template1": Generating minimal optimizer statistics (1 target)
vacuumdb: processing database "bench": Generating medium optimizer statistics (10 targets)
vacuumdb: processing database "postgres": Generating medium optimizer statistics (10 targets)
vacuumdb: processing database "template1": Generating medium optimizer statistics (10 targets)
vacuumdb: processing database "bench": Generating default (full) optimizer statistics
vacuumdb: processing database "postgres": Generating default (full) optimizer statistics
vacuumdb: processing database "template1": Generating default (full) optimizer statistics
Done

$ sudo -iu postgres ./delete_old_cluster.sh
```

Finally, take a new fresh full backup:

```bash
$ sudo -iu postgres pgbackrest --stanza=some_cool_stanza_name --type=full backup
...
P00   INFO: new backup label = 20190301-114628F
P00   INFO: backup command end: completed successfully
```

Since I configured `repo1-retention-full=1`, the expire command will react:

```bash
P00   INFO: expire command begin
P00   INFO: expire full backup 20190301-102816F
P00   INFO: remove expired backup 20190301-102816F
P00   INFO: remove archive path: /var/lib/pgbackrest/archive/some_cool_stanza_name/10-1
P00   INFO: expire command end: completed successfully
```

-----

# [](#conclusion)Conclusion

It's always better to take a backup before upgrading a major version of 
PostgreSQL. pgBackRest requires the `stanza-upgrade` to be executed to work 
with the new PostgreSQL version. 

It's not really complicated but it's definitively something you have to think 
about in your PostgreSQL upgrade procedure.