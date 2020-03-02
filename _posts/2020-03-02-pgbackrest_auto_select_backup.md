---
layout: post
title: pgBackRest auto-select backup
---

[pgBackRest](http://pgbackrest.org/) is a well-known powerful backup and 
restore tool. 

The 2.24 version, released on February 25, introduced auto-selection of backup 
set on restore when time target is specified. Auto-selection is performed only 
when `--set` is not specified. If a backup set for the given target time can't 
be found, the latest (default) backup set will be used. 

Let's illustrate it!

<!--MORE-->

-----

# PostgreSQL and pgBackRest installation

Let's install PostgreSQL and pgBackRest directly from the PGDG yum repositories:

```bash
$ sudo yum install -y yum install https://download.postgresql.org/pub/repos/\
yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
$ sudo yum install -y postgresql12-server postgresql12-contrib
$ sudo yum install -y pgbackrest
```

Check that pgBackRest is correctly installed:

```bash
$ sudo -iu postgres pgbackrest
pgBackRest 2.24 - General help

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

Create a basic PostgreSQL cluster:

```bash
$ export PGSETUP_INITDB_OPTIONS="--data-checksums"
$ sudo /usr/pgsql-12/bin/postgresql-12-setup initdb
```

Configure archiving in the `postgresql.conf` file:

```bash
$ cat<<EOF | sudo tee -a "/var/lib/pgsql/12/data/postgresql.conf"
archive_mode = on
archive_command = 'pgbackrest --stanza=happy_birthday archive-push %p'
EOF
```

<!-- date -s "2 MAR 2020 09:00:00" -->

Start the PostgreSQL cluster:

```bash
$ sudo systemctl start postgresql-12
```

-----

## Configure pgBackRest

```bash
$ cat<<EOF | sudo tee "/etc/pgbackrest.conf"
[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=2
process-max=2
log-level-console=info
log-level-file=debug
start-fast=y
delta=y

[happy_birthday]
pg1-path=/var/lib/pgsql/12/data
EOF
```

Finally, create the stanza and check that everything works fine:

```bash
$ sudo -iu postgres pgbackrest --stanza=happy_birthday stanza-create
...
P00   INFO: stanza-create command end: completed successfully

$ sudo -iu postgres pgbackrest --stanza=happy_birthday check
...
P00   INFO: WAL segment 000000010000000000000001 successfully stored...
P00   INFO: check command end: completed successfully
```

-----

## Insert some data and take backups

Let's finally take our first backup:

```bash
$ sudo -iu postgres pgbackrest --stanza=happy_birthday --type=full backup
...
P00   INFO: new backup label = 20200302-090023F
P00   INFO: backup command end: completed successfully
...
```

Create some data and a specific restore point:

```sql
postgres=# CREATE TABLE t1(id int, c1 text);
CREATE TABLE
postgres=# INSERT INTO t1 VALUES (1, 'Happy Birthday');
INSERT 0 1
postgres=# select pg_create_restore_point('RP1');
 pg_create_restore_point 
-------------------------
 0/5000090
(1 row)
```

Find the precise time when the restore point has been created in the logs:

```
2020-03-02 09:05:45.050 CET [27122] LOG:  restore point "RP1" created at 0/5000090
2020-03-02 09:05:45.050 CET [27122] STATEMENT:  select pg_create_restore_point('RP1');
```

Insert another data:

```sql
postgres=# SELECT now();
              now              
-------------------------------
 2020-03-02 09:05:58.558377+01
(1 row)
postgres=# INSERT INTO t1 VALUES (2, 'happy birthday');
INSERT 0 1
postgres=# SELECT pg_switch_wal();
 pg_switch_wal 
---------------
 0/5000188
(1 row)
```

Take a new backup:

```bash
$ sudo -iu postgres pgbackrest --stanza=happy_birthday --type=full backup
...
P00   INFO: new backup label = 20200302-090658F
P00   INFO: backup command end: completed successfully
...
```

Check backup status:

```bash
$ sudo -iu postgres pgbackrest info
stanza: happy_birthday
    status: ok
    cipher: none

    db (current)
        wal archive min/max (12-1): 000000010000000000000003/000000010000000000000006

        full backup: 20200302-090023F
            timestamp start/stop: 2020-03-02 09:00:23 / 2020-03-02 09:00:34
            wal start/stop: 000000010000000000000003 / 000000010000000000000003
            database size: 24.2MB, backup size: 24.2MB
            repository size: 2.9MB, repository backup size: 2.9MB

        full backup: 20200302-090658F
            timestamp start/stop: 2020-03-02 09:06:58 / 2020-03-02 09:07:03
            wal start/stop: 000000010000000000000006 / 000000010000000000000006
            database size: 24.2MB, backup size: 24.2MB
            repository size: 2.9MB, repository backup size: 2.9MB
```

-----

# Restore using the restore point

Stop the PostgreSQL cluster and restore the backup using our restore point:

```bash
$ sudo systemctl stop postgresql-12

$ sudo -iu postgres pgbackrest restore --stanza=happy_birthday --type=name --target=RP1
P00   INFO: restore backup set 20200302-090658F
...
P00   INFO: restore command end: completed successfully
```

Check the restore settings:

```bash
$ cat /var/lib/pgsql/12/data/postgresql.auto.conf
restore_command = 'pgbackrest --stanza=happy_birthday archive-get %f "%p"'
recovery_target_name = 'RP1'
```

Start the cluster and check the data:

```bash
$ sudo systemctl start postgresql-12

$ sudo -iu postgres psql -c "SELECT * FROM t1;"
 id |       c1       
----+----------------
  1 | Happy Birthday
  2 | happy birthday
(2 rows)
```

The backup set restored was created after our restore point. We should have 
used the `--set` option to specify which backup we want to restore.

-----

# Restore using a specific target time

Stop the PostgreSQL cluster and restore the backup using a specific target time:

```bash
$ sudo systemctl stop postgresql-12

$ sudo -iu postgres pgbackrest restore --stanza=happy_birthday --type=time --target="2020-03-02 09:05:50"
P00   INFO: restore backup set 20200302-090023F
...
P00   INFO: restore command end: completed successfully
```

Thanks to the target time, **pgBackRest** selected the good backup set.

Check the restore settings:

```bash
$ cat /var/lib/pgsql/12/data/postgresql.auto.conf
restore_command = 'pgbackrest --stanza=happy_birthday archive-get %f "%p"'
recovery_target_time = '2020-03-02 09:05:50'
```

The specified target is right before the second `INSERT`.

Start the cluster and check the data:

```bash
$ sudo systemctl start postgresql-12

$ sudo -iu postgres psql -c "SELECT * FROM t1;"
 id |       c1       
----+----------------
  1 | Happy Birthday
(1 row)
```

-----

# Conclusion

This new release brings a nice tweak to avoid confusion when `--set` is needed 
but not obvious.
