---
layout: post
title: check_pgbackrest 2.0 has been released
---

[check_pgbackrest](https://github.com/pgstef/check_pgbackrest) is designed to monitor **pgBackRest** backups, relying on the status information given by the [info](https://pgbackrest.org/command.html#command-info) command. 

![](../../../images/logo-horizontal.png){:width="800px"}

The biggest change in this new release is that the tool will now only support **pgBackRest 2.32** and above in order to only use its internal commands.
This remove Perl dependencies no-longer needed to reach repository hosts or S3 compatible object stores.
This also brings Azure compatible object stores support.
The `repo-*` arguments have then been deprecated.

[pgBackRest](http://pgbackrest.org/) **2.32** has been released this week and brings official support for the repository commands: `repo-ls` and `repo-get`.

Let's find out in this post how nice this feature is.

<!--MORE-->

-----

# Installation

On a fresh CentOS7 machine, let's quickly install PostgreSQL and **pgBackRest** to take a first backup locally.

Configure the PGDG yum repositories:

```bash
root# yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/\
pgdg-redhat-repo-latest.noarch.rpm
```

Then, install PostgreSQL: 

```bash
root# yum install -y postgresql13-server postgresql13-contrib
```

The **pgBackRest** package will require some dependencies located in the EPEL repository. If not already done on your
system, add it:

```bash
root# yum install -y epel-release
```

Install **pgBackRest** and check its version:

```bash
root# yum install -y pgbackrest
root# sudo -u postgres pgbackrest version
pgBackRest 2.32
```

Finally, create a basic PostgreSQL instance:

```bash
root# export PGSETUP_INITDB_OPTIONS="--data-checksums"
root# /usr/pgsql-13/bin/postgresql-13-setup initdb
root# systemctl enable postgresql-13
root# systemctl start postgresql-13
```

-----

# Configuration

We'll now prepare the configuration for our stanza called `mycluster`:

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
repo1-cipher-pass=si5J8aq9MGiMzNGghZqXbmwpVNEV5Sjc

[mycluster]
pg1-path=/var/lib/pgsql/13/data
```

The `base64` cipher-pass has been randomly generated with:

```bash
$ openssl rand -base64 24
si5J8aq9MGiMzNGghZqXbmwpVNEV5Sjc
```

Configure archiving in the `postgresql.conf` file:

```
listen_addresses = '*'
archive_mode = on
archive_command = 'pgbackrest --stanza=mycluster archive-push %p'
```

The PostgreSQL cluster must be restarted after making these changes and before performing a backup.

```bash
root# systemctl restart postgresql-13.service
```

Let's finally create the stanza and check the configuration:

```bash
postgres$ pgbackrest --stanza=mycluster stanza-create
...
P00   INFO: stanza-create for stanza 'mycluster' on repo1
P00   INFO: stanza-create command end: completed successfully

postgres$ pgbackrest --stanza=mycluster check
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

-----

# Perform a backup

Let's take our first backup :

```bash
postgres$ pgbackrest --stanza=mycluster --type=full backup
P00   INFO: backup command begin 2.32: ...
P00   INFO: execute non-exclusive pg_start_backup(): backup begins after the requested immediate checkpoint completes
P00   INFO: backup start archive = 00000001000000000000009F, lsn = 0/9F000028
P00   INFO: full backup size = 1.5GB
P00   INFO: execute non-exclusive pg_stop_backup() and wait for all WAL segments to archive
P00   INFO: backup stop archive = 00000001000000000000009F, lsn = 0/9F000100
P00   INFO: check archive for segment(s) 00000001000000000000009F:00000001000000000000009F
P00   INFO: new backup label = 20210210-161324F
P00   INFO: backup command end: completed successfully
P00   INFO: expire command begin 2.32: ...
P00   INFO: expire command end: completed successfully
```

The `info` command will give you some information about the backups and archives:

```bash
postgres$ pgbackrest --stanza=mycluster info
stanza: mycluster
    status: ok
    cipher: aes-256-cbc

    db (current)
        wal archive min/max (13): 00000001000000000000009F/00000001000000000000009F

        full backup: 20210210-161324F
            timestamp start/stop: 2021-02-10 16:13:24 / 2021-02-10 16:13:44
            wal start/stop: 00000001000000000000009F / 00000001000000000000009F
            database size: 1.5GB, database backup size: 1.5GB
            repo1: backup set size: 85MB, backup size: 85MB
```

-----

# Repository Commands

Now, we can look at those repository commands available with the latest release of **pgBackRest**.

Let's list the repository content:

```bash
postgres$ ls /var/lib/pgbackrest/archive/mycluster/13-1/0000000100000000/
00000001000000000000009F.00000028.backup  
00000001000000000000009F-32bbf21c264fbd9066ab43f284578a3b7ac3cf8e.gz
```

That's fine when you look at a local directory. But could become more difficult with remote storage, S3 buckets, etc.

The `repo-ls` command will help you here:

```bash
postgres$ pgbackrest help repo-ls
pgBackRest 2.32 - 'repo-ls' command help

List files in a repository.

Similar to the unix ls command but works on any supported repository type. This
command is primarily for administration, investigation, and testing. It is not
a required part of a normal pgBackRest setup.

The default text output prints one file name per line. JSON output is available
by specifying --output=json.

Command Options:

  --filter                         filter output with a regular expression
  --output                         output format [default=text]
  --recurse                        include all subpaths in output [default=n]
  --sort                           sort output ascending, descending, or none
                                   [default=asc]

postgres$ pgbackrest repo-ls archive/mycluster/13-1/0000000100000000
00000001000000000000009F-32bbf21c264fbd9066ab43f284578a3b7ac3cf8e.gz
00000001000000000000009F.00000028.backup
```

Then, we could want to check the content of `00000001000000000000009F.00000028.backup` but it wouldn't be easy since this file has been encrypted. That's where `repo-get` comes in:

```bash
postgres$ pgbackrest help repo-get
pgBackRest 2.32 - 'repo-get' command help

Get files from a repository.

Similar to the unix cat command but works on any supported repository type.
This command is primarily for administration, investigation, and testing. It is
not a required part of a normal pgBackRest setup.

If the repository is encrypted then repo-get will automatically decrypt the
file. Files are not automatically decompressed but the output can be piped
through the appropriate decompression command, e.g. gzip -d.

Command Options:

  --ignore-missing                 ignore missing source file [default=n]

postgres$ pgbackrest repo-get archive/mycluster/13-1/0000000100000000/00000001000000000000009F.00000028.backup
START WAL LOCATION: 0/9F000028 (file 00000001000000000000009F)
STOP WAL LOCATION: 0/9F000100 (file 00000001000000000000009F)
CHECKPOINT LOCATION: 0/9F000060
BACKUP METHOD: streamed
BACKUP FROM: master
START TIME: 2021-02-10 16:13:25 CET
LABEL: pgBackRest backup started at 2021-02-10 16:13:25.141193+01
START TIMELINE: 1
STOP TIME: 2021-02-10 16:13:44 CET
STOP TIMELINE: 1
```

It can also be very useful to check a WAL segment content:

```bash
postgres$ pgbackrest repo-get archive/mycluster/13-1/\
0000000100000000/00000001000000000000009F-32bbf21c264fbd9066ab43f284578a3b7ac3cf8e.gz |gunzip > /tmp/00000001000000000000009F
postgres$ /usr/pgsql-13/bin/pg_waldump /tmp/00000001000000000000009F
rmgr: Standby     len (rec/tot):     50/    50, tx:          0, lsn: 0/9F000028, prev 0/9E77AF50, 
    desc: RUNNING_XACTS nextXid 498 latestCompletedXid 497 oldestRunningXid 498
rmgr: XLOG        len (rec/tot):    114/   114, tx:          0, lsn: 0/9F000060, prev 0/9F000028, 
    desc: CHECKPOINT_ONLINE redo 0/9F000028; tli 1; prev tli 1; fpw true; xid 0:498; oid 24576; multi 1; offset 0; 
    oldest xid 479 in DB 1; oldest multi 1 in DB 1; oldest/newest commit timestamp xid: 0/0; oldest running xid 498; online
rmgr: XLOG        len (rec/tot):     34/    34, tx:          0, lsn: 0/9F0000D8, prev 0/9F000060, desc: BACKUP_END 0/9F000028
rmgr: XLOG        len (rec/tot):     24/    24, tx:          0, lsn: 0/9F000100, prev 0/9F0000D8, desc: SWITC
```

-----

# Impact on check_pgbackrest

With those 2 commands, **check_pgbackrest** doesn't need to reach directly the repository to fetch its content using overly-complicated Perl modules anymore.

As soon as the packages will be available in the PGDG repositories, it's easy to install:

```bash
root# yum install nagios-plugins-pgbackrest

root# /usr/lib64/nagios/plugins/check_pgbackrest --version
check_pgbackrest version 2.0, Perl 5.16.3

root# /usr/lib64/nagios/plugins/check_pgbackrest --list
List of available services:

    archives            Check WAL archives.
    check_pgb_version   Check the version of this check_pgbackrest script.
    retention           Check the retention policy.
```

The retention service hasn't changed much in this release. Here's a simple example:

```bash
root# /usr/lib64/nagios/plugins/check_pgbackrest \
         --service=retention --stanza=mycluster --output=human --retention-full=1
Service        : BACKUPS_RETENTION
Returns        : 0 (OK)
Message        : backups policy checks ok
Long message   : full=1
Long message   : diff=0
Long message   : incr=0
Long message   : latest=full,20210210-161324F
Long message   : latest_age=37m7s

root# /usr/lib64/nagios/plugins/check_pgbackrest \
        --service=retention --stanza=mycluster --output=human --retention-full=2
Service        : BACKUPS_RETENTION
Returns        : 2 (CRITICAL)
Message        : not enough full backups, 2 required
Long message   : full=1
Long message   : diff=0
Long message   : incr=0
Long message   : latest=full,20210210-161324F
Long message   : latest_age=37m3s
```

As mentioned, the biggest change comes for the archives service. It is now way simpler:

```bash
root# /usr/lib64/nagios/plugins/check_pgbackrest --service=archives --stanza=mycluster --output=human
Service        : WAL_ARCHIVES
Returns        : 0 (OK)
Message        : 1 WAL archived
Message        : latest archived since 39m37s
Long message   : latest_archive_age=39m37s
Long message   : num_archives=1
Long message   : archives_dir=archive/mycluster/13-1
Long message   : min_wal=00000001000000000000009F
Long message   : max_wal=00000001000000000000009F
Long message   : latest_archive=00000001000000000000009F
Long message   : latest_bck_archive_start=00000001000000000000009F
Long message   : latest_bck_type=full
Long message   : oldest_archive=00000001000000000000009F
Long message   : oldest_bck_archive_start=00000001000000000000009F
Long message   : oldest_bck_type=full
```

Let's test it inserting some data again:

```bash
postgres$ /usr/pgsql-13/bin/pgbench -i -s 100 test
```

Then, remove manually an archived WAL file:

```bash
postgres$ rm -f /var/lib/pgbackrest/archive/mycluster/13-1/0000000100000001/\
            000000010000000100000022-dbb15ee9b15c3021e1276a7a3fdd20e19c68bfb0.gz
```

The `info` command won't report it missing while **check_pgbackrest** should:

```bash
postgres$ pgbackrest info
stanza: mycluster
    status: ok
    cipher: aes-256-cbc

    db (current)
        wal archive min/max (13): 00000001000000000000009F/00000001000000010000003B

        full backup: 20210210-161324F
            timestamp start/stop: 2021-02-10 16:13:24 / 2021-02-10 16:13:44
            wal start/stop: 00000001000000000000009F / 00000001000000000000009F
            database size: 1.5GB, database backup size: 1.5GB
            repo1: backup set size: 85MB, backup size: 85MB

root#  /usr/lib64/nagios/plugins/check_pgbackrest --service=archives --stanza=mycluster --output=human
Service        : WAL_ARCHIVES
Returns        : 2 (CRITICAL)
Message        : wrong sequence, 1 missing file(s) (000000010000000100000022 / 000000010000000100000022)
Long message   : latest_archive_age=1m8s
Long message   : num_archives=156
Long message   : num_missing_archives=1
Long message   : oldest_missing_archive=000000010000000100000022
Long message   : latest_missing_archive=000000010000000100000022
Long message   : archives_dir=archive/mycluster/13-1
Long message   : min_wal=00000001000000000000009F
Long message   : max_wal=00000001000000010000003B
Long message   : latest_archive=00000001000000010000003B
Long message   : latest_bck_archive_start=00000001000000000000009F
Long message   : latest_bck_type=full
Long message   : oldest_archive=00000001000000000000009F
Long message   : oldest_bck_archive_start=00000001000000000000009F
Long message   : oldest_bck_type=full
```

-----

# Conclusion

As you can see, it's always important to watch and verify the consistency of our backups. **pgBackRest** is an awesome tool and will continue to receive new interesting features in the future.

This time, the new release allowed **check_pgbackrest** to get rid off unnecessary Perl modules and make it easier to use with the counterpart of only supporting **pgBackRest** above **2.32**.