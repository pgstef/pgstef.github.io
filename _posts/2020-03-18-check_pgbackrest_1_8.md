---
layout: post
title: check_pgbackrest 1.8 has been released
draft: true
---

[check_pgbackrest](https://labs.dalibo.com/check_pgbackrest) is designed to 
monitor `pgBackRest` backups, relying on the status information given by the 
[info](https://pgbackrest.org/command.html#command-info) command. 

![](../../../images/logo-horizontal.png){:width="800px"}

The changes in this new release are:
* missing archives output: the complete list is now only shown in `--debug` mode;
* new `--list-archives` argument to print the list of all the archived WAL segments found.

<!--MORE-->

-----

# Missing archives

Let's use the 1.7 release and see the basic output:

```bash
$ /usr/lib64/nagios/plugins/check_pgbackrest --version
check_pgbackrest version 1.7, Perl 5.16.3

$ /usr/lib64/nagios/plugins/check_pgbackrest --stanza=my_stanza 
	--service=archives --repo-path=/var/lib/pgbackrest/archive
WAL_ARCHIVES OK - 24 WAL archived, latest archived since 2m27s | 
	latest_archive_age=147s num_archives=24

$ /usr/lib64/nagios/plugins/check_pgbackrest --stanza=my_stanza 
	--service=archives --repo-path=/var/lib/pgbackrest/archive --output=human
Service        : WAL_ARCHIVES
Returns        : 0 (OK)
Message        : 24 WAL archived
Message        : latest archived since 3m1s
Long message   : latest_archive_age=3m1s
Long message   : num_archives=24
Long message   : archives_dir=/var/lib/pgbackrest/archive/my_stanza/12-1
Long message   : min_wal=000000010000000000000003
Long message   : max_wal=00000001000000000000001A
Long message   : oldest_archive=000000010000000000000003
Long message   : latest_archive=00000001000000000000001A
Long message   : latest_bck_archive_start=000000010000000000000005
Long message   : latest_bck_type=incr
```

And now, with some missing archives:

```bash
$ /usr/lib64/nagios/plugins/check_pgbackrest --stanza=my_stanza 
	--service=archives --repo-path=/var/lib/pgbackrest/archive
WAL_ARCHIVES CRITICAL - 
	wrong sequence or missing file @ '000000010000000000000010', 
	wrong sequence or missing file @ '000000010000000000000011', 
	wrong sequence or missing file @ '000000010000000000000012', 
	wrong sequence or missing file @ '000000010000000000000013' | 
	latest_archive_age=257s num_archives=20

$ /usr/lib64/nagios/plugins/check_pgbackrest --stanza=my_stanza 
	--service=archives --repo-path=/var/lib/pgbackrest/archive --output=human
Service        : WAL_ARCHIVES
Returns        : 2 (CRITICAL)
Message        : wrong sequence or missing file @ '000000010000000000000010'
Message        : wrong sequence or missing file @ '000000010000000000000011'
Message        : wrong sequence or missing file @ '000000010000000000000012'
Message        : wrong sequence or missing file @ '000000010000000000000013'
Long message   : latest_archive_age=4m26s
Long message   : num_archives=20
Long message   : archives_dir=/var/lib/pgbackrest/archive/my_stanza/12-1
Long message   : min_wal=000000010000000000000003
Long message   : max_wal=00000001000000000000001A
Long message   : oldest_archive=000000010000000000000003
Long message   : latest_archive=00000001000000000000001A
Long message   : latest_bck_archive_start=000000010000000000000005
Long message   : latest_bck_type=incr
```

The returned message can be very long in case of lots of missing archives... 
That's why it has been changed in the new release.

-----

# 1.8 release

Run the same commands with 1.8 release:

```bash
$ /usr/lib64/nagios/plugins/check_pgbackrest --version
check_pgbackrest version 1.8, Perl 5.16.3

$ /usr/lib64/nagios/plugins/check_pgbackrest --stanza=my_stanza 
	--service=archives --repo-path=/var/lib/pgbackrest/archive
WAL_ARCHIVES CRITICAL - 
	wrong sequence, 4 missing file(s) (000000010000000000000010 / 
		000000010000000000000013) | 
	latest_archive_age=1987s 
	num_archives=20 
	num_missing_archives=4 
	oldest_missing_archive=000000010000000000000010 
	latest_missing_archive=000000010000000000000013

$ /usr/lib64/nagios/plugins/check_pgbackrest --stanza=my_stanza 
	--service=archives --repo-path=/var/lib/pgbackrest/archive --output=human
Service        : WAL_ARCHIVES
Returns        : 2 (CRITICAL)
Message        : wrong sequence, 4 missing file(s) (000000010000000000000010 / 
	000000010000000000000013)
Long message   : latest_archive_age=33m37s
Long message   : num_archives=20
Long message   : num_missing_archives=4
Long message   : oldest_missing_archive=000000010000000000000010
Long message   : latest_missing_archive=000000010000000000000013
Long message   : archives_dir=/var/lib/pgbackrest/archive/my_stanza/12-1
Long message   : min_wal=000000010000000000000003
Long message   : max_wal=00000001000000000000001A
Long message   : oldest_archive=000000010000000000000003
Long message   : latest_archive=00000001000000000000001A
Long message   : latest_bck_archive_start=000000010000000000000005
Long message   : latest_bck_type=incr
```

In the nagios-style output, the returned message has been simplified while new 
information has been added in the _performance data_ part.

The complete list of missing archives can now be found in the `--debug` output:

```bash
$ /usr/lib64/nagios/plugins/check_pgbackrest --stanza=my_stanza 
	--service=archives --repo-path=/var/lib/pgbackrest/archive --output=human 
	--debug
...
DEBUG: missing 000000010000000000000010
DEBUG: missing 000000010000000000000011
DEBUG: missing 000000010000000000000012
DEBUG: missing 000000010000000000000013
Service        : WAL_ARCHIVES
Returns        : 2 (CRITICAL)
Message        : wrong sequence, 4 missing file(s) (000000010000000000000010 / 
	000000010000000000000013)
Long message   : latest_archive_age=33m52s
Long message   : num_archives=20
Long message   : num_missing_archives=4
Long message   : oldest_missing_archive=000000010000000000000010
Long message   : latest_missing_archive=000000010000000000000013
Long message   : archives_dir=/var/lib/pgbackrest/archive/my_stanza/12-1
Long message   : min_wal=000000010000000000000003
Long message   : max_wal=00000001000000000000001A
Long message   : oldest_archive=000000010000000000000003
Long message   : latest_archive=00000001000000000000001A
Long message   : latest_bck_archive_start=000000010000000000000005
Long message   : latest_bck_type=incr
```

-----

# New `--list-archives` argument

In addition to the missing archives, the debug mode can now also output all 
the archives found with the new `--list-archives` argument:

```bash
$ /usr/lib64/nagios/plugins/check_pgbackrest --stanza=my_stanza 
	--service=archives --repo-path=/var/lib/pgbackrest/archive --output=human 
	--debug --list-archives
...
DEBUG: missing 000000010000000000000010
DEBUG: missing 000000010000000000000011
DEBUG: missing 000000010000000000000012
DEBUG: missing 000000010000000000000013
DEBUG: found 000000010000000000000003
DEBUG: found 000000010000000000000004
DEBUG: found 000000010000000000000005
DEBUG: found 000000010000000000000006
DEBUG: found 000000010000000000000007
DEBUG: found 000000010000000000000008
DEBUG: found 000000010000000000000009
DEBUG: found 00000001000000000000000A
DEBUG: found 00000001000000000000000B
DEBUG: found 00000001000000000000000C
DEBUG: found 00000001000000000000000D
DEBUG: found 00000001000000000000000E
DEBUG: found 00000001000000000000000F
DEBUG: found 000000010000000000000014
DEBUG: found 000000010000000000000015
DEBUG: found 000000010000000000000016
DEBUG: found 000000010000000000000017
DEBUG: found 000000010000000000000018
DEBUG: found 000000010000000000000019
DEBUG: found 00000001000000000000001A
Service        : WAL_ARCHIVES
Returns        : 2 (CRITICAL)
Message        : wrong sequence, 4 missing file(s) (000000010000000000000010 / 
	000000010000000000000013)
Long message   : latest_archive_age=34m15s
Long message   : num_archives=20
Long message   : num_missing_archives=4
Long message   : oldest_missing_archive=000000010000000000000010
Long message   : latest_missing_archive=000000010000000000000013
Long message   : archives_dir=/var/lib/pgbackrest/archive/my_stanza/12-1
Long message   : min_wal=000000010000000000000003
Long message   : max_wal=00000001000000000000001A
Long message   : oldest_archive=000000010000000000000003
Long message   : latest_archive=00000001000000000000001A
Long message   : latest_bck_archive_start=000000010000000000000005
Long message   : latest_bck_type=incr
```

-----

# [](#conclusion)Conclusion

[check_pgbackrest](https://github.com/dalibo/check_pgbackrest) is an open 
project, licensed under the PostgreSQL license. 

Any contribution to improve it is welcome.
