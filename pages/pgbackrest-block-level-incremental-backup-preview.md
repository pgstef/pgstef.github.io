# Block incremental backup

Block incremental backup is currently a work in progress and even though the primary goal of this kind of feature is usually to save space in the repository by only storing changed parts of a file rather than the entire file, the ongoing implementation is focused on restore performance more than saving space.

This test scenario will create some tables containing 10'000'000 rows each and comparing incremental backups after removing 100'000 (1%) of it, using various compression types.

FIXME - restore part

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
pgbackrest --stanza=my_stanza --type=full backup --repo1-block

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

pgbackrest --stanza=my_stanza --type=incr backup --repo1-block
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
full backup: 20221121-101408F
    timestamp start/stop: 2022-11-21 10:14:08 / 2022-11-21 10:16:52
    wal start/stop: 0000000100000007000000F8 / 000000010000000800000093
    database size: 14.0GB, database backup size: 14.0GB
    repo1: backup set size: 547.0MB, backup size: 547.0MB

incr backup: 20221121-101408F_20221121-102631I
    timestamp start/stop: 2022-11-21 10:26:31 / 2022-11-21 10:27:48
    wal start/stop: 000000010000000A00000012 / 000000010000000A00000012
    database size: 14GB, database backup size: 11.5GB
    repo1: backup set size: 547MB, backup size: 425.2MB
    backup reference list: 20221121-101408F, 20221121-101408F_20221121-102631I

full backup: 20221121-103008F
    timestamp start/stop: 2022-11-21 10:30:08 / 2022-11-21 10:31:43
    wal start/stop: 000000010000000A00000014 / 000000010000000A00000014
    database size: 14GB, database backup size: 14GB
    repo1: backup set size: 553.3MB, backup size: 553.3MB

incr backup: 20221121-103008F_20221121-103551I
    timestamp start/stop: 2022-11-21 10:35:51 / 2022-11-21 10:36:14
    wal start/stop: 000000010000000A00000021 / 000000010000000A00000021
    database size: 14GB, database backup size: 8.5GB
    repo1: backup set size: 241.2MB, backup size: 5.3MB
    backup reference list: 20221121-103008F, 20221121-103008F_20221121-103551I
```

Let's compare disk space usage of the file-level and block-level incremental backups:

```bash
$ du -hs 20221121-101408F_20221121-102631I/pg_data/base/16388
426M	20221121-101408F_20221121-102631I/pg_data/base/16388

$ du -hs 20221121-103008F_20221121-103551I/pg_data/base/16388
5.5M	20221121-103008F_20221121-103551I/pg_data/base/16388
```

Internally, new fields have been added to the backup manifest to store the blocks map:

```bash
$ cat 20221121-101408F_20221121-102631I/backup.manifest | grep 16451
pg_data/base/16388/16451=
    {"checksum":"0105f13b7e717214c49d03dc1dfa1f5eb13a86b1","checksum-page":true,
    "repo-size":38726325,"size":1073741824,"timestamp":1669026392}
pg_data/base/16388/16451.1=
    {"checksum":"72a63d36e49e540602ad41d42d630c433be93414","checksum-page":true,
    "repo-size":9709769,"size":269213696,"timestamp":1669025951}
...

$ cat 20221121-103008F_20221121-103551I/backup.manifest | grep 16451
pg_data/base/16388/16451=
    {"bims":21528,"bis":128,
    "checksum":"807df11a61b071d3f5aeb4d9fa2879d718af64e2","checksum-page":true,
    "repo-size":553866,"size":1073741824,"timestamp":1669026951}
pg_data/base/16388/16451.1=
    {"bims":7212,"bis":96,
    "checksum":"72a63d36e49e540602ad41d42d630c433be93414","checksum-page":true,
    "reference":"20221121-103008F",
    "repo-size":9848182,"size":269213696,"timestamp":1669025951}
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
full backup: 20221121-151247F
    timestamp start/stop: 2022-11-21 15:12:47 / 2022-11-21 15:13:12
    wal start/stop: 000000010000001C00000086 / 000000010000001C0000009C
    database size: 14.0GB, database backup size: 14.0GB
    repo1: backup set size: 985.4MB, backup size: 985.4MB

incr backup: 20221121-151247F_20221121-152137I
    timestamp start/stop: 2022-11-21 15:21:37 / 2022-11-21 15:22:01
    wal start/stop: 0000000100000021000000C2 / 0000000100000021000000C2
    database size: 14GB, database backup size: 12.5GB
    repo1: backup set size: 985.6MB, backup size: 817.6MB
    backup reference list: 20221121-151247F, 20221121-151247F_20221121-152137I

full backup: 20221121-152240F
    timestamp start/stop: 2022-11-21 15:22:40 / 2022-11-21 15:23:08
    wal start/stop: 0000000100000021000000C4 / 0000000100000021000000C4
    database size: 14GB, database backup size: 14GB
    repo1: backup set size: 990.7MB, backup size: 990.7MB

incr backup: 20221121-152240F_20221121-152356I
    timestamp start/stop: 2022-11-21 15:23:56 / 2022-11-21 15:24:18
    wal start/stop: 0000000100000021000000D1 / 0000000100000021000000D1
    database size: 14GB, database backup size: 8.5GB
    repo1: backup set size: 441.4MB, backup size: 9.1MB
    backup reference list: 20221121-152240F, 20221121-152240F_20221121-152356I
```

Let's compare disk space usage of the file-level and block-level incremental backups:

```bash
$ du -hs 20221121-151247F_20221121-152137I/pg_data/base/24649
818M    20221121-151247F_20221121-152137I/pg_data/base/24649

$ du -hs 20221121-152240F_20221121-152356I/pg_data/base/24649
9.3M    20221121-152240F_20221121-152356I/pg_data/base/24649
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
full backup: 20221121-145505F
    timestamp start/stop: 2022-11-21 14:55:05 / 2022-11-21 14:55:36
    wal start/stop: 0000000100000013000000C6 / 0000000100000013000000DD
    database size: 14.0GB, database backup size: 14.0GB
    repo1: backup set size: 421.9MB, backup size: 421.9MB

incr backup: 20221121-145505F_20221121-150035I
    timestamp start/stop: 2022-11-21 15:00:35 / 2022-11-21 15:01:03
    wal start/stop: 000000010000001800000037 / 000000010000001800000037
    database size: 14GB, database backup size: 12.5GB
    repo1: backup set size: 424MB, backup size: 352.9MB
    backup reference list: 20221121-145505F, 20221121-145505F_20221121-150035I

full backup: 20221121-150151F
    timestamp start/stop: 2022-11-21 15:01:51 / 2022-11-21 15:02:21
    wal start/stop: 000000010000001800000039 / 000000010000001800000039
    database size: 14GB, database backup size: 14GB
    repo1: backup set size: 374.5MB, backup size: 374.5MB

incr backup: 20221121-150151F_20221121-150411I
    timestamp start/stop: 2022-11-21 15:04:11 / 2022-11-21 15:04:33
    wal start/stop: 000000010000001800000046 / 000000010000001800000046
    database size: 14GB, database backup size: 8.5GB
    repo1: backup set size: 169.7MB, backup size: 3.6MB
    backup reference list: 20221121-150151F, 20221121-150151F_20221121-150411I
```

Let's compare disk space usage of the file-level and block-level incremental backups:

```bash
$ du -hs 20221121-145505F_20221121-150035I/pg_data/base/24580
354M	20221121-145505F_20221121-150035I/pg_data/base/24580

$ du -hs 20221121-150151F_20221121-150411I/pg_data/base/24580
3.8M	20221121-150151F_20221121-150411I/pg_data/base/24580
```

-----

## Backup results comparison table

|     | Full time | size   | Incr time | size   | Block full time | size   | Block Incr time | size   |
|-----|-----------|--------|-----------|--------|-----------------|--------|-----------------|--------|
| gz  | 164s      | 547 MB | 77s       | 425 MB | 95s             | 553 MB | 23s             | 5.3 MB |
| lz4 | 25s       | 985 MB | 24s       | 817 MB | 28s             | 990 MB | 22s             | 9.1 MB |
| zst | 31s       | 422 MB | 28s       | 353 MB | 30s             | 374 MB | 22s             | 3.6 MB |

Obviously, disk-space saving will highly depend on the workload. Gzip is very-good at text compression but lot slower.
Even though LZ4 is known to be faster, Zstandard provides the best compression rate while saving a lot of time too.

It is important to keep in mind that the main focus of this feature implementation is not reducing the backup time which is here reduced by selecting the best compression algorithm possible.

Zstandard providing very interesting results, we can see that for 1% of our removed data, the file-level incremental backup is only reduced to ~87%, while the block-level is reduced to less than **1%**!

-----

# Restore command test scenario

FIXME

-----

# Conclusion

FIXME
