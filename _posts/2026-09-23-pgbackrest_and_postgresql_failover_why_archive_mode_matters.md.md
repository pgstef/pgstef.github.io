---
layout: post
title: "pgBackRest and PostgreSQL failover: why archive_mode matters"
date: 2026-09-23 15:35:00 +0200
---

A recent question on the community channels described a difficult situation: a standby had been promoted with `archive_mode=off`, and restarting the new primary was something the team wanted to avoid. Could they enable archiving on a downstream standby and take a pgBackRest backup from there instead?

Their attempt failed with `archive_mode must be enabled`, even with `archive-mode-check=n`. Removing the checks from pgBackRest's source code allowed their proof of concept to succeed, but was that enough to trust the approach?

My recommendation was to return to a supported configuration, either through a restart or a controlled switchover. But I wanted to take a closer look and see whether `archive-mode-check=n` might allow the standby-based approach.

So, in this post, we'll try to reproduce the situation, examine what pgBackRest checks, and see why a successful backup and restore don't tell the whole recovery story.

<!--MORE-->

-----

# pgBackRest and PostgreSQL failover: why `archive_mode` matters

The test environment consists of three VMs running AlmaLinux 10 and PostgreSQL 18: `pg1`, `pg2`, `pg3`.
We'll set up cascading replication: `pg1` → `pg2` → `pg3`.

## Set up the primary (`pg1`)

Let's first set up pgBackRest on the primary (`pg1`):

* In `/etc/pgbackrest.conf`:

```ini
[global]
repo1-path=/shared
repo1-retention-full=4
log-level-console=info
log-level-file=detail
compress-type=zst
start-fast=y

[demo]
pg1-path=/var/lib/pgsql/18/data
```

The configuration here is relatively simple: we will store WAL archives and backups on a `/shared` drive accessible from all hosts involved.

* In `postgresql.conf`:

```ini
archive_mode = on
archive_command = 'pgbackrest --stanza=demo archive-push %p'
```

Remember, changing `archive_mode` requires a PostgreSQL restart, while changing `archive_command` only requires a reload.

* Then, let's initialise the repository and make sure WAL archiving is working:

```bash
$ pgbackrest --stanza=demo stanza-create
$ pgbackrest --stanza=demo check
```

* Time to take the first backup:

```bash
$ pgbackrest --stanza=demo --type=full backup
P00   INFO: backup command begin 2.59.1: ...
P00   INFO: execute backup start:
backup begins after the requested immediate checkpoint completes
P00   INFO: backup start archive = 000000010000000000000003, lsn = 0/3000028
P00   INFO: check archive for prior segment 000000010000000000000002
P00   INFO: execute backup stop and wait for all WAL segments to archive
P00   INFO: backup stop archive = 000000010000000000000003, lsn = 0/3000158
P00   INFO: check archive for segment(s) 000000010000000000000003:000000010000000000000003
P00   INFO: new backup label = 20260922-150006F
P00   INFO: full backup size = 22.7MB, file total = 970
P00   INFO: backup command end: completed successfully
```

* Let's insert some data using `pgbench`:

```bash
$ createdb test
$ /usr/pgsql-18/bin/pgbench -i -s 100 test
$ /usr/pgsql-18/bin/pgbench test -T 60 -P1
```

* And take a new backup:

```bash
$ pgbackrest --stanza=demo --type=full backup
P00   INFO: backup command begin 2.59.1: ...
P00   INFO: execute backup start:
backup begins after the requested immediate checkpoint completes
P00   INFO: backup start archive = 000000010000000000000069, lsn = 0/690000D8
P00   INFO: check archive for prior segment 000000010000000000000068
P00   INFO: execute backup stop and wait for all WAL segments to archive
P00   INFO: backup stop archive = 000000010000000000000069, lsn = 0/690001D0
P00   INFO: check archive for segment(s) 000000010000000000000069:000000010000000000000069
P00   INFO: new backup label = 20260922-150220F
P00   INFO: full backup size = 1.5GB, file total = 1286
P00   INFO: backup command end: completed successfully
```
```bash
$ pgbackrest --stanza=demo info
stanza: demo
  status: ok
  cipher: none

  db (current)
    wal archive min/max (18): 000000010000000000000003/000000010000000000000069

    full backup: 20260922-150006F
      timestamp start/stop: 2026-09-22 15:00:06+00 / 2026-09-22 15:00:09+00
      wal start/stop: 000000010000000000000003 / 000000010000000000000003
      database size: 22.7MB, database backup size: 22.7MB
      repo1: backup set size: 2.7MB, backup size: 2.7MB

    full backup: 20260922-150220F
      timestamp start/stop: 2026-09-22 15:02:20+00 / 2026-09-22 15:02:27+00
      wal start/stop: 000000010000000000000069 / 000000010000000000000069
      database size: 1.5GB, database backup size: 1.5GB
      repo1: backup set size: 76.6MB, backup size: 76.6MB
```

## Set up the first standby (`pg2`)

* On `pg1`, create a replication user:

```bash
$ psql
postgres=# CREATE ROLE replic_user WITH LOGIN REPLICATION PASSWORD 'mypwd';
```

* Edit `pg_hba.conf`:

```ini
host   replication   replic_user   pg2   scram-sha-256
```

* Reload:

```bash
$ sudo systemctl reload postgresql-18.service
```

* Configure `.pgpass` on `pg2`:

```bash
$ echo '*:*:replication:replic_user:mypwd' >> ~postgres/.pgpass
$ chown postgres: ~postgres/.pgpass
$ chmod 0600 ~postgres/.pgpass
```

* Copy `pg1` using `pg_basebackup`:

```bash
$ pg_basebackup --format=plain --wal-method=stream \
    --checkpoint=fast --progress -D /var/lib/pgsql/18/data \
    -h pg1 -U replic_user -R
```

* As we plan to make `pg2` the next primary, let's configure pgBackRest too. In `/etc/pgbackrest.conf`:

```ini
[global]
repo1-path=/shared
repo1-retention-full=4
log-level-console=info
log-level-file=detail
compress-type=zst
start-fast=y

[demo]
pg1-path=/var/lib/pgsql/18/data
```

* And in `postgresql.conf`:

```ini
archive_mode = off
primary_conninfo = 'user=replic_user host=pg1'
```

Note that we deliberately leave `archive_mode = off` to illustrate why this is supposed to be a bad idea...

Once PostgreSQL has started, let's check the running processes:

* On `pg2`:

```bash
$ ps -o pid,cmd fx
    PID CMD
   4118 ps -o pid,cmd fx
   4106 /usr/pgsql-18/bin/postgres -D /var/lib/pgsql/18/data/
   4107  \_ postgres: logger
   4108  \_ postgres: io worker 2
   4109  \_ postgres: io worker 0
   4110  \_ postgres: io worker 1
   4111  \_ postgres: checkpointer
   4112  \_ postgres: background writer
   4113  \_ postgres: startup waiting for 00000001000000000000006D
   4114  \_ postgres: walreceiver
```

* On `pg1`:

```bash
$ ps -o pid,cmd fx
    PID CMD
   4377 ps -o pid,cmd fx
   4340 /usr/pgsql-18/bin/postgres -D /var/lib/pgsql/18/data/
   4341  \_ postgres: logger
   4342  \_ postgres: io worker 1
   4343  \_ postgres: io worker 0
   4344  \_ postgres: io worker 2
   4345  \_ postgres: checkpointer
   4346  \_ postgres: background writer
   4348  \_ postgres: walwriter
   4349  \_ postgres: autovacuum launcher
   4350  \_ postgres: archiver last was 00000001000000000000006C.00000028.backup
   4351  \_ postgres: logical replication launcher
   4371  \_ postgres: walsender replic_user 192.168.121.240(44122) streaming 0/6D000060
```

```sql
postgres=# SELECT * FROM pg_stat_replication \gx
-[ RECORD 1 ]----+-----------------------------
pid              | 4371
usesysid         | 16415
usename          | replic_user
application_name | walreceiver
client_addr      | 192.168.121.240
client_hostname  |
client_port      | 44122
backend_start    | 2026-09-22 15:34:17.0958+00
backend_xmin     |
state            | streaming
sent_lsn         | 0/6D000060
write_lsn        | 0/6D000060
flush_lsn        | 0/6D000060
replay_lsn       | 0/6D000060
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2026-09-22 15:35:52.15612+00
```

Let's check that we can still take backups from the primary (`pg1`):

```bash
$ pgbackrest --stanza=demo --type=incr backup
P00   INFO: backup command begin 2.59.1: ...
P00   INFO: last backup label = 20260922-150220F, version = 2.59.1
P00   INFO: execute backup start:
backup begins after the requested immediate checkpoint completes
P00   INFO: backup start archive = 00000001000000000000006E, lsn = 0/6E000028
P00   INFO: check archive for prior segment 00000001000000000000006D
P00   INFO: execute backup stop and wait for all WAL segments to archive
P00   INFO: backup stop archive = 00000001000000000000006E, lsn = 0/6E000158
P00   INFO: check archive for segment(s) 00000001000000000000006E:00000001000000000000006E
P00   INFO: new backup label = 20260922-150220F_20260922-153628I
P00   INFO: incr backup size = 3.4MB, file total = 1286
P00   INFO: backup command end: completed successfully
```
```bash
$ pgbackrest --stanza=demo info
stanza: demo
  status: ok
  cipher: none

  db (current)
    wal archive min/max (18): 000000010000000000000003/00000001000000000000006E

    full backup: 20260922-150006F
      timestamp start/stop: 2026-09-22 15:00:06+00 / 2026-09-22 15:00:09+00
      wal start/stop: 000000010000000000000003 / 000000010000000000000003
      database size: 22.7MB, database backup size: 22.7MB
      repo1: backup set size: 2.7MB, backup size: 2.7MB

    full backup: 20260922-150220F
      timestamp start/stop: 2026-09-22 15:02:20+00 / 2026-09-22 15:02:27+00
      wal start/stop: 000000010000000000000069 / 000000010000000000000069
      database size: 1.5GB, database backup size: 1.5GB
      repo1: backup set size: 76.6MB, backup size: 76.6MB

    incr backup: 20260922-150220F_20260922-153628I
      timestamp start/stop: 2026-09-22 15:36:28+00 / 2026-09-22 15:36:29+00
      wal start/stop: 00000001000000000000006E / 00000001000000000000006E
      database size: 1.5GB, database backup size: 3.4MB
      repo1: backup set size: 76.6MB, backup size: 1MB
      backup reference total: 1 full
```

With the current configuration, the backup command is expected to fail on `pg2` as it is currently a standby:

```bash
$ pgbackrest --stanza=demo backup
P00   INFO: backup command begin 2.59.1: ...
P00  ERROR: [056]: unable to find primary cluster - cannot proceed
            HINT: are all available clusters in recovery?
P00   INFO: backup command end: aborted with exception [056]
```

## Set up the second standby (`pg3`)

* On `pg2`, edit `pg_hba.conf`:

```ini
host   replication   replic_user   pg3   scram-sha-256
```

* Reload:

```bash
$ sudo systemctl reload postgresql-18.service
```

* Configure `.pgpass` on `pg3`:

```bash
$ echo '*:*:replication:replic_user:mypwd' >> ~postgres/.pgpass
$ chown postgres: ~postgres/.pgpass
$ chmod 0600 ~postgres/.pgpass
```

* Copy `pg2` using `pg_basebackup`:

```bash
$ pg_basebackup --format=plain --wal-method=stream \
    --checkpoint=fast --progress -D /var/lib/pgsql/18/data \
    -h pg2 -U replic_user -R
```

* Check `postgresql.conf` and adjust `primary_conninfo` if needed:

```ini
archive_mode = off
primary_conninfo = 'user=replic_user host=pg2'
```

Here too, we deliberately leave `archive_mode = off`.

Once PostgreSQL has started, let's check the running processes:

* On `pg3`:

```bash
$ ps -o pid,cmd fx
    PID CMD
   4093 ps -o pid,cmd fx
   4079 /usr/pgsql-18/bin/postgres -D /var/lib/pgsql/18/data/
   4080  \_ postgres: logger
   4081  \_ postgres: io worker 1
   4082  \_ postgres: io worker 0
   4083  \_ postgres: io worker 2
   4084  \_ postgres: checkpointer
   4085  \_ postgres: background writer
   4086  \_ postgres: startup recovering 00000001000000000000006F
   4087  \_ postgres: walreceiver
```

* On `pg2`:

```bash
$ ps -o pid,cmd fx
    PID CMD
   4238 ps -o pid,cmd fx
   4106 /usr/pgsql-18/bin/postgres -D /var/lib/pgsql/18/data/
   4107  \_ postgres: logger
   4108  \_ postgres: io worker 2
   4109  \_ postgres: io worker 0
   4110  \_ postgres: io worker 1
   4111  \_ postgres: checkpointer
   4112  \_ postgres: background writer
   4113  \_ postgres: startup recovering 00000001000000000000006F
   4114  \_ postgres: walreceiver streaming 0/6F000168
   4234  \_ postgres: walsender replic_user 192.168.121.36(44596) streaming 0/6F000168
```

```sql
postgres=# SELECT * FROM pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 4234
usesysid         | 16415
usename          | replic_user
application_name | walreceiver
client_addr      | 192.168.121.36
client_hostname  |
client_port      | 44596
backend_start    | 2026-09-22 15:43:18.868865+00
backend_xmin     |
state            | streaming
sent_lsn         | 0/6F000168
write_lsn        | 0/6F000168
flush_lsn        | 0/6F000168
replay_lsn       | 0/6F000168
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2026-09-22 15:44:23.933373+00
```

## Promote the first standby (`pg2`)

* Before stopping PostgreSQL on the primary (`pg1`), let's first add some data using `pgbench`:

```bash
$ /usr/pgsql-18/bin/pgbench test -T 60 -P1
```
```bash
$ sudo systemctl stop postgresql-18
```

* Promote `pg2`:

```sql
postgres=# SELECT pg_promote();
 pg_promote
------------
 t
(1 row)

postgres=# SELECT pg_switch_wal();
 pg_switch_wal
---------------
 0/92000128
(1 row)
```

* Check the running processes and try to take a backup:

```bash
$ ps -o pid,cmd fx
    PID CMD
   4379 ps -o pid,cmd fx
   4106 /usr/pgsql-18/bin/postgres -D /var/lib/pgsql/18/data/
   4107  \_ postgres: logger
   4108  \_ postgres: io worker 2
   4109  \_ postgres: io worker 0
   4110  \_ postgres: io worker 1
   4111  \_ postgres: checkpointer
   4112  \_ postgres: background writer
   4234  \_ postgres: walsender replic_user 192.168.121.36(44596) streaming 0/930020B8
   4362  \_ postgres: walwriter
   4363  \_ postgres: autovacuum launcher
   4364  \_ postgres: logical replication launcher
```

```bash
$ pgbackrest --stanza=demo backup
P00   INFO: backup command begin 2.59.1: ...
P00   INFO: last backup label = 20260922-150220F_20260922-153628I, version = 2.59.1
P00  ERROR: [087]: archive_mode must be enabled
P00   INFO: backup command end: aborted with exception [087]
```

Oops... we can't take backups as `archive_mode` is disabled!
If we don't want to restart this host, let's try to take a backup from the standby server using `archive_mode=always`.

## Try taking a backup from the standby (`pg3`)

* To be able to take a backup from a standby server, it must have SSH (or TLS) access to the primary:

```bash
$ ssh postgres@pg2 whoami
postgres
```

* Adjust the `/etc/pgbackrest.conf` configuration:

```ini
[global]
repo1-path=/shared
repo1-retention-full=4
log-level-console=info
log-level-file=detail
compress-type=zst
start-fast=y
backup-standby=y

[demo]
pg1-path=/var/lib/pgsql/18/data
pg2-host=pg2
pg2-host-user=postgres
pg2-path=/var/lib/pgsql/18/data
```

We have access to the backup repository (`/shared`) and to the current primary (`pg2`) through SSH.

* Adjust `postgresql.conf`:

```ini
archive_mode = always
archive_command = 'pgbackrest --stanza=demo archive-push %p'
```

Don't forget to restart PostgreSQL as we changed `archive_mode`.

* Let's try to take a backup using the [`archive-mode-check`](https://pgbackrest.org/configuration.html#section-backup/option-archive-mode-check) option:

```bash
$ pgbackrest --stanza=demo --no-archive-mode-check backup
P00   INFO: backup command begin 2.59.1: ...
P00   INFO: last backup label = 20260922-150220F_20260922-153628I, version = 2.59.1
P00  ERROR: [087]: archive_mode must be enabled
P00   INFO: backup command end: aborted with exception [087]
```

The backup still fails. The `archive-mode-check` option is only there to protect against `archive_mode=always` because WAL segments pushed from a standby server might be logically the same as WAL segments pushed from the primary but have different checksums. So when disabling this option, it is critical to ensure that only one archiver is writing to the repository.

The [source code](https://github.com/pgbackrest/pgbackrest/blob/release/2.59.1/src/command/check/common.c#L73) checks that `archive_mode` is enabled on the primary and that `archive_command` invokes a `pgbackrest` subcommand. These checks help prevent users from taking inconsistent backups without the WAL archives needed for recovery.

* The proper fix is to use the following settings on both `pg2` and `pg3`:

```ini
archive_mode = on
archive_command = 'pgbackrest --stanza=demo archive-push %p'
```

* Once done, we can take backups from the standby:

```bash
$ pgbackrest --stanza=demo --type=full backup
P00   INFO: backup command begin 2.59.1: ...
P00   INFO: execute backup start:
backup begins after the requested immediate checkpoint completes
P00   INFO: backup start archive = 000000020000000000000095, lsn = 0/95000028
P00   INFO: wait for replay on the standby to reach 0/95000028
P00   INFO: replay on the standby reached 0/95000028
P00   INFO: check archive for prior segment 000000020000000000000094
P00   INFO: execute backup stop and wait for all WAL segments to archive
P00   INFO: backup stop archive = 000000020000000000000095, lsn = 0/95000158
P00   INFO: check archive for segment(s) 000000020000000000000095:000000020000000000000095
P00   INFO: new backup label = 20260922-161237F
P00   INFO: full backup size = 1.5GB, file total = 1286
P00   INFO: backup command end: completed successfully
```
```bash
$ pgbackrest --stanza=demo info
stanza: demo
  status: ok
  cipher: none

  db (current)
    wal archive min/max (18): 000000010000000000000003/000000020000000000000095

    full backup: 20260922-150006F
      timestamp start/stop: 2026-09-22 15:00:06+00 / 2026-09-22 15:00:09+00
      wal start/stop: 000000010000000000000003 / 000000010000000000000003
      database size: 22.7MB, database backup size: 22.7MB
      repo1: backup set size: 2.7MB, backup size: 2.7MB

    full backup: 20260922-150220F
      timestamp start/stop: 2026-09-22 15:02:20+00 / 2026-09-22 15:02:27+00
      wal start/stop: 000000010000000000000069 / 000000010000000000000069
      database size: 1.5GB, database backup size: 1.5GB
      repo1: backup set size: 76.6MB, backup size: 76.6MB

    incr backup: 20260922-150220F_20260922-153628I
      timestamp start/stop: 2026-09-22 15:36:28+00 / 2026-09-22 15:36:29+00
      wal start/stop: 00000001000000000000006E / 00000001000000000000006E
      database size: 1.5GB, database backup size: 3.4MB
      repo1: backup set size: 76.6MB, backup size: 1MB
      backup reference total: 1 full

    full backup: 20260922-161237F
      timestamp start/stop: 2026-09-22 16:12:37+00 / 2026-09-22 16:12:53+00
      wal start/stop: 000000020000000000000095 / 000000020000000000000095
      database size: 1.5GB, database backup size: 1.5GB
      repo1: backup set size: 79.2MB, backup size: 79.2MB
```

## What did we miss by leaving `archive_mode` off?

Our initial primary (`pg1`) was on the first timeline (`00000001`), and after the promotion, the new primary (`pg2`) created a new timeline (`00000002`).

* Let's have a look inside the WAL archives repository:

```bash
$ pgbackrest repo-ls archive/demo/18-1/0000000100000000
...
000000010000000000000091-e004283b60f3d5b40055fa55a9e7e9687df11197.zst

$ pgbackrest repo-ls archive/demo/18-1/0000000200000000
000000020000000000000093-eb4d97a75cd5f61a8091ae0a58778f43f581e366.zst
000000020000000000000094-57794c5806b9209bad0d60f630f02e50ed9d4fe9.zst
000000020000000000000095-597cef837ae8f584b7fdf9b833798a89008688c8.zst
000000020000000000000095.00000028.backup
```

So basically, we don't have anything between `000000010000000000000091` and `000000020000000000000093`.
We don't know what happened between those 2 WAL segments, or exactly when the timeline switch occurred.
So, from the repository content itself, it would be impossible to jump from the first timeline to the second one, because we're also missing the `.history` file!

What's in the `.history` file exactly?

```bash
$ cat 18/data/pg_wal/00000002.history
1 0/920000A0  no recovery target specified
```

## A word of advice

When you build a DR environment, before you promote any server, make sure `archive_mode` is set to `on` and `archive_command` is defined.
And when you're not ready to have archiving (meaning you don't have an `archive_command` yet), it is often recommended to turn on `archive_mode` at the start of your cluster's life, but keep `archive_command` empty (or simply `/bin/true`) as changing `archive_command` later will then simply require a PostgreSQL reload, not a restart.

## Going further

Now that we know where the source code limitation is, could we mess with the source code to make it work, provided we set up WAL archiving on the standby?

Long story short, yes:

```bash
$ pgbackrest --stanza=demo --type=full backup
P00   INFO: backup command begin 2.60.0dev: ...
P00   WARN: archive_mode is off!
P00   WARN: archive_command not checked!
P00   INFO: execute backup start:
backup begins after the requested immediate checkpoint completes
P00   INFO: backup start archive = 00000002000000000000009A, lsn = 0/9A000028
P00   INFO: wait for replay on the standby to reach 0/9A000028
P00   INFO: replay on the standby reached 0/9A000028
P00   INFO: check archive for prior segment 000000020000000000000099
P00   INFO: execute backup stop and wait for all WAL segments to archive
P00   INFO: backup stop archive = 00000002000000000000009A, lsn = 0/9A000158
P00   INFO: check archive for segment(s) 00000002000000000000009A:00000002000000000000009A
P00   INFO: new backup label = 20260922-164401F
P00   INFO: full backup size = 1.5GB, file total = 1286
P00   INFO: backup command end: completed successfully
```

With `archive_mode=off` on the primary, and `archive_mode=always` + `archive_command` set on the standby, we could in theory have a valid backup. This is especially relevant because pgBackRest can check that the WAL segments needed for backup consistency have been archived before completing the backup command: `INFO: check archive for segment(s) 00000002000000000000009A:00000002000000000000009A`.

What would we miss by doing this? Let's have a look inside the WAL archives repository:

```bash
$ pgbackrest repo-ls archive/demo/18-1/0000000200000000
000000020000000000000093-eb4d97a75cd5f61a8091ae0a58778f43f581e366.zst
000000020000000000000094-57794c5806b9209bad0d60f630f02e50ed9d4fe9.zst
000000020000000000000095-597cef837ae8f584b7fdf9b833798a89008688c8.zst
000000020000000000000095.00000028.backup
000000020000000000000096-5bcf1051b0ecb1f83e21da2282711a182b4a72d9.zst
000000020000000000000097-64e2601978646859d0d5c1190de704d53c698d2b.zst
000000020000000000000098-9294abdf3f0839a6de9866a3c5a1571af07e94c9.zst
000000020000000000000099-c2c329f0127555b71203eba3b6608935893ce660.zst
00000002000000000000009A-a588cea8dceb285eef12838ef0481dbb0765985a.zst
```

For the previous backup, we did have a backup label archived:

```bash
$ pgbackrest repo-get archive/demo/18-1/0000000200000000/000000020000000000000095.00000028.backup
START WAL LOCATION: 0/95000028 (file 000000020000000000000095)
STOP WAL LOCATION: 0/95000158 (file 000000020000000000000095)
CHECKPOINT LOCATION: 0/95000080
BACKUP METHOD: streamed
BACKUP FROM: primary
START TIME: 2026-09-22 16:12:37 UTC
LABEL: pgBackRest backup started at 2026-09-22 16:12:37.255031+00
START TIMELINE: 2
STOP TIME: 2026-09-22 16:12:53 UTC
STOP TIMELINE: 2
```

Could we find it in the backup set itself?

```bash
$ pgbackrest repo-get backup/demo/20260922-161237F/pg_data/backup_label.zst| zstd -d
START WAL LOCATION: 0/95000028 (file 000000020000000000000095)
CHECKPOINT LOCATION: 0/95000080
BACKUP METHOD: streamed
BACKUP FROM: primary
START TIME: 2026-09-22 16:12:37 UTC
LABEL: pgBackRest backup started at 2026-09-22 16:12:37.255031+00
START TIMELINE: 2
```

It is there, but it doesn't contain as much information. Let's see if we have one for our latest backup:

```bash
$ pgbackrest repo-get backup/demo/20260922-164401F/pg_data/backup_label.zst |zstd -d
START WAL LOCATION: 0/9A000028 (file 00000002000000000000009A)
CHECKPOINT LOCATION: 0/9A000080
BACKUP METHOD: streamed
BACKUP FROM: primary
START TIME: 2026-09-22 16:44:01 UTC
LABEL: pgBackRest backup started at 2026-09-22 16:44:01.453875+00
START TIMELINE: 2
```

In theory then, PostgreSQL would have just enough information to be able to recover.
Let's try to restore the latest backup on our old primary (`pg1`) to rebuild it as a new standby (which will also allow us to test the backup):

```bash
$ pgbackrest --stanza=demo --type=standby --delta restore
P00   INFO: restore command begin 2.60.0dev: ...
P00   INFO: repo1: restore backup set 20260922-164401F, recovery will start at 2026-09-22 16:44:01
P00   INFO: remove invalid files/links/paths from '/var/lib/pgsql/18/data'
P00   INFO: write updated /var/lib/pgsql/18/data/postgresql.auto.conf
P00   INFO: restore global/pg_control (performed last to ensure aborted restores cannot be started)
P00   INFO: restore size = 1.5GB, file total = 1286
P00   INFO: restore command end: completed successfully
```

* Adjust `postgresql.conf`:

```ini
archive_mode = on
archive_command = 'pgbackrest --stanza=demo archive-push %p'
restore_command = 'pgbackrest --stanza=demo archive-get %f "%p"'
primary_conninfo = 'user=replic_user host=pg2'
```

After starting PostgreSQL, we should now see two nodes connected to our current primary (`pg2`):

```sql
postgres=# SELECT * FROM pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 1831
usesysid         | 16415
usename          | replic_user
application_name | walreceiver
client_addr      | 192.168.121.36
client_hostname  |
client_port      | 41630
backend_start    | 2026-09-23 06:56:19.495963+00
backend_xmin     |
state            | streaming
sent_lsn         | 0/9B000320
write_lsn        | 0/9B000320
flush_lsn        | 0/9B000320
replay_lsn       | 0/9B000320
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2026-09-23 07:02:52.289008+00
-[ RECORD 2 ]----+------------------------------
pid              | 2835
usesysid         | 16415
usename          | replic_user
application_name | walreceiver
client_addr      | 192.168.121.135
client_hostname  |
client_port      | 56634
backend_start    | 2026-09-23 07:02:43.346844+00
backend_xmin     |
state            | streaming
sent_lsn         | 0/9B000320
write_lsn        | 0/9B000320
flush_lsn        | 0/9B000320
replay_lsn       | 0/9B000320
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2026-09-23 07:02:53.397657+00
```

Let's stop `pg2` and promote `pg1` to get back to our initial situation where `pg1` was the primary:

```bash
$ psql -c "SELECT pg_promote();"
 pg_promote
------------
 t
(1 row)

$ psql -c "SELECT pg_switch_wal();"
 pg_switch_wal
---------------
 0/9B0004F0
(1 row)
```

* Before restarting `pg2` to make it a standby server, adjust `postgresql.conf` there:

```ini
archive_mode = on
archive_command = 'pgbackrest --stanza=demo archive-push %p'
restore_command = 'pgbackrest --stanza=demo archive-get %f "%p"'
primary_conninfo = 'user=replic_user host=pg1'
```

* And create the standby signal:

```bash
$ touch 18/data/standby.signal
```

After starting PostgreSQL on `pg2`, we should now see it connected to `pg1`:

```sql
postgres=# SELECT * FROM pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 2843
usesysid         | 16415
usename          | replic_user
application_name | walreceiver
client_addr      | 192.168.121.240
client_hostname  |
client_port      | 46480
backend_start    | 2026-09-23 07:07:58.70969+00
backend_xmin     |
state            | streaming
sent_lsn         | 0/9C000060
write_lsn        | 0/9C000060
flush_lsn        | 0/9C000060
replay_lsn       | 0/9C000060
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2026-09-23 07:08:08.761663+00
```

And `pg3` should still be connected to `pg2`:

```sql
postgres=# SELECT * FROM pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 3017
usesysid         | 16415
usename          | replic_user
application_name | walreceiver
client_addr      | 192.168.121.36
client_hostname  |
client_port      | 57286
backend_start    | 2026-09-23 07:06:04.175795+00
backend_xmin     |
state            | streaming
sent_lsn         | 0/9C000060
write_lsn        | 0/9C000060
flush_lsn        | 0/9C000060
replay_lsn       | 0/9C000060
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2026-09-23 07:08:28.762633+00
```

Let's take a fresh backup on `pg1` to make sure everything's back to normal:

```bash
$ pgbackrest --stanza=demo --type=full backup
P00   INFO: backup command begin 2.60.0dev: ...
P00   INFO: execute backup start:
backup begins after the requested immediate checkpoint completes
P00   INFO: backup start archive = 00000003000000000000009D, lsn = 0/9D000028
P00   INFO: check archive for prior segment 00000003000000000000009C
P00   INFO: execute backup stop and wait for all WAL segments to archive
P00   INFO: backup stop archive = 00000003000000000000009D, lsn = 0/9D000158
P00   INFO: check archive for segment(s) 00000003000000000000009D:00000003000000000000009D
P00   INFO: new backup label = 20260923-070913F
P00   INFO: full backup size = 1.5GB, file total = 1287
P00   INFO: backup command end: completed successfully
```
```bash
$ pgbackrest --stanza=demo info
stanza: demo
  status: ok
  cipher: none

  db (current)
    wal archive min/max (18): 000000010000000000000069/00000003000000000000009D

    full backup: 20260922-150220F
      timestamp start/stop: 2026-09-22 15:02:20+00 / 2026-09-22 15:02:27+00
      wal start/stop: 000000010000000000000069 / 000000010000000000000069
      database size: 1.5GB, database backup size: 1.5GB
      repo1: backup set size: 76.6MB, backup size: 76.6MB

    incr backup: 20260922-150220F_20260922-153628I
      timestamp start/stop: 2026-09-22 15:36:28+00 / 2026-09-22 15:36:29+00
      wal start/stop: 00000001000000000000006E / 00000001000000000000006E
      database size: 1.5GB, database backup size: 3.4MB
      repo1: backup set size: 76.6MB, backup size: 1MB
      backup reference total: 1 full

    full backup: 20260922-161237F
      timestamp start/stop: 2026-09-22 16:12:37+00 / 2026-09-22 16:12:53+00
      wal start/stop: 000000020000000000000095 / 000000020000000000000095
      database size: 1.5GB, database backup size: 1.5GB
      repo1: backup set size: 79.2MB, backup size: 79.2MB

    full backup: 20260922-164401F
      timestamp start/stop: 2026-09-22 16:44:01+00 / 2026-09-22 16:44:55+00
      wal start/stop: 00000002000000000000009A / 00000002000000000000009A
      database size: 1.5GB, database backup size: 1.5GB
      repo1: backup set size: 79.2MB, backup size: 79.2MB

    full backup: 20260923-070913F
      timestamp start/stop: 2026-09-23 07:09:13+00 / 2026-09-23 07:09:19+00
      wal start/stop: 00000003000000000000009D / 00000003000000000000009D
      database size: 1.5GB, database backup size: 1.5GB
      repo1: backup set size: 79.2MB, backup size: 79.2MB
```

---

# Conclusion

Removing the checks allowed us to complete a backup in this experiment, and we successfully restored it. That is a useful result, but it answers a narrower question than whether this workaround is safe to rely on. The new backup did not repair the missing WAL around the earlier promotion. Being able to restore a backup taken after that gap is different from being able to recover an older backup across the timeline switch.

`archive-mode-check=n` does not bypass the check that rejects `archive_mode=off` on the primary. These checks enforce the archiving assumptions pgBackRest relies on. Removing them transfers responsibility for those assumptions to whoever maintains and operates the modified build; one successful PoC does not validate every failure and recovery scenario.

My recommendation remains the same: restore a supported configuration through a planned restart or a controlled switchover to a correctly configured standby. As a reminder, set `archive_mode=on` on promotion candidates beforehand: changing it later requires a restart, while changing `archive_command` only requires a reload. Before promoting a server, make sure its `archive_command` is configured and archiving is ready to work. And, of course, test recovery as well as backup creation :-)
