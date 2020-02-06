---
layout: post
title: Monitor pgBackRest backups with check_pgbackrest 1.7
draft: true
---

[check_pgbackrest](https://labs.dalibo.com/check_pgbackrest) is designed to 
monitor `pgBackRest` backups, relying on the status information given by the 
`info` command. 

![A pgBackRest backup check plugin for Nagios](../../../images/logo-horizontal.png =800x)

The main features are:
* check WAL archives consistency;
* check the retention policy;
* check its own version;
* multiple output format: human, json and nagios.

<!--MORE-->

-----

# [](#installation)Installation

The RPM file made it's way to the PGDG Yum repository! Special thanks to 
[Devrim](https://twitter.com/DevrimGunduz) for that!

It can also be found in the Dalibo Labs Yum repository.

1. Install the [Official PostgreSQL yum repo package](https://yum.postgresql.org/repopackages.php) or the [Dalibo Labs yum repo package](https://yum.dalibo.org/labs)

2. `yum install nagios-plugins-pgbackrest`

Remark: `epel-release` also needs to be installed first.

-----

# [](#retention)Monitor the backup retention

The service fails when
  * the number of full backups is less than `--retention-full`;
  * the newest backup is older than `--retention-age`;
  * the newest full backup is older than `--retention-age-to-full`.

```bash
$ check_pgbackrest --stanza=my_stanza 
  --service=retention --retention-full=1 --output=human
  --retention-age=24h --retention-age-to-full=7d

Service        : BACKUPS_RETENTION
Returns        : 0 (OK)
Message        : backups policy checks ok
Long message   : full=1
Long message   : diff=1
Long message   : incr=1
Long message   : latest=incr,20200131-150158F_20200131-150410I
Long message   : latest_age=2m47s
Long message   : latest_full=20200131-150158F
Long message   : latest_full_age=5m
```

-----

# [](#archives)Monitor WAL segments archives

The `pgbackrest info` command
  * shows the oldest (min) archive and the most recent one (max);
  * doesn't check if all the archives in between are really on the disk;
  * doesn't give the age of the most recent archive (until 2.21);
  * ...

The `archives` service checks if all archived WALs exist between the oldest and 
the latest WAL needed for the recovery.

-----

## Local storage

This service requires the `--repo-path` argument to specify where the archived 
WALs are stored.

Archives must be compressed (.gz). If needed, use "compress-level=0" instead 
of "compress=n".

Use the `--wal-segsize` argument to set the WAL segment size if you don't use 
the default one.

```bash
$ check_pgbackrest --stanza=my_stanza
  --service=archives --repo-path=/var/lib/pgbackrest/archive --output=human
Service        : WAL_ARCHIVES
Returns        : 0 (OK)
Message        : 81 WAL archived
Message        : latest archived since 1m59s
Long message   : latest_archive_age=1m59s
Long message   : num_archives=81
Long message   : archives_dir=/var/lib/pgbackrest/archive/my_stanza/12-1
Long message   : min_wal=000000010000000000000003
Long message   : max_wal=000000010000000000000053
Long message   : oldest_archive=000000010000000000000003
Long message   : latest_archive=000000010000000000000053
Long message   : latest_bck_archive_start=000000010000000000000007
Long message   : latest_bck_type=incr
```

-----

## Remote storage

If the archives are pushed to another server, use the `--repo-host` and 
`--repo-host-user` arguments :

```bash
$ check_pgbackrest --stanza=my_stanza 
  --service=archives --repo-path=/var/lib/pgbackrest/archive 
  --repo-host="backup-srv" --repo-host-user=postgres

WAL_ARCHIVES OK - 4 WAL archived, latest archived since 25m30s | 
  latest_archive_age=25m30s num_archives=4
```

-----

## S3 storage

The service can use the Amazon S3 API by getting `repo1-s3-key` and `repo1-s3-key-secret` parameters directly from the `pgBackRest` configuration file.

Use the `--repo-s3` argument to turn on that behavior:

```bash
$ check_pgbackrest --stanza=my_stanza 
  --service=archives --repo-path=/repo1/archive 
  --repo-s3 --repo-s3-over-http

WAL_ARCHIVES OK - 4 WAL archived, latest archived since 1m7s | 
  latest_archive_age=1m7s num_archives=4
```

-----

## Go further

Use the `--ignore-archived-before` argument to ignore the archived WALs 
generated before the provided interval. Used to only check the latest archives.

Use the `--ignore-archived-after` argument to ignore the archived WALs 
generated after the provided interval. Used under heavy archiving load.

The `--latest-archive-age-alert` argument defines the max age of the latest 
archived WAL as an interval before raising a critical alert.

-----

# [](#vagrant) Tests using Vagrant

Tests scenarios have been prepared using CentOS 7 boxes and `libvirt` vm provider:

```bash
[check_pgbackrest]$ ls test
Makefile  perf  provision  README.md  regress  ssh  Vagrantfile
```

## 1. pgBackRest configured to backup and archive on a CIFS mount

  * `icinga-srv` executes check_pgbackrest by ssh with Icinga 2;
  * `pgsql-srv` hosting a pgsql cluster with pgBackRest installed;
  * `backup-srv` hosting the CIFS share.

Backups and archiving are done locally on `pgsql-srv` on the CIFS mount point.

## 2. pgBackRest configured to backup and archive remotely

  * `icinga-srv` executes check_pgbackrest by ssh with Icinga 2;
  * `pgsql-srv` hosting a pgsql cluster with pgBackRest installed;
  * `backup-srv` hosting the pgBackRest backups and archives.

Backups of `pgsql-srv` are taken from `backup-srv`. 
Archives are pushed from `pgsql-srv` to `backup-srv`.
Checks (retention and archives) are done both locally (on `backup-srv`) and 
remotely (on `pgsql-srv`). Checks are performed from `icinga-srv` by ssh.
pgBackRest backups are use to build a Streaming Replication with `backup-srv` 
as standby server.

## 3. pgBackRest configured to backup and archive to a MinIO S3 bucket

  * `icinga-srv` executes check_pgbackrest by ssh with Icinga 2;
  * `pgsql-srv` hosting a pgsql cluster with pgBackRest installed;
  * `backup-srv` hosting the MinIO server.

-----

# [](#evolution)Evolution

The first evolution I'd like to implement would be to use the `pgbackrest ls` 
command to get the archives list. Indeed, `mtime` property for archives is 
available since `pgBackRest` 2.21. At the moment, we'd still need a `cat` 
command for the `.history` files.

```bash
$ pgbackrest help ls
pgBackRest 2.23 - 'ls' command help

List paths/files in the repository.

This is intended to be a general purpose list function, but for now it only
works on the repository.

Command Options:

  --filter                         filter output with a regular expression
  --output                         output format [default=text]
  --recurse                        include all subpaths in output [default=n]
  --sort                           sort output ascending, descending, or none
                                   [default=asc]
```

Another possible evolution would be the Debian support with specific test cases 
(using Vagrant) and a `.deb` package.

If you have any idea to improve the tool, please, share it! :-)

-----

# [](#conclusion)Conclusion

[check_pgbackrest](https://github.com/dalibo/check_pgbackrest) is an open 
project, licensed under the PostgreSQL license. 

Any contribution to improve it is welcome.
