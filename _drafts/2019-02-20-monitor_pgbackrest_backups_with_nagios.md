---
layout: post
title: Monitor pgBackRest backups with Nagios
draft: true
---

[pgBackRest](http://pgbackrest.org/) is a well-known powerful backup and 
restore tool.

Relying on the status information given by the "info" command, we've build a 
specific plugin for Nagios : [check_pgbackrest](https://github.com/dalibo/check_pgbackrest).

This post will help you discover this plugin and assume you already know 
pgBackRest and Nagios.

<!--MORE-->

-----

Let's assume we have a PostgreSQL cluster with pgBackRest working correctly.

Given this simple configuration:

```ini
[global]
repo1-path=/some_shared_space/
repo1-retention-full=2

[mystanza]
pg1-path=/var/lib/pgsql/11/data
```

Let's get the status of our backups with the `pgbackrest info` command:

```
stanza: mystanza
    status: ok
    cipher: none

    db (current)
        wal archive min/max (11-1): 00000001000000040000003C/000000010000000B0000004E

        full backup: 20190219-121527F
            timestamp start/stop: 2019-02-19 12:15:27 / 2019-02-19 12:18:15
            wal start/stop: 00000001000000040000003C / 000000010000000400000080
            database size: 3.0GB, backup size: 3.0GB
            repository size: 168.5MB, repository backup size: 168.5MB

        incr backup: 20190219-121527F_20190219-121815I
            timestamp start/stop: 2019-02-19 12:18:15 / 2019-02-19 12:20:38
            wal start/stop: 000000010000000400000082 / 0000000100000004000000B8
            database size: 3.0GB, backup size: 2.9GB
            repository size: 175.2MB, repository backup size: 171.6MB
            backup reference list: 20190219-121527F

        incr backup: 20190219-121527F_20190219-122039I
            timestamp start/stop: 2019-02-19 12:20:39 / 2019-02-19 12:22:55
            wal start/stop: 0000000100000004000000C1 / 0000000100000004000000F4
            database size: 3.0GB, backup size: 3.0GB
            repository size: 180.9MB, repository backup size: 177.3MB
            backup reference list: 20190219-121527F, 20190219-121527F_20190219-121815I

        full backup: 20190219-122255F
            timestamp start/stop: 2019-02-19 12:22:55 / 2019-02-19 12:25:47
            wal start/stop: 000000010000000500000000 / 00000001000000050000003D
            database size: 3.0GB, backup size: 3.0GB
            repository size: 186.5MB, repository backup size: 186.5MB

        incr backup: 20190219-122255F_20190219-122548I
            timestamp start/stop: 2019-02-19 12:25:48 / 2019-02-19 12:28:17
            wal start/stop: 000000010000000500000040 / 000000010000000500000077
            database size: 3GB, backup size: 3.0GB
            repository size: 192.3MB, repository backup size: 188.7MB
            backup reference list: 20190219-122255F

        incr backup: 20190219-122255F_20190219-122817I
            timestamp start/stop: 2019-02-19 12:28:17 / 2019-02-19 12:30:36
            wal start/stop: 00000001000000050000007F / 0000000100000005000000B1
            database size: 3GB, backup size: 3.0GB
            repository size: 197.2MB, repository backup size: 193.5MB
            backup reference list: 20190219-122255F
```

We can now use the check_pgbackrest Nagios plugin. See the `INSTALL.md` file
for the complete list of prerequisites.

```bash
$ sudo yum install perl-JSON epel-release perl-Net-SFTP-Foreign
```

To display "human readable" output, we'll use the `--format=human` argument.

-----

# [](#retention)Monitor the backup retention

The `retention` service will fail when the number of full backups is less than 
the `--retention-full` argument.

Example:

```bash
$ ./check_pgbackrest --service=retention --stanza=mystanza --retention-full=2 --format=human
Service        : BACKUPS_RETENTION
Returns        : 0 (OK)
Message        : backups policy checks ok
Long message   : full=2
Long message   : diff=0
Long message   : incr=4
Long message   : latest=incr,20190219-122255F_20190219-122817I
Long message   : latest_age=1h18m50s
```

```bash
$ ./check_pgbackrest --service=retention --stanza=mystanza --retention-full=3 --format=human
Service        : BACKUPS_RETENTION
Returns        : 2 (CRITICAL)
Message        : not enough full backups, 3 required
Long message   : full=2
Long message   : diff=0
Long message   : incr=4
Long message   : latest=incr,20190219-122255F_20190219-122817I
Long message   : latest_age=1h19m25s
```

It can also fail when the newest backup is older than the `--retention-age` 
argument.

The following units are accepted (not case sensitive): 
`s (second), m (minute), h (hour), d (day)`. 
You can use more than one unit per given value.

```bash
$ ./check_pgbackrest --service=retention --stanza=mystanza --retention-age=1h --format=human
Service        : BACKUPS_RETENTION
Returns        : 2 (CRITICAL)
Message        : backups are too old
Long message   : full=2
Long message   : diff=0
Long message   : incr=4
Long message   : latest=incr,20190219-122255F_20190219-122817I
Long message   : latest_age=1h19m56s
```

```bash
$ ./check_pgbackrest --service=retention --stanza=mystanza --retention-age=2h --format=human
Service        : BACKUPS_RETENTION
Returns        : 0 (OK)
Message        : backups policy checks ok
Long message   : full=2
Long message   : diff=0
Long message   : incr=4
Long message   : latest=incr,20190219-122255F_20190219-122817I
Long message   : latest_age=1h19m59s
```

Those 2 options can be used simultaneously:

```bash
$ ./check_pgbackrest --service=retention --stanza=mystanza --retention-age=2h --retention-full=2 
BACKUPS_RETENTION OK - backups policy checks ok | 
full=2 diff=0 incr=4 latest=incr,20190219-122255F_20190219-122817I latest_age=1h20m36s
```

This service works fine for local or remote backups since it only relies on 
the info command.

-----

# [](#archives)Monitor local WAL segments archives

The `archives` service checks if all archived WALs exist between the oldest and 
the latest WAL needed for the recovery.

This service requires the `--repo-path` argument to specify where the archived 
WALs are stored locally.

Archives must be compressed (.gz). If needed, use "compress-level=0" instead 
of "compress=n".

Use the `--wal-segsize` argument to set the WAL segment size if you don't use 
the default one.

The following units are accepted (not case sensitive):
`b (Byte), k (KB), m (MB), g (GB), t (TB), p (PB), e (EB) or Z (ZB)`. 
Only integers are accepted. Eg. `1.5MB` will be refused, use `1500kB`.

The factor between units is 1024 bytes. Eg. `1g = 1G = 1024*1024*1024`.

Example:

```bash
$ ./check_pgbackrest --service=archives --stanza=mystanza --repo-path="/some_shared_space/archive" --format=human
Service        : WAL_ARCHIVES
Returns        : 0 (OK)
Message        : 1811 WAL archived, latest archived since 41m48s
Long message   : latest_wal_age=41m48s
Long message   : num_archives=1811
Long message   : archives_dir=/some_shared_space/archive/mystanza/11-1
Long message   : oldest_archive=00000001000000040000003C-1937e658f8693e3949583d909456ef84398abd03.gz
Long message   : latest_archive=000000010000000B0000004E-2b9cc85b487a8e7b297148169018d46e6b7f1ed2.gz
```

-----

# [](#remote-archives)Monitor remote WAL segments archives

The `archives` service can also check remote archived WALs using SFTP with the 
`--repo-host` and `--repo-host-user` arguments.

As reminder, you have to setup a trusted SSH communication between the hosts.

We'll also here assume you have a working setup.

Here's a simple configuration:

- On the database server

```ini
[global]
repo1-host=remote
repo1-host-user=postgres

[mystanza]
pg1-path=/var/lib/pgsql/11/data
```

- On the backup server

```ini
[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=2

[mystanza]
pg1-path=/var/lib/pgsql/11/data
pg1-host=myserver
pg1-host-user=postgres
```

While the backups are taken from the remote server, the `pgbackrest info` 
command can be executed on both servers:

```
stanza: mystanza
    status: ok
    cipher: none

    db (current)
        wal archive min/max (11-1): 000000010000000B0000006B/000000010000000D00000078

        full backup: 20190219-143643F
            timestamp start/stop: 2019-02-19 14:36:43 / 2019-02-19 14:40:34
            wal start/stop: 000000010000000B0000006B / 000000010000000B000000A9
            database size: 3GB, backup size: 3GB
            repository size: 242MB, repository backup size: 242MB

        incr backup: 20190219-143643F_20190219-144035I
            timestamp start/stop: 2019-02-19 14:40:35 / 2019-02-19 14:43:23
            wal start/stop: 000000010000000B000000AD / 000000010000000B000000E2
            database size: 3GB, backup size: 3.0GB
            repository size: 246.3MB, repository backup size: 242.7MB
            backup reference list: 20190219-143643F

        incr backup: 20190219-143643F_20190219-144325I
            timestamp start/stop: 2019-02-19 14:43:25 / 2019-02-19 14:46:32
            wal start/stop: 000000010000000B000000EC / 000000010000000C00000022
            database size: 3GB, backup size: 3GB
            repository size: 250.5MB, repository backup size: 246.9MB
            backup reference list: 20190219-143643F, 20190219-143643F_20190219-144035I

        full backup: 20190219-144634F
            timestamp start/stop: 2019-02-19 14:46:34 / 2019-02-19 14:50:27
            wal start/stop: 000000010000000C0000002B / 000000010000000C00000069
            database size: 3GB, backup size: 3GB
            repository size: 253.7MB, repository backup size: 253.7MB

        incr backup: 20190219-144634F_20190219-145028I
            timestamp start/stop: 2019-02-19 14:50:28 / 2019-02-19 14:53:10
            wal start/stop: 000000010000000C0000006C / 000000010000000C000000A5
            database size: 3GB, backup size: 3GB
            repository size: 258.1MB, repository backup size: 254.5MB
            backup reference list: 20190219-144634F

        incr backup: 20190219-144634F_20190219-145311I
            timestamp start/stop: 2019-02-19 14:53:11 / 2019-02-19 14:56:26
            wal start/stop: 000000010000000C000000AB / 000000010000000C000000E3
            database size: 3GB, backup size: 3GB
            repository size: 262MB, repository backup size: 258.4MB
            backup reference list: 20190219-144634F, 20190219-144634F_20190219-145028I
```

Example from the database server: 

```bash
$ ./check_pgbackrest --service=archives --stanza=mystanza --repo-path="/var/lib/pgbackrest/archive" --repo-host=remote --format=human
Service        : WAL_ARCHIVES
Returns        : 0 (OK)
Message        : 526 WAL archived, latest archived since 41s
Long message   : latest_wal_age=41s
Long message   : num_archives=526
Long message   : archives_dir=/var/lib/pgbackrest/archive/mystanza/11-1
Long message   : min_wal=000000010000000B0000006B
Long message   : max_wal=000000010000000D00000078
Long message   : oldest_archive=000000010000000B0000006B-2609fef06d974e5918be051d8a409e7b8b50c818.gz
Long message   : latest_archive=000000010000000D00000078-f46f2ccdd176e4de9036d70fc51e1a7dd75aebbf.gz
```

From the backup server, use the "local" command:

```bash
$ ./check_pgbackrest --service=archives --stanza=mystanza --repo-path="/var/lib/pgbackrest/archive"
```

In case of missing archived WAL segment, you'll get an error:

```bash
$ ./check_pgbackrest --service=archives --stanza=mystanza --repo-path="/var/lib/pgbackrest/archive" --repo-host=remote
WAL_ARCHIVES CRITICAL - wrong sequence or missing file @ '000000010000000D00000037'
```

-----

# [](#remark)Remark

With pgBackRest 2.10, you might not get the `min_wal` and `max_wal` values:

```bash
Long message   : min_wal=000000010000000B0000006B
Long message   : max_wal=000000010000000D00000078
```

That behavior comes from the `pgbackrest info` command. Indeed, when specifying 
`--stanza=mystanza`, that information is missing:

```bash
wal archive min/max (11-1): none present
```

-----

# [](#tips)Tips

The `--command` argument allows to specify which pgBackRest executable file to 
use (default: "pgbackrest").

The `--config` parameter allows to provide a specific configuration file to 
pgBackRest.

If needed, some prefix command to execute the pgBackRest info command can be 
specified with the `--prefix` option (eg: "sudo -iu postgres").

-----

# [](#conclusion)Conclusion

[check_pgbackrest](https://github.com/dalibo/check_pgbackrest) is an open 
project, licensed under the PostgreSQL license. 

Any contribution to improve it is welcome.
