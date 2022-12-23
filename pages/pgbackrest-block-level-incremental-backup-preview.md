# Block incremental backup

Block incremental backup is currently a work in progress and even though the primary goal of this kind of feature is usually to save space in the repository by only storing changed parts of a file rather than the entire file, the ongoing implementation is focused on restore performance more than saving space.

This test scenario will first have to look at the backup command by creating some tables containing 10'000'000 rows each and comparing incremental backups after removing 100'000 (1%) of it, using various compression types.

Then, we'll focus on the restore command comparing file-level and block-level incremental backups.

-----

# Backup command test scenario

Let's first initialize the test database:

```bash
sudo -iu postgres
dropdb test && createdb test
/usr/pgsql-15/bin/pgbench -i -s 100 test
psql -d test -c "CREATE TABLE test1 AS TABLE pgbench_accounts;"
psql -d test -c "CREATE TABLE test2 AS TABLE pgbench_accounts;"
psql -d test -c "CREATE TABLE test3 AS TABLE pgbench_accounts;"
psql -d test -c "CREATE TABLE test4 AS TABLE pgbench_accounts;"
psql -d test -c "CREATE TABLE test5 AS TABLE pgbench_accounts;"
psql -d test -c "CREATE TABLE test6 AS TABLE pgbench_accounts;"
psql -d test -c "CREATE TABLE test7 AS TABLE pgbench_accounts;"
psql -d test -c "CREATE TABLE test8 AS TABLE pgbench_accounts;"
psql -d test -c "CREATE TABLE test9 AS TABLE pgbench_accounts;"
psql -d test -c "CREATE TABLE test10 AS TABLE pgbench_accounts;"
psql -d test -c '\dt+'
                                         List of relations
 Schema |       Name       | Type  |  Owner   | Persistence | Access method |  Size   | Description 
--------+------------------+-------+----------+-------------+---------------+---------+-------------
 public | pgbench_accounts | table | postgres | permanent   | heap          | 1281 MB | 
 public | pgbench_branches | table | postgres | permanent   | heap          | 40 kB   | 
 public | pgbench_history  | table | postgres | permanent   | heap          | 0 bytes | 
 public | pgbench_tellers  | table | postgres | permanent   | heap          | 80 kB   | 
 public | test1            | table | postgres | permanent   | heap          | 1281 MB | 
 public | test10           | table | postgres | permanent   | heap          | 1281 MB | 
 public | test2            | table | postgres | permanent   | heap          | 1281 MB | 
 public | test3            | table | postgres | permanent   | heap          | 1281 MB | 
 public | test4            | table | postgres | permanent   | heap          | 1281 MB | 
 public | test5            | table | postgres | permanent   | heap          | 1281 MB | 
 public | test6            | table | postgres | permanent   | heap          | 1281 MB | 
 public | test7            | table | postgres | permanent   | heap          | 1281 MB | 
 public | test8            | table | postgres | permanent   | heap          | 1281 MB | 
 public | test9            | table | postgres | permanent   | heap          | 1281 MB | 
(14 rows)
```

For this test, I'll use a local repository to store the backups.

Take a full backup, remove 1% of the data and take a file-level incremental backup:

```bash
pgbackrest --stanza=my_stanza --type=full backup

psql -d test -c "DELETE FROM test1 WHERE aid between 500000 and 600000;"
psql -d test -c "DELETE FROM test2 WHERE aid between 1500000 and 1600000;"
psql -d test -c "DELETE FROM test3 WHERE aid between 2500000 and 2600000;"
psql -d test -c "DELETE FROM test4 WHERE aid between 3500000 and 3600000;"
psql -d test -c "DELETE FROM test5 WHERE aid between 4500000 and 4600000;"
psql -d test -c "DELETE FROM test6 WHERE aid between 5500000 and 5600000;"
psql -d test -c "DELETE FROM test7 WHERE aid between 6500000 and 6600000;"
psql -d test -c "DELETE FROM test8 WHERE aid between 7500000 and 7600000;"
psql -d test -c "DELETE FROM test9 WHERE aid between 8500000 and 8600000;"
psql -d test -c "DELETE FROM test10 WHERE aid between 9500000 and 9600000;"
psql -d test -c "VACUUM ANALYSE;"

pgbackrest --stanza=my_stanza --type=incr backup
```

Doing the same using the new `repo-block` option:

```bash
pgbackrest --stanza=my_stanza --type=full backup --repo1-block --repo1-bundle

psql -d test -c "DELETE FROM test1 WHERE aid between 300000 and 400000;"
psql -d test -c "DELETE FROM test2 WHERE aid between 1300000 and 1400000;"
psql -d test -c "DELETE FROM test3 WHERE aid between 2300000 and 2400000;"
psql -d test -c "DELETE FROM test4 WHERE aid between 3300000 and 3400000;"
psql -d test -c "DELETE FROM test5 WHERE aid between 4300000 and 4400000;"
psql -d test -c "DELETE FROM test6 WHERE aid between 5300000 and 5400000;"
psql -d test -c "DELETE FROM test7 WHERE aid between 6300000 and 6400000;"
psql -d test -c "DELETE FROM test8 WHERE aid between 7300000 and 7400000;"
psql -d test -c "DELETE FROM test9 WHERE aid between 8300000 and 8400000;"
psql -d test -c "DELETE FROM test10 WHERE aid between 9300000 and 9400000;"
psql -d test -c "VACUUM ANALYSE;"

pgbackrest --stanza=my_stanza --type=incr backup --repo1-block --repo1-bundle
```

-----

## Gzip

By default, pgBackRest uses gzip compression:

```ini
compress-type=gz
compress-level=6
```

Here's the info output containing our backups:

```
full backup: 20221223-095108F
    timestamp start/stop: 2022-12-23 09:51:08 / 2022-12-23 09:52:47
    wal start/stop: 00000001000000B30000000A / 00000001000000B30000003E
    database size: 14GB, database backup size: 14GB
    repo1: backup set size: 547.3MB, backup size: 547.3MB

incr backup: 20221223-095108F_20221223-095726I
    timestamp start/stop: 2022-12-23 09:57:26 / 2022-12-23 09:59:11
    wal start/stop: 00000001000000B8000000BF / 00000001000000B8000000BF
    database size: 14GB, database backup size: 12.5GB
    repo1: backup set size: 547.4MB, backup size: 462.3MB
    backup reference list: 20221223-095108F

full backup: 20221223-095912F
    timestamp start/stop: 2022-12-23 09:59:12 / 2022-12-23 10:00:54
    wal start/stop: 00000001000000B8000000C1 / 00000001000000B8000000C1
    database size: 14GB, database backup size: 14GB
    repo1: backup set size: 553.6MB, backup size: 553.6MB

incr backup: 20221223-095912F_20221223-100110I
    timestamp start/stop: 2022-12-23 10:01:10 / 2022-12-23 10:01:34
    wal start/stop: 00000001000000B8000000CE / 00000001000000B8000000CE
    database size: 14GB, database backup size: 8.5GB
    repo1: backup set size: 241.3MB, backup size: 5.4MB
    backup reference list: 20221223-095912F
```

Let's compare disk space usage of the file-level and block-level incremental backups:

```bash
$ du -hs 20221223-095108F_20221223-095726I/pg_data/base/404338
463M    20221223-095108F_20221223-095726I/pg_data/base/404338

$ du -hs 20221223-095912F_20221223-100110I/pg_data/base/404338
5.3M    20221223-095912F_20221223-100110I/pg_data/base/404338
```

Internally, new fields have been added to the backup manifest to store the blocks map:

```bash
$ cat 20221223-095912F_20221223-100110I/backup.manifest |grep 404338/404361
pg_data/base/404338/404361.1=
    {"bims":7212,"bis":96,
    "checksum":"27fd785e3214eafbd8a89050068caba1fa406e60","checksum-page":true,
    "rck":"8adbc1acf55eb02af6ff3f7680695c7a2147eb12",
    "reference":"20221223-095912F",
    "repo-size":9850013,"size":269213696,"timestamp":1671789267}
pg_data/base/404338/404361_fsm=
    {"bims":78,"bis":16,"bni":2,"bno":13650,
    "checksum":"7471bdd5cb764513c7e8bfbe921a11f303a08c43","checksum-page":true,
    "rck":"0328231665bb8917f36a48c3fde952a1801078fc",
    "repo-size":871,"size":352256,"timestamp":1671789670}
```

-----

## LZ4

Now, using LZ4 compression with its default compress-level:

```ini
compress-type=lz4
compress-level=1
```

Here's the info output containing our backups:

```
full backup: 20221223-101100F
    timestamp start/stop: 2022-12-23 10:11:00 / 2022-12-23 10:11:26
    wal start/stop: 00000001000000BC000000ED / 00000001000000BD00000001
    database size: 14GB, database backup size: 14GB
    repo1: backup set size: 986.0MB, backup size: 986.0MB

incr backup: 20221223-101100F_20221223-101601I
    timestamp start/stop: 2022-12-23 10:16:01 / 2022-12-23 10:16:25
    wal start/stop: 00000001000000C200000089 / 00000001000000C200000089
    database size: 14GB, database backup size: 12.5GB
    repo1: backup set size: 985.9MB, backup size: 817.5MB
    backup reference list: 20221223-101100F

full backup: 20221223-101626F
    timestamp start/stop: 2022-12-23 10:16:26 / 2022-12-23 10:16:53
    wal start/stop: 00000001000000C20000008A / 00000001000000C20000008A
    database size: 14GB, database backup size: 14GB
    repo1: backup set size: 990.8MB, backup size: 990.8MB

incr backup: 20221223-101626F_20221223-101708I
    timestamp start/stop: 2022-12-23 10:17:08 / 2022-12-23 10:17:31
    wal start/stop: 00000001000000C200000097 / 00000001000000C200000097
    database size: 14GB, database backup size: 8.5GB
    repo1: backup set size: 441.6MB, backup size: 9.1MB
    backup reference list: 20221223-101626F
```

Let's compare disk space usage of the file-level and block-level incremental backups:

```bash
$ du -hs 20221223-101100F_20221223-101601I/pg_data/base/404407
818M    20221223-101100F_20221223-101601I/pg_data/base/404407

$ du -hs 20221223-101626F_20221223-101708I/pg_data/base/404407
9.0M    20221223-101626F_20221223-101708I/pg_data/base/404407
```

-----

## Zstandard

Now, using Zstandard compression with its default compress-level:

```
compress-type=zst
compress-level=3
```

Here's the info output containing our backups:

```
full backup: 20221223-102321F
    timestamp start/stop: 2022-12-23 10:23:21 / 2022-12-23 10:23:53
    wal start/stop: 00000001000000C6000000BA / 00000001000000C6000000D0
    database size: 14GB, database backup size: 14GB
    repo1: backup set size: 422.2MB, backup size: 422.2MB

incr backup: 20221223-102321F_20221223-102825I
    timestamp start/stop: 2022-12-23 10:28:25 / 2022-12-23 10:28:53
    wal start/stop: 00000001000000CC00000050 / 00000001000000CC00000050
    database size: 14GB, database backup size: 12.5GB
    repo1: backup set size: 424.4MB, backup size: 353MB
    backup reference list: 20221223-102321F

full backup: 20221223-102854F
    timestamp start/stop: 2022-12-23 10:28:54 / 2022-12-23 10:29:21
    wal start/stop: 00000001000000CC00000052 / 00000001000000CC00000052
    database size: 14GB, database backup size: 14GB
    repo1: backup set size: 374.7MB, backup size: 374.7MB

incr backup: 20221223-102854F_20221223-102937I
    timestamp start/stop: 2022-12-23 10:29:37 / 2022-12-23 10:29:59
    wal start/stop: 00000001000000CC0000005F / 00000001000000CC0000005F
    database size: 14GB, database backup size: 8.5GB
    repo1: backup set size: 169.6MB, backup size: 3.6MB
    backup reference list: 20221223-102854F
```

Let's compare disk space usage of the file-level and block-level incremental backups:

```bash
$ du -hs 20221223-102321F_20221223-102825I/pg_data/base/404476
354M    20221223-102321F_20221223-102825I/pg_data/base/404476

$ du -hs 20221223-102854F_20221223-102937I/pg_data/base/404476
3.6M    20221223-102854F_20221223-102937I/pg_data/base/404476
```

-----

## Backup results comparison table

|     | Full time | size   | Incr time | size   | Block full time | size   | Block Incr time | size   |
|-----|-----------|--------|-----------|--------|-----------------|--------|-----------------|--------|
| gz  | 99s       | 547 MB | 105s      | 462 MB | 103s            | 553 MB | 24s             | 5.4 MB |
| lz4 | 27s       | 986 MB | 25s       | 817 MB | 28s             | 990 MB | 23s             | 9.1 MB |
| zst | 32s       | 422 MB | 29s       | 353 MB | 28s             | 374 MB | 23s             | 3.6 MB |

Obviously, disk-space saving will highly depend on the workload. Gzip is pretty good at text compression but lot slower.
Even though LZ4 is known to be faster, Zstandard provides the best compression rate while saving a lot of time too.

It is important to keep in mind that the main focus of this feature implementation is not reducing the backup time which is here reduced by selecting the best compression algorithm possible.

Zstandard providing very interesting results, we can see that for 1% of our removed data, the file-level incremental backup is only reduced to ~84%, while the block-level is reduced to less than **1%**!

During this test, done on local disks, we can notice that the various compression methods run at the same time for block-level incremental backups because the limiting factor is performing the checksums (reading the data files, not copying or compressing them).

-----

# Restore command test scenario

The first step is to create a database with 80'000'000 rows:

```bash
/usr/pgsql-15/bin/pgbench -i -s 800 test
psql -d test -c '\dt+'
                                         List of relations
 Schema |       Name       | Type  |  Owner   | Persistence | Access method |  Size   | Description
--------+------------------+-------+----------+-------------+---------------+---------+-------------
 public | pgbench_accounts | table | postgres | permanent   | heap          | 10 GB   |
 public | pgbench_branches | table | postgres | permanent   | heap          | 64 kB   |
 public | pgbench_history  | table | postgres | permanent   | heap          | 0 bytes |
 public | pgbench_tellers  | table | postgres | permanent   | heap          | 384 kB  |
(4 rows)
```

Then, we'll take a full backup, remove some rows scattered across the data files of the database and take an incremental backup:

```bash
pgbackrest --stanza=my_stanza --type=full --repo1-retention-full=1 backup

psql -d test -c "DELETE FROM pgbench_accounts WHERE aid between  1000000 and  2000000;"
psql -d test -c "DELETE FROM pgbench_accounts WHERE aid between  9000000 and 10000000;"
psql -d test -c "DELETE FROM pgbench_accounts WHERE aid between 18000000 and 19000000;"
psql -d test -c "DELETE FROM pgbench_accounts WHERE aid between 25000000 and 26000000;"
psql -d test -c "DELETE FROM pgbench_accounts WHERE aid between 33000000 and 34000000;"
psql -d test -c "DELETE FROM pgbench_accounts WHERE aid between 42000000 and 43000000;"
psql -d test -c "DELETE FROM pgbench_accounts WHERE aid between 50000000 and 51000000;"
psql -d test -c "DELETE FROM pgbench_accounts WHERE aid between 59000000 and 60000000;"
psql -d test -c "DELETE FROM pgbench_accounts WHERE aid between 65000000 and 66000000;"
psql -d test -c "DELETE FROM pgbench_accounts WHERE aid between 74000000 and 75000000;"
psql -d test -c "VACUUM ANALYSE;"

pgbackrest --stanza=my_stanza --type=incr backup
```

-----

## File-level

Running the default backup commands gave:

```
full backup: 20221223-124827F
    timestamp start/stop: 2022-12-23 12:48:27 / 2022-12-23 12:48:53
    wal start/stop: 00000001000000D500000045 / 00000001000000D500000045
    database size: 11.7GB, database backup size: 11.7GB
    repo1: backup set size: 545.6MB, backup size: 545.6MB

incr backup: 20221223-124827F_20221223-124956I
    timestamp start/stop: 2022-12-23 12:49:56 / 2022-12-23 12:50:28
    wal start/stop: 00000001000000D600000018 / 00000001000000D600000018
    database size: 11.7GB, database backup size: 11.7GB
    repo1: backup set size: 546.3MB, backup size: 542.4MB
    backup reference list: 20221223-124827F
```

From the storage point-of-view:

```
# 20221223-124827F
29M     404618.1.zst
184K    404618.10.zst
29M     404618.2.zst
29M     404618.3.zst
29M     404618.4.zst
29M     404618.5.zst
29M     404618.6.zst
29M     404618.7.zst
29M     404618.8.zst
29M     404618.9.zst
29M     404618.zst

# 20221223-124827F_20221223-124956I
29M     404618.1.zst
29M     404618.2.zst
29M     404618.3.zst
29M     404618.4.zst
29M     404618.5.zst
29M     404618.6.zst
29M     404618.7.zst
29M     404618.8.zst
29M     404618.9.zst
29M     404618.zst
```

The restore commands:

```bash
$ pgbackrest --stanza=my_stanza restore --pg1-path=/tmp/restored_data --set=20221223-124827F
...
P00   INFO: restore command end: completed successfully (24190ms)

$ pgbackrest --stanza=my_stanza restore --pg1-path=/tmp/restored_data --set=20221223-124827F_20221223-124956I --delta
...
P00   INFO: restore command end: completed successfully (25258ms)
```

-----

## Block-level

Running the backup commands using the `--repo1-block --repo1-bundle` option gave:

```
full backup: 20221223-125256F
    timestamp start/stop: 2022-12-23 12:52:56 / 2022-12-23 12:53:24
    wal start/stop: 00000001000000D800000082 / 00000001000000D800000082
    database size: 11.7GB, database backup size: 11.7GB
    repo1: backup set size: 512.5MB, backup size: 512.5MB

incr backup: 20221223-125256F_20221223-125430I
    timestamp start/stop: 2022-12-23 12:54:30 / 2022-12-23 12:54:56
    wal start/stop: 00000001000000D900000055 / 00000001000000D900000055
    database size: 11.7GB, database backup size: 11.7GB
    repo1: backup set size: 70.8MB, backup size: 66.9MB
    backup reference list: 20221223-125256F
```

Let's have a look at the storage size:

```
# 20221223-125256F
25M     404648.1.pgbi
164K    404648.10.pgbi
25M     404648.2.pgbi
25M     404648.3.pgbi
25M     404648.4.pgbi
25M     404648.5.pgbi
25M     404648.6.pgbi
25M     404648.7.pgbi
25M     404648.8.pgbi
25M     404648.9.pgbi
25M     404648.pgbi

# 20221223-125256F_20221223-125430I
3.1M    404648.1.pgbi
3.2M    404648.2.pgbi
3.1M    404648.3.pgbi
3.1M    404648.4.pgbi
3.1M    404648.5.pgbi
3.1M    404648.6.pgbi
3.1M    404648.7.pgbi
3.1M    404648.8.pgbi
3.2M    404648.9.pgbi
3.1M    404648.pgbi
```

Let's first restore the full backup and then the incremental one:

```bash
$ pgbackrest --stanza=my_stanza restore --pg1-path=/tmp/restored_data --set=20221223-125256F
...
P00   INFO: restore command end: completed successfully (23564ms)

$ pgbackrest --stanza=my_stanza restore --pg1-path=/tmp/restored_data --set=20221223-125256F_20221223-125430I --delta
...
P00   INFO: restore command end: completed successfully (23013ms)
```

-----

## Restore results comparison table

* Local MinIO instance:

|             | Full time | Incr time | Incr file size | Full restore time | Incr restore time |
|-------------|-----------|-----------|----------------|-------------------|-------------------|
| file-level  | 26s       | 32s       |  29 MB         | 24s               | 25s               |
| block-level | 28s       | 27s       | 3.2 MB         | 23s               | 23s               |

Obviously here on local disks, the network/disk latency is pretty low and we don't see much differences even though delta restores should be faster since we are able to pull individual blocks out of the repository and update an existing file rather than recopying the entire file.

* Let's try with an Azure container:

|             | Full time               | Incr time | Full restore time | Incr restore time  |
|-------------|-------------------------|-----------|-------------------|--------------------|
| file-level  | 20m 30s, 9m 45s, 11m 7s | 13m 8s    | 2m 47s, 58s, 56s  | 1m 19s, 52s, 1m 7s |
| block-level | 10m 24s, 8m 56s, 9m 26s | 2m 5s     | 1m 7s, 50s, 1m 6s | 31s, 36s, 28s      |

Even with the time variation due to the remote storage access, we easily notice that with the size to upload/download being way smaller, the incremental backup are restores are also way faster.

-----

# Conclusion

Depending on the actual workload, this new feature can really save a lot of disk-space in the repository, which can also speed-up the backup command because we'll only send the blocks modified to the repository, not the entire data file. That will also be reflected in delta restores, fetching less data from the repository and updating existing files.
