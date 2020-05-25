---
layout: post
title: "pgBackRest preview - A tour of retention policy options"
---

[pgBackRest](http://pgbackrest.org/) is a well-known powerful backup and 
restore tool. Old backups and archives are removed by the `expire` command 
based upon the defined retention policy.

Since the latest version published last month, new features regarding retention 
have been committed. We'll here first overview those changes and then make a 
tour of the retention policy options that should be available in the next release.

<!--MORE-->

-----

# New features

## Time-based retention

[Add time-based retention for full backups:](https://github.com/pgbackrest/pgbackrest/commit/cdebfb09)

```
The --repo-retention-full-type option allows retention of full backups based 
on a time period, specified in days.

The new option will default to 'count' and therefore will not affect current 
installations. Setting repo-retention-full-type to 'time' will allow the user 
to use a time period, in days, to indicate full backup retention. Using this 
method, a full backup can be expired only if the time the backup completed is 
older than the number of days set with repo-retention-full (calculated from 
the moment the 'expire' command is run) and at least one full backup meets the 
retention period. If archive retention has not been configured, then the default 
settings will expire archives that are prior to the oldest retained full backup. 

For example, if there are three full backups ending in times that are 25 days 
old (F1), 20 days old (F2) and 10 days old (F3), then if the full retention 
period is 15 days, then only F1 will be expired; F2 will be retained because 
F1 is not at least 15 days old.
```

This possibility opens a lot of opportunities. However, it's not so obvious 
that **at least one full backup** over the time-period defined will be kept.

---

## Expire a specific backup set

[Add `--set` option to the expire command:](https://github.com/pgbackrest/pgbackrest/commit/1c1a7104)

```
The specified backup set (i.e. the backup label provided and all of its 
dependent backups, if any) will be expired regardless of backup retention 
rules except that at least one full backup must remain in the repository.
```

The description is pretty simple, it will now be possible to expire a specific 
backup set using the `--set` option.

However, WAL archives will be kept according their own retention policy because 
of the next commit explained.

---

## WAL archives strict retention policy

[Expire WAL archive only when repo-retention-archive threshold is met:](https://github.com/pgbackrest/pgbackrest/commit/c5241e50)

```
Previously when retention-archive was set (either by the user or by default), 
archives prior to the archive-start of the oldest remaining full backup (after 
backup expiration occurred) would be expired even though the retention-archive 
threshold had not been met. For example, if there were 1 full backup remaining 
after backup expiration and the retention-archive was set to 2 and 
retention-archive-type=full, then archives prior to the archive-start of the 
remaining full backup would still be removed even though retention-archive 
required 2 full backups remaining before archives should be expired.

The thought was to keep the archive directory clean and since the full backup 
did not require prior archives, it was safe to delete them. However, this has 
caused problems for some users in the past (because they needed the WAL for 
other purposes) and with the new adhoc and time-based retention features, 
it was decided that the archives should remain until the threshold was met. 
The archives will eventually be removed and if having them causes space issues, 
the expire command and the retention-archive can always be run and adjusted.
```

The `expire` command removed all the archives prior to the archive-start of the 
oldest remaining full backup. That won't be the case anymore for more flexibility.
However, we'll see later in the examples that it can somehow mess with archives 
cleaning.

WAL archives retention policy should only be used to expire archives more 
aggressively than backups, and only if it's really necessary. 

-----

# Retention policy options

## `repo-retention-full`

Full backup retention count/time.

When a full backup expires, all differential and incremental backups associated 
with the full backup will also expire. When the option is not defined, a 
warning will be issued. If indefinite retention is desired then set the option 
to the max value.

---

## `repo-retention-full-type`

Determines whether the `repo-retention-full` setting represents a time period 
(days) or count of full backups to keep. 

If set to time then full backups older than `repo-retention-full` will be 
removed from the repository if there is at least one backup that is equal to 
or greater than the `repo-retention-full` setting. 

For example, if `repo-retention-full` is 30 (days) and there are 2 full 
backups: one 25 days old and one 35 days old, no full backups will be expired 
because expiring the 35 day old backup would leave only the 25 day old backup, 
which would violate the 30 day retention policy of having **at least** one 
backup 30 days old before an older one can be expired. 

Archived WAL older than the oldest full backup remaining will be automatically 
expired unless `repo-retention-archive-type` and `repo-retention-archive` 
are explicitly set.

---

## `repo-retention-diff`

Number of differential backups to retain.

When a differential backup expires, all incremental backups associated with 
the differential backup will also expire. When not defined all differential 
backups will be kept until the full backups they depend on expire.

---

## `repo-retention-archive-type`

Backup type for WAL retention.

If set to full, pgBackRest will keep archive logs for the number of full 
backups defined by `repo-retention-archive`. 

If set to diff (differential), pgBackRest will keep archive logs for the number 
of full **and** differential backups defined by `repo-retention-archive`, 
meaning if the last backup taken was a full backup, it will be counted as a 
differential for the purpose of `repo-retention`. 

If set to incr (incremental), pgBackRest will keep archive logs for the number 
of full, differential, **and** incremental backups defined by `repo-retention-archive`. 

**It is recommended** that this setting not be changed from the default which 
will only expire WAL in conjunction with expiring full backups.

---

## `repo-retention-archive`

Number of backups worth of continuous WAL to retain.

NOTE: WAL segments required to make a backup consistent are always retained 
until the backup is expired regardless of how this option is configured.

If this value is not set and `repo-retention-full-type` is count (default), 
then the archive to expire will default to the `repo-retention-full` 
(or `repo-retention-diff`) value corresponding to the `repo-retention-archive-type` 
if set to full (or diff). This will ensure that WAL is only expired for backups 
that are already expired. 

If `repo-retention-full-type` is time, then this value will default to removing 
archives that are earlier than the oldest full backup retained after satisfying 
the `repo-retention-full` setting.

This option **must** be set if `repo-retention-archive-type` is set to incr. 

If disk space is at a premium, then this setting, in conjunction with 
`repo-retention-archive-type`, can be used to aggressively expire WAL segments. 
However, doing so negates the ability to perform PITR from the backups with 
expired WAL and is therefore **not recommended**.

-----

# PostgreSQL and pgBackRest installation

For the purpose of this article and to apply the examples, we'll use a 
CentOS-7 fresh server.

First of all, let's install PostgreSQL directly from the PGDG yum repositories:

```bash
$ sudo yum install -y yum install https://download.postgresql.org/pub/repos/\
yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
$ sudo yum install -y postgresql12-server postgresql12-contrib
```

Now, we need to compile pgBackRest from the latest sources:

```bash
$ sudo yum install -y gcc make openssl-devel libxml2-devel bzip2-devel \
postgresql-devel perl-ExtUtils-Embed git
$ sudo mkdir /build
$ sudo git clone https://github.com/pgbackrest/pgbackrest.git /build/pgbackrest
$ cd /build/pgbackrest/src && sudo ./configure
$ sudo make -s -C /build/pgbackrest/src
$ sudo mkdir -p -m 750 /var/lib/pgbackrest
$ sudo chown postgres:postgres /var/lib/pgbackrest
$ sudo mkdir -p -m 700 /var/log/pgbackrest
$ sudo chown postgres:postgres /var/log/pgbackrest
$ sudo mv /build/pgbackrest/src/pgbackrest /usr/bin/pgbackrest
```

Once installed in `/usr/bin/`, check pgBackRest version:

```bash
$ sudo -iu postgres pgbackrest version
pgBackRest 2.27dev
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
archive_command = 'pgbackrest --stanza=my_stanza archive-push %p'
EOF
```

Start the PostgreSQL cluster:

```bash
$ sudo systemctl start postgresql-12
```

Now, configure pgBackRest:

```bash
$ cat<<EOF | sudo tee "/etc/pgbackrest.conf"
[global]
repo1-path=/var/lib/pgbackrest
process-max=2
log-level-console=warn
log-level-file=debug
start-fast=y
delta=y

[my_stanza]
pg1-path=/var/lib/pgsql/12/data
EOF
```

Finally, create the stanza and check that everything works fine:

```bash
$ sudo -iu postgres pgbackrest --stanza=my_stanza stanza-create
$ sudo -iu postgres pgbackrest --stanza=my_stanza check
```

-----

# Examples

## Full and differential backups count expiration

The easiest policy is to setup full with or without differential backups count 
expiration.

Let's then add the following parameters to `/etc/pgbackrest.conf` in the 
`[global]` part:

```ini
repo1-retention-full=2
repo1-retention-full-type=count
repo1-retention-diff=1
```

We'll keep 2 full and 1 differential backup.

Let's take one backup of each kind (full/diff/incr):

```bash
$ sudo -iu postgres pgbackrest --stanza=my_stanza --type=full backup
$ sudo -iu postgres pgbackrest --stanza=my_stanza --type=diff backup
$ sudo -iu postgres pgbackrest --stanza=my_stanza --type=incr backup
$ sudo -iu postgres pgbackrest --stanza=my_stanza info
stanza: my_stanza
    status: ok
    cipher: none

    db (current)
        wal archive min/max (12-1): 000000010000000000000001/000000010000000000000006

        full backup: 20200525-090008F
            timestamp start/stop: 2020-05-25 09:00:08 / 2020-05-25 09:00:19
            wal start/stop: 000000010000000000000003 / 000000010000000000000003
            database size: 24.2MB, backup size: 24.2MB
            repository size: 2.9MB, repository backup size: 2.9MB

        diff backup: 20200525-090008F_20200525-090022D
            timestamp start/stop: 2020-05-25 09:00:22 / 2020-05-25 09:00:23
            wal start/stop: 000000010000000000000004 / 000000010000000000000004
            database size: 24.2MB, backup size: 8.3KB
            repository size: 2.9MB, repository backup size: 396B
            backup reference list: 20200525-090008F

        incr backup: 20200525-090008F_20200525-090031I
            timestamp start/stop: 2020-05-25 09:00:31 / 2020-05-25 09:00:33
            wal start/stop: 000000010000000000000006 / 000000010000000000000006
            database size: 24.2MB, backup size: 8.3KB
            repository size: 2.9MB, repository backup size: 401B
            backup reference list: 20200525-090008F
```

Let's take a new differential backup to observe the expiration:

```bash
$ sudo -iu postgres pgbackrest --stanza=my_stanza --type=diff backup
$ sudo -iu postgres pgbackrest --stanza=my_stanza info
stanza: my_stanza
    status: ok
    cipher: none

    db (current)
        wal archive min/max (12-1): 000000010000000000000001/000000010000000000000008

        full backup: 20200525-090008F
            timestamp start/stop: 2020-05-25 09:00:08 / 2020-05-25 09:00:19
            wal start/stop: 000000010000000000000003 / 000000010000000000000003
            database size: 24.2MB, backup size: 24.2MB
            repository size: 2.9MB, repository backup size: 2.9MB

        diff backup: 20200525-090008F_20200525-090102D
            timestamp start/stop: 2020-05-25 09:01:02 / 2020-05-25 09:01:04
            wal start/stop: 000000010000000000000008 / 000000010000000000000008
            database size: 24.2MB, backup size: 8.3KB
            repository size: 2.9MB, repository backup size: 400B
            backup reference list: 20200525-090008F
```

As we can see, the oldest WAL archive in the repository is 
000000010000000000000001, while the oldest start WAL backup location is 
000000010000000000000003.

To expire it, we need to respect the `repo-retention-full` policy. It wouldn't 
have been the case before commit [c5241e50](https://github.com/pgbackrest/pgbackrest/commit/c5241e50).

Let's take a new full backup:

```bash
$ sudo -iu postgres pgbackrest --stanza=my_stanza --type=full backup
$ sudo -iu postgres pgbackrest --stanza=my_stanza info
stanza: my_stanza
    status: ok
    cipher: none

    db (current)
        wal archive min/max (12-1): 000000010000000000000003/00000001000000000000000A

        full backup: 20200525-090008F
            timestamp start/stop: 2020-05-25 09:00:08 / 2020-05-25 09:00:19
            wal start/stop: 000000010000000000000003 / 000000010000000000000003
            database size: 24.2MB, backup size: 24.2MB
            repository size: 2.9MB, repository backup size: 2.9MB

        full backup: 20200525-090149F
            timestamp start/stop: 2020-05-25 09:01:49 / 2020-05-25 09:01:54
            wal start/stop: 00000001000000000000000A / 00000001000000000000000A
            database size: 24.2MB, backup size: 24.2MB
            repository size: 2.9MB, repository backup size: 2.9MB
```

The WAL archives before 000000010000000000000003 have been removed. 

Also, we can remark that the differential backup have been expired too:

```bash
INFO: expire diff backup 20200525-090008F_20200525-090102D
INFO: remove expired backup 20200525-090008F_20200525-090102D
INFO: expire command end: completed successfully
```

---

## Archives specific retention policy, aggressive expiration

<!--
```bash
$ sudo yum install -y nagios-plugins-pgbackrest
$  sudo -iu postgres /usr/lib64/nagios/plugins/check_pgbackrest --version
check_pgbackrest version 1.8, Perl 5.16.3
$ sudo -iu postgres /usr/lib64/nagios/plugins/check_pgbackrest \
--stanza=my_stanza --service=archives --repo-path=/var/lib/pgbackrest/archive \
--output=human --debug --list-archive
...
DEBUG: found 000000010000000000000003
DEBUG: found 000000010000000000000004
DEBUG: found 000000010000000000000005
DEBUG: found 000000010000000000000006
DEBUG: found 000000010000000000000007
DEBUG: found 000000010000000000000008
DEBUG: found 000000010000000000000009
DEBUG: found 00000001000000000000000A
...
```
-->

We should now have all the WAL archives below:
* 000000010000000000000003
* 000000010000000000000004
* 000000010000000000000005
* 000000010000000000000006
* 000000010000000000000007
* 000000010000000000000008
* 000000010000000000000009
* 00000001000000000000000A

What if we want to expire WAL archives between 000000010000000000000003 and 
00000001000000000000000A?

Adjust `/etc/pgbackrest.conf` with:

```ini
repo1-retention-full=2
repo1-retention-full-type=count
repo1-retention-archive-type=full
repo1-retention-archive=1
```

Let's now manually run the expire command and check the archives repository:

```bash
$ sudo -iu postgres pgbackrest --stanza=my_stanza expire
$ sudo ls /var/lib/pgbackrest/archive/my_stanza/12-1/0000000100000000/
000000010000000000000003.00000060.backup
000000010000000000000003-d3fccf102289b1abec25065708ec26e073b331a0.gz
00000001000000000000000A.00000028.backup
00000001000000000000000A-09b73b4444c60873d757cea635cfdbacd7068762.gz
```

Except the archives needed for the backups consistency, all the files older 
than the last full backup have been removed.

---

## Archives specific retention policy, more archives than backups ?

Let's now aim the opposite. Keep more archives than backups.

Adjust `/etc/pgbackrest.conf` with:

```ini
repo1-retention-full=2
repo1-retention-full-type=count
repo1-retention-archive-type=full
repo1-retention-archive=3
```

Generate some activity:

```bash
$ sudo -iu postgres pgbackrest --stanza=my_stanza check
$ sudo -iu postgres pgbackrest --stanza=my_stanza --type=full backup
$ sudo -iu postgres pgbackrest --stanza=my_stanza info
stanza: my_stanza
    status: ok
    cipher: none

    db (current)
        wal archive min/max (12-1): 000000010000000000000003/00000001000000000000000D

        full backup: 20200525-090149F
            timestamp start/stop: 2020-05-25 09:01:49 / 2020-05-25 09:01:54
            wal start/stop: 00000001000000000000000A / 00000001000000000000000A
            database size: 24.2MB, backup size: 24.2MB
            repository size: 2.9MB, repository backup size: 2.9MB

        full backup: 20200525-090745F
            timestamp start/stop: 2020-05-25 09:07:45 / 2020-05-25 09:07:50
            wal start/stop: 00000001000000000000000D / 00000001000000000000000D
            database size: 24.2MB, backup size: 24.2MB
            repository size: 2.9MB, repository backup size: 2.9MB

$ sudo -iu postgres pgbackrest --stanza=my_stanza check
$ sudo -iu postgres pgbackrest --stanza=my_stanza --type=full backup
$ sudo -iu postgres pgbackrest --stanza=my_stanza info
stanza: my_stanza
    status: ok
    cipher: none

    db (current)
        wal archive min/max (12-1): 000000010000000000000003/000000010000000000000010

        full backup: 20200525-090745F
            timestamp start/stop: 2020-05-25 09:07:45 / 2020-05-25 09:07:50
            wal start/stop: 00000001000000000000000D / 00000001000000000000000D
            database size: 24.2MB, backup size: 24.2MB
            repository size: 2.9MB, repository backup size: 2.9MB

        full backup: 20200525-090908F
            timestamp start/stop: 2020-05-25 09:09:08 / 2020-05-25 09:09:14
            wal start/stop: 000000010000000000000010 / 000000010000000000000010
            database size: 24.2MB, backup size: 24.2MB
            repository size: 2.9MB, repository backup size: 2.9MB

$ sudo ls /var/lib/pgbackrest/archive/my_stanza/12-1/0000000100000000/
000000010000000000000003.00000060.backup
000000010000000000000003-d3fccf102289b1abec25065708ec26e073b331a0.gz
00000001000000000000000A.00000028.backup
00000001000000000000000A-09b73b4444c60873d757cea635cfdbacd7068762.gz
00000001000000000000000B-6b66b302c275b80aabcfcc42199bcadf90dadc89.gz
00000001000000000000000C-0eff30953bebf1ca55e9a328d7fe4fde0fe9356b.gz
00000001000000000000000D.00000028.backup
00000001000000000000000D-3eeb462f0b47721843e2840952c1abe815a826ef.gz
00000001000000000000000E-1156ff47bc5daddffb0a48d4fafb47b9903f5540.gz
00000001000000000000000F-8ce2b1e48733a73443c62378a73f42fa0337de0a.gz
000000010000000000000010.00000028.backup
000000010000000000000010-0a82ae890f0bbc0a7da7a9f16ccaf52121749891.gz
```

As we can notice, **000000010000000000000003** is still there while we would 
have expected to only keep WAL archives after **00000001000000000000000A**.

It's pretty easy to imagine that as we don't really know WHAT was the start 
point of the third full backup (since it has been expired), we can't expire 
the WAL archives correctly.

The `repo-retention-archive` option should really be employed with **extreme** 
precaution since it can mess with the WAL archives that would be needed 
for _Point-in-time Recovery_ or would even block the archives cleaning.

This option is better employed with `repo-retention-archive-type` **diff** or 
**incr**.

---

## Archives specific retention policy, with differential backups

Adjust `/etc/pgbackrest.conf` with:

```ini
repo1-retention-full=2
repo1-retention-full-type=count
repo1-retention-diff=2
repo1-retention-archive-type=diff
repo1-retention-archive=3
```

The idea is to have 4 backups (2 full, 2 differential) and only keep WAL 
archives for 3 of them.

Generate some activity:

```bash
$ sudo -iu postgres pgbackrest --stanza=my_stanza check
$ sudo -iu postgres pgbackrest --stanza=my_stanza --type=diff backup
$ sudo -iu postgres pgbackrest --stanza=my_stanza check
$ sudo -iu postgres pgbackrest --stanza=my_stanza info
stanza: my_stanza
    status: ok
    cipher: none

    db (current)
        wal archive min/max (12-1): 00000001000000000000000D/000000010000000000000014

        full backup: 20200525-090745F
            timestamp start/stop: 2020-05-25 09:07:45 / 2020-05-25 09:07:50
            wal start/stop: 00000001000000000000000D / 00000001000000000000000D
            database size: 24.2MB, backup size: 24.2MB
            repository size: 2.9MB, repository backup size: 2.9MB

        full backup: 20200525-090908F
            timestamp start/stop: 2020-05-25 09:09:08 / 2020-05-25 09:09:14
            wal start/stop: 000000010000000000000010 / 000000010000000000000010
            database size: 24.2MB, backup size: 24.2MB
            repository size: 2.9MB, repository backup size: 2.9MB

        diff backup: 20200525-090908F_20200525-091631D
            timestamp start/stop: 2020-05-25 09:16:31 / 2020-05-25 09:16:33
            wal start/stop: 000000010000000000000013 / 000000010000000000000013
            database size: 24.2MB, backup size: 9.3KB
            repository size: 2.9MB, repository backup size: 728B
            backup reference list: 20200525-090908F

$ sudo -iu postgres pgbackrest --stanza=my_stanza --type=full backup
$ sudo -iu postgres pgbackrest --stanza=my_stanza check
$ sudo -iu postgres pgbackrest --stanza=my_stanza info
stanza: my_stanza
    status: ok
    cipher: none

    db (current)
        wal archive min/max (12-1): 000000010000000000000010/000000010000000000000017

        full backup: 20200525-090908F
            timestamp start/stop: 2020-05-25 09:09:08 / 2020-05-25 09:09:14
            wal start/stop: 000000010000000000000010 / 000000010000000000000010
            database size: 24.2MB, backup size: 24.2MB
            repository size: 2.9MB, repository backup size: 2.9MB

        diff backup: 20200525-090908F_20200525-091631D
            timestamp start/stop: 2020-05-25 09:16:31 / 2020-05-25 09:16:33
            wal start/stop: 000000010000000000000013 / 000000010000000000000013
            database size: 24.2MB, backup size: 9.3KB
            repository size: 2.9MB, repository backup size: 728B
            backup reference list: 20200525-090908F

        full backup: 20200525-091716F
            timestamp start/stop: 2020-05-25 09:17:16 / 2020-05-25 09:17:22
            wal start/stop: 000000010000000000000016 / 000000010000000000000016
            database size: 24.2MB, backup size: 24.2MB
            repository size: 2.9MB, repository backup size: 2.9MB

$ sudo -iu postgres pgbackrest --stanza=my_stanza --type=diff backup
$ sudo -iu postgres pgbackrest --stanza=my_stanza check
$ sudo -iu postgres pgbackrest --stanza=my_stanza info
stanza: my_stanza
    status: ok
    cipher: none

    db (current)
        wal archive min/max (12-1): 000000010000000000000010/00000001000000000000001A

        full backup: 20200525-090908F
            timestamp start/stop: 2020-05-25 09:09:08 / 2020-05-25 09:09:14
            wal start/stop: 000000010000000000000010 / 000000010000000000000010
            database size: 24.2MB, backup size: 24.2MB
            repository size: 2.9MB, repository backup size: 2.9MB

        full backup: 20200525-091716F
            timestamp start/stop: 2020-05-25 09:17:16 / 2020-05-25 09:17:22
            wal start/stop: 000000010000000000000016 / 000000010000000000000016
            database size: 24.2MB, backup size: 24.2MB
            repository size: 2.9MB, repository backup size: 2.9MB

        diff backup: 20200525-091716F_20200525-091736D
            timestamp start/stop: 2020-05-25 09:17:36 / 2020-05-25 09:17:38
            wal start/stop: 000000010000000000000019 / 000000010000000000000019
            database size: 24.2MB, backup size: 9.8KB
            repository size: 2.9MB, repository backup size: 765B
            backup reference list: 20200525-091716F

$ sudo -iu postgres pgbackrest --stanza=my_stanza --type=diff backup
$ sudo -iu postgres pgbackrest --stanza=my_stanza check
$ sudo -iu postgres pgbackrest --stanza=my_stanza info
stanza: my_stanza
    status: ok
    cipher: none

    db (current)
        wal archive min/max (12-1): 000000010000000000000010/00000001000000000000001D

        full backup: 20200525-090908F
            timestamp start/stop: 2020-05-25 09:09:08 / 2020-05-25 09:09:14
            wal start/stop: 000000010000000000000010 / 000000010000000000000010
            database size: 24.2MB, backup size: 24.2MB
            repository size: 2.9MB, repository backup size: 2.9MB

        full backup: 20200525-091716F
            timestamp start/stop: 2020-05-25 09:17:16 / 2020-05-25 09:17:22
            wal start/stop: 000000010000000000000016 / 000000010000000000000016
            database size: 24.2MB, backup size: 24.2MB
            repository size: 2.9MB, repository backup size: 2.9MB

        diff backup: 20200525-091716F_20200525-091736D
            timestamp start/stop: 2020-05-25 09:17:36 / 2020-05-25 09:17:38
            wal start/stop: 000000010000000000000019 / 000000010000000000000019
            database size: 24.2MB, backup size: 9.8KB
            repository size: 2.9MB, repository backup size: 765B
            backup reference list: 20200525-091716F

        diff backup: 20200525-091716F_20200525-091849D
            timestamp start/stop: 2020-05-25 09:18:49 / 2020-05-25 09:18:51
            wal start/stop: 00000001000000000000001C / 00000001000000000000001C
            database size: 24.2MB, backup size: 10KB
            repository size: 2.9MB, repository backup size: 778B
            backup reference list: 20200525-091716F
```

At this point, we should have the archives between 000000010000000000000010 
and 00000001000000000000001D. Since we've included differential backups in the 
archives retention policy and requested to remove the archives older than the 
3 last backups, pgBackRest should have removed the archives between 
000000010000000000000010 and 000000010000000000000016.

It would then give:
* found 000000010000000000000010
* missing 000000010000000000000011
* missing 000000010000000000000012
* missing 000000010000000000000013
* missing 000000010000000000000014
* missing 000000010000000000000015
* found 000000010000000000000016
* found 000000010000000000000017
* found 000000010000000000000018
* found 000000010000000000000019
* found 00000001000000000000001A
* found 00000001000000000000001B
* found 00000001000000000000001C
* found 00000001000000000000001D

Let's check it:

```bash
$ sudo ls /var/lib/pgbackrest/archive/my_stanza/12-1/0000000100000000/
000000010000000000000010.00000028.backup
000000010000000000000010-0a82ae890f0bbc0a7da7a9f16ccaf52121749891.gz
000000010000000000000016.00000028.backup
000000010000000000000016-8d2b582e41c8f20411724121841b5bcf49bcb58a.gz
000000010000000000000017-eeee42ef6d7f5c8cf580b42282fb0c8d9df1bdd1.gz
000000010000000000000018-2b3b61815e88a8063a1f3c18995e85def5b0d893.gz
000000010000000000000019.00000028.backup
000000010000000000000019-58d1b03d5f4b7e8eec16fb1055b4fb74a08905ff.gz
00000001000000000000001A-79ed8daa0a55a53a99007b0aea5676fe5d844089.gz
00000001000000000000001B-58e82550d5b7a521cfa75e6be204e1a45c93237d.gz
00000001000000000000001C.00000028.backup
00000001000000000000001C-345c8ebc064465e73700153b5c32516738be4b0d.gz
00000001000000000000001D-c8821c7ba3dc9f4ff44a9291dfa222be1075aa9f.gz
```

That's what we expected. Even if it seems an easy way to save backup space by 
removing WAL archives, it can be very tricky to predict exactly which archives 
to keep or to remove. If you decide to use those options, do it very carefully!

---

## Full backups time-based expiration

Adjust `/etc/pgbackrest.conf` with:

```ini
repo1-retention-full=5
repo1-retention-full-type=time
```

We currently have 4 backups, taken on 2020-05-25. We'll require to expire it 
after 5 days.

```bash
$ sudo -iu postgres pgbackrest --stanza=my_stanza info
stanza: my_stanza
    status: ok
    cipher: none

    db (current)
        wal archive min/max (12-1): 000000010000000000000010/00000001000000000000001E

        full backup: 20200525-090908F
            timestamp start/stop: 2020-05-25 09:09:08 / 2020-05-25 09:09:14
            wal start/stop: 000000010000000000000010 / 000000010000000000000010
            database size: 24.2MB, backup size: 24.2MB
            repository size: 2.9MB, repository backup size: 2.9MB

        full backup: 20200525-091716F
            timestamp start/stop: 2020-05-25 09:17:16 / 2020-05-25 09:17:22
            wal start/stop: 000000010000000000000016 / 000000010000000000000016
            database size: 24.2MB, backup size: 24.2MB
            repository size: 2.9MB, repository backup size: 2.9MB

        diff backup: 20200525-091716F_20200525-091736D
            timestamp start/stop: 2020-05-25 09:17:36 / 2020-05-25 09:17:38
            wal start/stop: 000000010000000000000019 / 000000010000000000000019
            database size: 24.2MB, backup size: 9.8KB
            repository size: 2.9MB, repository backup size: 765B
            backup reference list: 20200525-091716F

        diff backup: 20200525-091716F_20200525-091849D
            timestamp start/stop: 2020-05-25 09:18:49 / 2020-05-25 09:18:51
            wal start/stop: 00000001000000000000001C / 00000001000000000000001C
            database size: 24.2MB, backup size: 10KB
            repository size: 2.9MB, repository backup size: 778B
            backup reference list: 20200525-091716F

$ sudo -iu postgres pgbackrest --stanza=my_stanza expire --log-level-console=info
INFO: time-based archive retention not met - archive logs will not be expired
```

5 days later, let's run the expire command:

```bash
$ sudo date -s "30 May 2020 09:30:00"
Sat May 30 09:30:00 CEST 2020

$ sudo -iu postgres pgbackrest --stanza=my_stanza expire --log-level-console=info
INFO: expire time-based backup 20200525-090908F

$ sudo -iu postgres pgbackrest --stanza=my_stanza info
stanza: my_stanza
    status: ok
    cipher: none

    db (current)
        wal archive min/max (12-1): 000000010000000000000016/00000001000000000000001E

        full backup: 20200525-091716F
            timestamp start/stop: 2020-05-25 09:17:16 / 2020-05-25 09:17:22
            wal start/stop: 000000010000000000000016 / 000000010000000000000016
            database size: 24.2MB, backup size: 24.2MB
            repository size: 2.9MB, repository backup size: 2.9MB

        diff backup: 20200525-091716F_20200525-091736D
            timestamp start/stop: 2020-05-25 09:17:36 / 2020-05-25 09:17:38
            wal start/stop: 000000010000000000000019 / 000000010000000000000019
            database size: 24.2MB, backup size: 9.8KB
            repository size: 2.9MB, repository backup size: 765B
            backup reference list: 20200525-091716F

        diff backup: 20200525-091716F_20200525-091849D
            timestamp start/stop: 2020-05-25 09:18:49 / 2020-05-25 09:18:51
            wal start/stop: 00000001000000000000001C / 00000001000000000000001C
            database size: 24.2MB, backup size: 10KB
            repository size: 2.9MB, repository backup size: 778B
            backup reference list: 20200525-091716F
```

Only the first full backup `20200525-090908F` have been expired. Indeed, 
pgBackRest will keep at least one full backup older than the time period 
requested.

Let's take a new full backup:

```bash
$ sudo -iu postgres pgbackrest --stanza=my_stanza --type=full backup
$ sudo -iu postgres pgbackrest --stanza=my_stanza info
stanza: my_stanza
    status: ok
    cipher: none

    db (current)
        wal archive min/max (12-1): 000000010000000000000016/000000010000000000000020

        full backup: 20200525-091716F
            timestamp start/stop: 2020-05-25 09:17:16 / 2020-05-25 09:17:22
            wal start/stop: 000000010000000000000016 / 000000010000000000000016
            database size: 24.2MB, backup size: 24.2MB
            repository size: 2.9MB, repository backup size: 2.9MB

        diff backup: 20200525-091716F_20200525-091736D
            timestamp start/stop: 2020-05-25 09:17:36 / 2020-05-25 09:17:38
            wal start/stop: 000000010000000000000019 / 000000010000000000000019
            database size: 24.2MB, backup size: 9.8KB
            repository size: 2.9MB, repository backup size: 765B
            backup reference list: 20200525-091716F

        diff backup: 20200525-091716F_20200525-091849D
            timestamp start/stop: 2020-05-25 09:18:49 / 2020-05-25 09:18:51
            wal start/stop: 00000001000000000000001C / 00000001000000000000001C
            database size: 24.2MB, backup size: 10KB
            repository size: 2.9MB, repository backup size: 778B
            backup reference list: 20200525-091716F

        full backup: 20200530-093103F
            timestamp start/stop: 2020-05-30 09:31:03 / 2020-05-30 09:31:09
            wal start/stop: 000000010000000000000020 / 000000010000000000000020
            database size: 24.2MB, backup size: 24.2MB
            repository size: 2.9MB, repository backup size: 2.9MB

$ sudo -iu postgres pgbackrest --stanza=my_stanza expire --log-level-console=info
INFO: time-based archive retention not met - archive logs will not be expired
```

Since the new backup doesn't match the time period (at least 5 days old), the 
previous full backup is kept.

Let's now wait some more days and verify the expiration:

```bash
$ sudo date -s "09 June 2020 09:30:00"
Tue Jun  9 09:30:00 CEST 2020

$ sudo -iu postgres pgbackrest --stanza=my_stanza expire --log-level-console=info
INFO: expire time-based backup set: 20200525-091716F, 20200525-091716F_20200525-091736D, 20200525-091716F_20200525-091849D

$ sudo -iu postgres pgbackrest --stanza=my_stanza info
stanza: my_stanza
    status: ok
    cipher: none

    db (current)
        wal archive min/max (12-1): 000000010000000000000020/000000010000000000000020

        full backup: 20200530-093103F
            timestamp start/stop: 2020-05-30 09:31:03 / 2020-05-30 09:31:09
            wal start/stop: 000000010000000000000020 / 000000010000000000000020
            database size: 24.2MB, backup size: 24.2MB
            repository size: 2.9MB, repository backup size: 2.9MB
```

The old backups (2020-05-25) have now all been removed.

---

## Expire specific backup set

Let's try the new `--set` option in the expire command:

```bash
$ sudo -iu postgres pgbackrest --stanza=my_stanza expire --log-level-console=info --set="20200530-093103F"
ERROR: [075]: full backup 20200530-093103F cannot be expired until another full backup has been created
```

At least one full backup must exist. Let's retry with another backup:

```bash
$ sudo -iu postgres pgbackrest --stanza=my_stanza --type=full backup
$ sudo -iu postgres pgbackrest --stanza=my_stanza info
stanza: my_stanza
    status: ok
    cipher: none

    db (current)
        wal archive min/max (12-1): 000000010000000000000020/000000010000000000000022

        full backup: 20200530-093103F
            timestamp start/stop: 2020-05-30 09:31:03 / 2020-05-30 09:31:09
            wal start/stop: 000000010000000000000020 / 000000010000000000000020
            database size: 24.2MB, backup size: 24.2MB
            repository size: 2.9MB, repository backup size: 2.9MB

        full backup: 20200609-094040F
            timestamp start/stop: 2020-06-09 09:40:40 / 2020-06-09 09:40:55
            wal start/stop: 000000010000000000000022 / 000000010000000000000022
            database size: 24.2MB, backup size: 24.2MB
            repository size: 2.9MB, repository backup size: 2.9MB

$ sudo -iu postgres pgbackrest --stanza=my_stanza expire --log-level-console=info --set="20200530-093103F"
INFO: expire adhoc backup 20200530-093103F
INFO: time-based archive retention not met - archive logs will not be expired

$ sudo -iu postgres pgbackrest --stanza=my_stanza info
stanza: my_stanza
    status: ok
    cipher: none

    db (current)
        wal archive min/max (12-1): 000000010000000000000020/000000010000000000000022

        full backup: 20200609-094040F
            timestamp start/stop: 2020-06-09 09:40:40 / 2020-06-09 09:40:55
            wal start/stop: 000000010000000000000022 / 000000010000000000000022
            database size: 24.2MB, backup size: 24.2MB
            repository size: 2.9MB, repository backup size: 2.9MB
```

The backup set was expired **but** as the retention policy wasn't met, WAL 
archives are kept until a full backup matching the retention policy exists.

```bash
$ sudo date -s "15 June 2020 10:00:00"
Mon Jun 15 10:00:00 CEST 2020

$ sudo -iu postgres pgbackrest --stanza=my_stanza --type=full backup
$ sudo -iu postgres pgbackrest --stanza=my_stanza info
stanza: my_stanza
    status: ok
    cipher: none

    db (current)
        wal archive min/max (12-1): 000000010000000000000020/000000010000000000000026

        full backup: 20200609-094040F
            timestamp start/stop: 2020-06-09 09:40:40 / 2020-06-09 09:40:55
            wal start/stop: 000000010000000000000022 / 000000010000000000000022
            database size: 24.2MB, backup size: 24.2MB
            repository size: 2.9MB, repository backup size: 2.9MB

        full backup: 20200615-100009F
            timestamp start/stop: 2020-06-15 10:00:09 / 2020-06-15 10:00:15
            wal start/stop: 000000010000000000000026 / 000000010000000000000026
            database size: 24.2MB, backup size: 24.2MB
            repository size: 2.9MB, repository backup size: 2.9MB

$ sudo date -s "20 June 2020 10:05:00"
Sat Jun 20 10:05:00 CEST 2020

$ sudo -iu postgres pgbackrest --stanza=my_stanza expire --log-level-console=info
INFO: expire time-based backup 20200609-094040F

$ sudo -iu postgres pgbackrest --stanza=my_stanza info
stanza: my_stanza
    status: ok
    cipher: none

    db (current)
        wal archive min/max (12-1): 000000010000000000000026/000000010000000000000026

        full backup: 20200615-100009F
            timestamp start/stop: 2020-06-15 10:00:09 / 2020-06-15 10:00:15
            wal start/stop: 000000010000000000000026 / 000000010000000000000026
            database size: 24.2MB, backup size: 24.2MB
            repository size: 2.9MB, repository backup size: 2.9MB
```

The time-based retention will always keep at least one full backup matching the 
time period requested. We'll then need to wait an extra backup to see the old 
archives expired.

Once again, as it can mess with WAL archives cleaning, use this option carefully 
and preferably only with recent backups. Don't use it to force old backups 
expiry, adjust retention policy to achieve that.

-----

# Conclusion

The pgBackRest documentation is really deep and complete. All the options and 
commands are described in detail. It can be a lot of information for beginners. 
However, it gives us all the information needed without having to check the 
source code. Thanks to that complete documentation, I could get a lot of 
content for this post.

However, as seen in the examples, those options are not always intuitive and 
should be used carefully. The best advice would be to keep it as simple as 
possible.

Finally, and once again, special thanks to all the contributors. Pay attention 
to the release notes when the new version will be available, interesting new 
features are planned!