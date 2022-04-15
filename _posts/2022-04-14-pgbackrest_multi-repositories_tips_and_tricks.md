---
layout: post
title: pgBackRest multi-repositories tips and tricks
date: 2022-04-15 09:00:00 +0200
---

Since April 2021 and the 2.33 release, pgBackRest allows using multiple repositories at the same time. This brings a lot of benefits like, for example, redundancy and the ability to define various retention policies.

I had the chance to talk about this feature recently at [pgDay Paris](../talks/en/20220324_pgDayParis_Using-multiple-backup-repositories-with-pgBackRest.pdf) to highlight the impact of this new feature on the existing pgBackRest commands.

A detailed example can also be found in an [EDB docs](https://www.enterprisedb.com/docs/supported-open-source/pgbackrest/08-multiple-repositories/) page I wrote last year when this feature was released.

In this post, we'll see the most frequent questions I get (in conferences or on community channels) and some tips and tricks.

<!--MORE-->

-----

# Impact on Postgres archiver

The first question I often get when speaking about this feature is the impact on Postres `archive_command`. Indeed, if one repository is failing, the `pgbackrest archive-push` command will fail and Postgres archiver will be blocked, piling up WAL segments ready to be archived.


Let's have an example with a NFS mount (`/mnt/demo-backups`) and a S3 bucket:

```ini
repo1-type=posix
repo1-path=/mnt/demo-backups
repo2-type=s3
repo2-s3-bucket=demo-bucket
repo2-s3-endpoint=s3.us-east-1.amazonaws.com
repo2-s3-key=accessKey1
repo2-s3-key-secret=verySecretKey1
repo2-s3-region=us-east-1
repo2-path=/demo-repo
```

If we screw up the `repo2` configuration, Postgres archiver will be blocked:

```bash
$ sudo -iu postgres ps -o pid,cmd fx |grep archiver
  40820  \_ postgres: main: archiver failed on 00000002000000000000002F
```

As usual, the best place to find out more details about why it is failing is inside Postgres logs:

```
ERROR: [104]: archive-push command encountered error(s):
       repo2: [FileMissingError] unable to load info file '.../archive.info' or '.../archive.info.copy':
       FileMissingError: unable to open missing file '.../archive.info' for read
       FileMissingError: unable to open missing file '.../archive.info.copy' for read
       HINT: archive.info cannot be opened but is required to push/get WAL segments.
       HINT: is archive_command configured correctly in postgresql.conf?
       HINT: has a stanza-create been performed?
       HINT: use --no-archive-check to disable archive checks during backup if you have an alternate archiving scheme.
... P00   INFO: archive-push command end: aborted with exception [104]
... [4082015] @ app= LOG:  archive command failed with exit code 104
... [4082016] @ app= DETAIL:  The failed archive command was: pgbackrest ... archive-push pg_wal/00000002000000000000002F
```

If we look inside the repositories, we'll see the WAL archive only in the first one:

```bash
$ pgbackrest --repo=1 repo-ls archive/stanza/14-1/0000000200000000/ |grep 00000002000000000000002F |wc -l
1
$ pgbackrest --repo=2 repo-ls archive/stanza/14-1/0000000200000000/ |grep 00000002000000000000002F |wc -l
0
```

Of course, there might be other WAL segments ready to be archived but not done yet because Postgres archiver is blocked:

```bash
$ ls data/pg_wal/archive_status/|grep ready
00000002000000000000002F.ready
000000020000000000000030.ready
000000020000000000000031.ready
$ pgbackrest --repo=1 repo-ls archive/stanza/14-1/0000000200000000/ |grep 000000020000000000000030 |wc -l
0
$ pgbackrest --repo=2 repo-ls archive/stanza/14-1/0000000200000000/ |grep 000000020000000000000030 |wc -l
0
```

Using some [archiving tricks](../2019/03/26/pgbackrest_archiving_tricks.html), we'll be able to bring fault-tolerance.

-----

## Asynchronous archiving

Postgres is only able to trigger the `archive_command` for one WAL segment at a time. Any error will prevent Postgres to remove/recycle the WAL file not archived. Furthermore, if the archiver process is blocked, the next WAL segments ready to be archived won't be triggered because Postgres will continuously try to archive the same failing WAL segment until it succeeds.

With `archive-async=y`, those ready archives (not triggered by Postgres yet) are pushed asynchronously using multiple-processes (partly defined by `process-max`) to working repositories!
Temporary data (acknowledgment files) will be stored into the `spool-path` so when Postgres triggers those WAL segments to be archived, the `archive_command` answer will be fasten a lot.

**Remark:** the spool path is intended to be located on a local Posix-compatible file-system, not a remote file-system such as NFS or CIFS.

Let's add some configuration to our current test setup:

```ini
archive-async=y
spool-path=/var/spool/pgbackrest
process-max=2
```

As soon as Postgres triggers another `archive_command` call, the WAL segment ready to be archived (but not requested by Postgres itself yet!) will make it to the working repository:

```bash
$ sudo -iu postgres ps -o pid,cmd fx |grep archiver
  40820  \_ postgres: main: archiver failed on 00000002000000000000002F
$ pgbackrest --repo=1 repo-ls archive/stanza/14-1/0000000200000000/ |grep 000000020000000000000030 |wc -l
1
$ pgbackrest --repo=2 repo-ls archive/stanza/14-1/0000000200000000/ |grep 000000020000000000000030 |wc -l
0
```

At this point, we have our WAL archives pushed to the working repository but the Postgres archiver is still blocked. WAL segments not archived to all repositories defined will then fill up the pg_wal directory and may cause Postgres to PANIC and stop if the disk space is full.

-----

## Archiving queue

To prevent the WAL space from filling up until Postgres stops completely, pgBackRest allows to define a maximum size of the archive queue: [`archive-push-queue-max`](https://pgbackrest.org/configuration.html#section-archive/option-archive-push-queue-max).

After the limit is reached, pgBackRest will notify Postgres that the WAL was successfully archived, but DROP IT! (A warning will be output to the Postgres log though.)
If this occurs, there will be **missing archives** in the repository and PITR will not be possible past that point (unless the same archive has been successfully pushed to another repository or until a new backup is taken).

Back to our example, we currently have 5 WAL segments (5 * 16Mo = 80Mo) ready to be archived to `repo2`:

```bash
$ ls data/pg_wal/archive_status/|grep ready
00000002000000000000002F.ready
000000020000000000000030.ready
000000020000000000000031.ready
000000020000000000000032.ready
000000020000000000000033.ready
```

For testing purposes only, let's set `archive-push-queue-max=64M` and see what happens. (**Remark:** don't use such a low value on production systems!)

```bash
$ ls data/pg_wal/archive_status/|grep ready |wc -l
0
$ sudo -iu postgres ps -o pid,cmd fx |grep archiver
  40820  \_ postgres: main: archiver last was 000000020000000000000033
```

The Postgres archiver process is now reporting success even though the WAL segments haven't been archived to the failing repository! We can notice it from the Postgres logs:

```
WARN: dropped WAL file '000000020000000000000033' because archive queue exceeded 64MB
```

This is a convenient way to prevent a disk space issue but it will generate missing archives. As usual, it is **always** very important to monitor archiving to ensure it continues working.

-----

# Using a dedicated backup server

To keep things (commands and configuration management) as simple as possible, it is best not to mix two very different use-cases: using a dedicated backup server (aka repo-host) and directly attached storage (NFS, S3, Azure,...). When possible, stick to only one mode.

If you want to use a dedicated backup server + a directly attached storage, I'd recommend the following two possibilities. Both will have one thing in common: the directly attached storage will be attached to the backup server, not to the Postgres node itself (that's also why it is called a repository host).

The first possibility is to use two backup servers: one with your local storage (or NFS mount point), one with your cloud storage (S3,...). The other (and easiest) one is to use the same backup servers but define the two repositories there.

To do that, split your repositories configuration to the backup server(s) you'll use:

```ini
repo1-type=posix
repo1-path=/mnt/demo-backups
```
```ini
repo2-type=s3
repo2-s3-bucket=demo-bucket
repo2-s3-endpoint=s3.us-east-1.amazonaws.com
repo2-s3-key=accessKey1
repo2-s3-key-secret=verySecretKey1
repo2-s3-region=us-east-1
repo2-path=/demo-repo
```

And then, update your Postgres node's configuration:

```ini
repo1-host=backup-server1
repo1-path=/mnt/demo-backups
repo2-host=backup-server1 or backup-server2
repo2-path=/demo-repo
```

**Remark:** unfortunately, repo-paths must match between the backup servers and the Postgres nodes configuration.

The archives will be pushed from the Postgres node (using `archive_command` and `pgbackrest archive-push`) as described earlier, providing fault-tolerance when used with asynchronous archiving.
The backups need to be scheduled individually for each repository. So if one repository is failing, the command triggered for the working repository should succeed anyway (IF the archiving system can successfully push archives to the working repository of course).

-----

# Conclusion

Combined with asynchronous archiving, the multi-repositories feature is really powerful even if it's not that easy to apprehend at first sight. The best advice, as usual, would be to try and test it yourself !
