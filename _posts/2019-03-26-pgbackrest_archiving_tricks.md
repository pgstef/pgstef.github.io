---
layout: post
title: pgBackRest archiving tricks
draft: true
---

[pgBackRest](http://pgbackrest.org/) is a well-known powerful backup and 
restore tool. 

While the documentation describes all the parameters, it's not always that 
simple to imagine what you can really do with it.

In this post, I will introduce the asynchronous archiving and the possibility 
to avoid PostgreSQL to go down in case of archiving problems.

With its "info" command, for performance reasons, pgBackRest doesn't check 
that all the needed WAL segments are still present. 
[check_pgbackrest](https://github.com/dalibo/check_pgbackrest) is clearly 
built for that. The two tricks mentioned above can produce gaps in the 
archived WAL segments. The new 1.5 release of check_pgbackrest provides ways 
to handle that, we'll also see how.

<!--MORE-->

-----

# [](#installation)Installation

First of all, install PostgreSQL and pgBackRest packages directly from the 
PGDG yum repositories:

```bash
$ sudo yum install -y https://download.postgresql.org/pub/repos/yum/11/redhat/\
rhel-7-x86_64/pgdg-centos11-11-2.noarch.rpm
$ sudo yum install -y postgresql11-server postgresql11-contrib
$ sudo yum install -y pgbackrest
```

Check that pgBackRest is correctly installed:

```bash
$ pgbackrest
pgBackRest 2.11 - General help

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

Create a basic PostgreSQL cluster :

```bash
$ sudo /usr/pgsql-11/bin/postgresql-11-setup initdb
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
archive-async=y
archive-push-queue-max=100MB
spool-path=/var/spool/pgbackrest

[some_cool_stanza_name]
pg1-path=/var/lib/pgsql/11/data
```

Make sure that the postgres user can write in `/var/lib/pgbackrest` and in 
`/var/spool/pgbackrest`.

Configure archiving in the `postgresql.conf` file:

```
archive_mode = on
archive_command = 'pgbackrest --stanza=some_cool_stanza_name archive-push %p'
```

Start the PostgreSQL cluster:

```bash
$ sudo systemctl start postgresql-11
```

Create the stanza and check the configuration:

```bash
$ sudo -iu postgres pgbackrest --stanza=some_cool_stanza_name stanza-create
P00   INFO: stanza-create command end: completed successfully

$ sudo -iu postgres pgbackrest --stanza=some_cool_stanza_name check
P00   INFO: WAL segment 000000010000000000000001 successfully stored in the 
    archive at '/var/lib/pgbackrest/archive/some_cool_stanza_name/
    11-1/0000000100000000/
    000000010000000000000001-03a91d4d64251a54cf9d48ed59382d3cce3c7652.gz'
P00   INFO: check command end: completed successfully 
```

Let's finally take our first backup:

```bash
$ sudo -iu postgres pgbackrest --stanza=some_cool_stanza_name --type=full backup
...
P00   INFO: new backup label = 20190325-142918F
P00   INFO: backup command end: completed successfully
...
```

-----

# []pgBackRest configuration explanations

What the documentation says:

-----

## [](#archive-async)archive-async

Push/get WAL segments asynchronously.

Enables asynchronous operation for the `archive-push` and `archive-get` 
commands.

Asynchronous operation is more efficient because it can reuse connections and 
take advantage of parallelism.

-----

## [](#archive-push-queue-max)archive-push-queue-max

Maximum size of the PostgreSQL archive queue.

After the limit is reached, the following will happen:
  * pgBackRest will notify PostgreSQL that the WAL was successfully archived, then DROP IT.
  * A warning will be output to the PostgreSQL log.

If this occurs then the archive log stream will be interrupted and PITR will 
not be possible past that point. **A new backup will be required to regain 
full restore capability**.

In asynchronous mode the entire queue will be dropped to prevent spurts of 
WAL getting through before the queue limit is exceeded again.

The purpose of this feature is to prevent the log volume from filling up at 
which point PostgreSQL will stop completely. 

*Better to lose the backup than have PostgreSQL go down.*

Don't use this feature if you want to rely entirely on your backups!

-----

## [](#spool-path)spool-path

This path is used to store data for the asynchronous `archive-push` and 
`archive-get` command.

The asynchronous `archive-push` command writes acknowledgments into the spool 
path when it has successfully stored WAL in the archive (and errors on 
failure) so the foreground process can quickly notify PostgreSQL. 

-----

# []Test the archiving process

Right after the backup, let's see what's in the spool directory:

```bash
$ ls /var/spool/pgbackrest/archive/some_cool_stanza_name/out/
000000010000000000000003.00000028.backup.ok
000000010000000000000003.ok
```

Generate a small database change and switch WAL:

```bash
$ sudo -iu postgres psql -c "DROP TABLE IF EXISTS my_table; CREATE TABLE my_table(id int); SELECT pg_switch_wal();"
NOTICE:  table "my_table" does not exist, skipping
 pg_switch_wal 
---------------
 0/4016850
(1 row)
```

Check the spool directory again:

```bash
$ ls -l /var/spool/pgbackrest/archive/some_cool_stanza_name/out/
000000010000000000000004.ok
```

What's in the archives directory?

```bash
$ ls /var/lib/pgbackrest/archive/some_cool_stanza_name/11-1/0000000100000000/
000000010000000000000003.00000028.backup
000000010000000000000003-5050f0829090a98c5f92ff112417a2bf6c115ffa.gz
000000010000000000000004-3f9de64182e110ddcfe34d1191ad71c90f4fef3e.gz
```

-----

## []Break it!

```bash
$ sudo chmod -R 500 /var/lib/pgbackrest/archive/some_cool_stanza_name/11-1/
```

Generate a small database change and switch WAL:

```bash
$ sudo -iu postgres psql -c "DROP TABLE IF EXISTS my_table; CREATE TABLE my_table(id int); SELECT pg_switch_wal();"
 pg_switch_wal 
---------------
 0/501BF88
(1 row)
```

Check the archiving process:

```bash
$ ls /var/lib/pgsql/11/data/pg_wal/archive_status/
000000010000000000000003.00000028.backup.done
000000010000000000000005.ready

$ ls /var/spool/pgbackrest/archive/some_cool_stanza_name/out/
000000010000000000000005.error
```

By default, a WAL segment is 16MB. We configured `archive-push-queue-max` to 
100MB, so approximatively 6 archived WAL segments. 

What happens after the seventh fail?

Generate a small database change and switch WAL 5 more times with the same 
command as above.

```bash
$ ls /var/spool/pgbackrest/archive/some_cool_stanza_name/out/
000000010000000000000005.error
000000010000000000000006.error
000000010000000000000007.error
000000010000000000000008.error
000000010000000000000009.error
00000001000000000000000A.error

$ ps -ef |grep postgres |grep archiver
00:00:00 postgres: archiver   failed on 000000010000000000000005
```

The archiver process is still blocked on the first fail.

Generate the seventh fail:

```bash
$ sudo -iu postgres psql -c "DROP TABLE IF EXISTS my_table; CREATE TABLE my_table(id int); SELECT pg_switch_wal();"
 pg_switch_wal 
---------------
 0/B0025B8
(1 row)

$ ls /var/spool/pgbackrest/archive/some_cool_stanza_name/out/
000000010000000000000005.ok
000000010000000000000006.ok
000000010000000000000007.ok
000000010000000000000008.ok
000000010000000000000009.ok
00000001000000000000000A.ok
00000001000000000000000B.ok

$ ps -ef |grep postgres |grep archiver
00:00:00 postgres: archiver   last was 00000001000000000000000B
```

The archiver isn't failing anymore **BUT** there's no WAL archived either:

```bash
$ sudo -iu postgres pgbackrest info --stanza=some_cool_stanza_name
stanza: some_cool_stanza_name
    status: ok
    cipher: none

    db (current)
        wal archive min/max (11-1): 000000010000000000000003/000000010000000000000004

        full backup: 20190325-142918F
            timestamp start/stop: 2019-03-25 14:29:18 / 2019-03-25 14:29:28
            wal start/stop: 000000010000000000000003 / 000000010000000000000003
            database size: 23.5MB, backup size: 23.5MB
            repository size: 2.8MB, repository backup size: 2.8MB
```

-----

## []Repair it

```bash
$ sudo chmod -R 750 /var/lib/pgbackrest/archive/some_cool_stanza_name/11-1/
```

Generate a small database change, switch WAL and check the archiving process:

```bash
$ sudo -iu postgres psql -c "DROP TABLE IF EXISTS my_table; CREATE TABLE my_table(id int); SELECT pg_switch_wal();"
 pg_switch_wal 
---------------
 0/C0194A8
(1 row)

$ ls /var/spool/pgbackrest/archive/some_cool_stanza_name/out/ 
00000001000000000000000C.ok

$ sudo -iu postgres pgbackrest info --stanza=some_cool_stanza_name
stanza: some_cool_stanza_name
    status: ok
    cipher: none

    db (current)
        wal archive min/max (11-1): 000000010000000000000003/00000001000000000000000C

        full backup: 20190325-142918F
            timestamp start/stop: 2019-03-25 14:29:18 / 2019-03-25 14:29:28
            wal start/stop: 000000010000000000000003 / 000000010000000000000003
            database size: 23.5MB, backup size: 23.5MB
            repository size: 2.8MB, repository backup size: 2.8MB

$ ls /var/lib/pgbackrest/archive/some_cool_stanza_name/11-1/0000000100000000/
000000010000000000000003.00000028.backup
000000010000000000000003-5050f0829090a98c5f92ff112417a2bf6c115ffa.gz
000000010000000000000004-3f9de64182e110ddcfe34d1191ad71c90f4fef3e.gz
00000001000000000000000C-c90e3f9fbac504f51f44e1446c653d8a124dbd86.gz
```

Archiving is working but there's missing archives and **pgBackRest doesn't 
see it**.

The gap is here generated by the `archive-push-queue-max` but you could also 
have a gap simply due to asynchronous archiving with `process-max` greater 
than 1.

-----

# [](#check_pgbackrest)check_pgbackrest 1.5

The new 1.5 release, offers some interesting changes:
  * Add `--debug` option to print some debug messages.
  * Add `ignore-archived-since` argument to ignore the archived WALs since the 
  provided interval.
  * Add `--latest-archive-age-alert` to define the max age of the latest 
  archived WAL before raising a critical alert.

Download check_pgbackrest:

```bash
$ sudo yum install -y perl-JSON
$ sudo -iu postgres
$ wget https://raw.githubusercontent.com/dalibo/check_pgbackrest/REL1_5/check_pgbackrest
$ chmod +x check_pgbackrest
```

This installation procedure above is just a simple example.

Now, check the archives chain to know if there's something missing:

```bash
$ ./check_pgbackrest --stanza=some_cool_stanza_name --service=archives 
                     --repo-path=/var/lib/pgbackrest/archive --format=human
Service        : WAL_ARCHIVES
Returns        : 2 (CRITICAL)
Message        : wrong sequence or missing file @ '000000010000000000000005'
Long message   : latest_archive_age=9m54s
Long message   : num_archives=3
Long message   : archives_dir=/var/lib/pgbackrest/archive/some_cool_stanza_name/11-1
Long message   : min_wal=000000010000000000000003
Long message   : max_wal=00000001000000000000000C
Long message   : oldest_archive=000000010000000000000003-5050f0829090a98c5f92ff112417a2bf6c115ffa.gz
Long message   : latest_archive=00000001000000000000000C-c90e3f9fbac504f51f44e1446c653d8a124dbd86.gz
```

Let's ignore the latest archive producing the gap:

```bash
$ ./check_pgbackrest --stanza=some_cool_stanza_name --service=archives 
                     --repo-path=/var/lib/pgbackrest/archive --format=human 
                     --debug --ignore-archived-since=15m
DEBUG: file 000000010000000000000003-5050f0829090a98c5f92ff112417a2bf6c115ffa.gz as interval since epoch : 36m52s
DEBUG: file 000000010000000000000004-3f9de64182e110ddcfe34d1191ad71c90f4fef3e.gz as interval since epoch : 33m58s
DEBUG: file 00000001000000000000000C-c90e3f9fbac504f51f44e1446c653d8a124dbd86.gz as interval since epoch : 11m45s
DEBUG: max_wal changed to 000000010000000000000004
DEBUG: checking WAL 000000010000000000000003-5050f0829090a98c5f92ff112417a2bf6c115ffa.gz
DEBUG: checking WAL 000000010000000000000004-3f9de64182e110ddcfe34d1191ad71c90f4fef3e.gz
Service        : WAL_ARCHIVES
Returns        : 0 (OK)
Message        : 2 WAL archived, latest archived since 33m58s
Long message   : latest_archive_age=33m58s
Long message   : num_archives=2
Long message   : archives_dir=/var/lib/pgbackrest/archive/some_cool_stanza_name/11-1
Long message   : min_wal=000000010000000000000003
Long message   : max_wal=000000010000000000000004
Long message   : oldest_archive=000000010000000000000003-5050f0829090a98c5f92ff112417a2bf6c115ffa.gz
Long message   : latest_archive=000000010000000000000004-3f9de64182e110ddcfe34d1191ad71c90f4fef3e.gz
```

You also might want to receive an alert if the latest archive is too old:

```bash
$ ./check_pgbackrest --stanza=some_cool_stanza_name --service=archives 
                     --repo-path=/var/lib/pgbackrest/archive --format=human 
                     --ignore-archived-since=20m --latest-archive-age-alert=10m
Service        : WAL_ARCHIVES
Returns        : 2 (CRITICAL)
Message        : latest_archive_age (39m16s) exceeded
Long message   : latest_archive_age=39m16s
Long message   : num_archives=2
Long message   : archives_dir=/var/lib/pgbackrest/archive/some_cool_stanza_name/11-1
Long message   : min_wal=000000010000000000000003
Long message   : max_wal=000000010000000000000004
Long message   : oldest_archive=000000010000000000000003-5050f0829090a98c5f92ff112417a2bf6c115ffa.gz
Long message   : latest_archive=000000010000000000000004-3f9de64182e110ddcfe34d1191ad71c90f4fef3e.gz
```

The 2 options are here combined to avoid the alert on the missing archived 
WAL segments.

-----

# [](#conclusion)Conclusion

pgBackRest offers a lot of possibilities but, mainly for performance reasons, 
doesn't check the archives consistency. 

Combine it with, for example, a good monitoring system and the 
check_pgbackrest plugin for more safety.