---
layout: post
title: pgBackRest preview - Bundle files in the repository during backup
date: 2022-02-23 09:00:00 +0100
---

A nice new feature has been added on 14 Feb 2022: **Bundle files in the repository during backup**.
Details can be found in commits [34d6495](https://github.com/pgbackrest/pgbackrest/commit/34d649579eb3bd1530aa99f0ed1879e7d3125424) and [efc09db](https://github.com/pgbackrest/pgbackrest/commit/efc09db7b9ece6e7b7f92538d56d6ab7b9798f8f).

This feature combines smaller files during backup to reduce the number of files written to the repository (enabled with `--bundle`).
Files are batched up to `--bundle-size` and then compressed/encrypted individually and stored sequentially in the bundle.
`--bundle-limit` limits which files can be added to bundles. Files larger than this size will be stored separately.
On backup [resume](https://pgbackrest.org/configuration.html#section-backup/option-resume), the bundles are removed and any remaining file is eligible to be resumed.

> **IMPORTANT NOTE**: this feature is still experimental and will not be released in pgBackRest _2.38_.

<!--MORE-->

-----

# Feature testing

This feature works best when the cluster to backup contains a lot of small files.

Using a simple bash script, let's generate plenty of small tables containing only a few rows:

```bash
export NB_ROWS=1
export NB_TABLES=100000
export TABLE_NAME='test'

sudo -iu postgres dropdb --if-exists $TABLE_NAME
sudo -iu postgres createdb $TABLE_NAME

for i in $(seq 1 $NB_TABLES); do
    sudo -iu postgres psql -d $TABLE_NAME -c "CREATE TABLE IF NOT EXISTS t$i (id int);"
    sudo -iu postgres psql -d $TABLE_NAME -c "INSERT INTO t1 SELECT generate_series(0, $NB_ROWS);"
done
sudo -iu postgres psql -d $PGDATABASE -c "VACUUM ANALYSE;"
sudo -iu postgres psql -d $PGDATABASE -c "CHECKPOINT;"

echo "--Nb of files in $PGDATA"
sudo -iu postgres ls -1RL $PGDATA | wc -l
```

Default values have been set for `bundle-size` (*20MiB*) and `bundle-limit` (*2MiB*).

Let's then take backups varying those values and compare the backup times:

1. --no-bundle
2. --bundle --bundle-limit=2MiB --bundle-size=10MiB
3. --bundle --bundle-limit=2MiB --bundle-size=20MiB
4. --bundle --bundle-limit=5MiB --bundle-size=50MiB
5. --bundle --bundle-limit=10MiB --bundle-size=100MiB
6. --bundle --bundle-limit=100MiB --bundle-size=1GiB

-----

## Results

| Nb of files in PGDATA | full backup size | no-bundle | limit=2MiB<br/>size=10MiB | limit=2MiB<br/>size=20MiB | limit=5MiB<br/>size=50MiB | limit=10MiB<br/>size=100MiB | limit=100MiB<br/>size=1GiB |
|-----------------------|------------------|-----------|---------------------------|---------------------------|---------------------------|-----------------------------|----------------------------|
| 11369                 | 14.7GB           | 249s      | 251s                      | 256s                      | 263s                      | 273s                        | 272s                       |
| 101361                | 15.2GB           | 335s      | 279s                      | 268s                      | 266s                      | 271s                        | 264s                       |
| 1115512               | 17.8GB           | 1264s     | 635s                      | 619s                      | 630s                      | 622s                        | 612s                       |

Next to all the small tables, a simple pgbench database has been generated to increase the cluster size for this test.
The backups have been taken locally and for the last run, all the small tables have been spread across several databases.

```sql
       Name       |  Size
------------------+---------
 pgbackrest-test  | 640 MB
 pgbackrest-test2 | 640 MB
 pgbackrest-test3 | 640 MB
 pgbackrest-test4 | 640 MB
 pgbackrest-test5 | 640 MB
 pgbench          | 15 GB
```

Given this feature might be even better on object-stores, let's drop the pgbench data and compare local backups with S3 storage.

| Nb of files in PGDATA | full backup size | no-bundle | limit=2MiB<br/>size=10MiB | limit=2MiB<br/>size=20MiB | limit=5MiB<br/>size=50MiB | limit=10MiB<br/>size=100MiB | limit=100MiB<br/>size=1GiB |
|-----------------------|------------------|-----------|---------------------------|---------------------------|---------------------------|-----------------------------|----------------------------|
| 1115484 (local)       | 3.1GB            | 945s      | 170s                      | 166s                      | 169s                      | 168s                        | 167s                       |
| 1115484 (S3)          | 3.1GB            | 6256s     | 1370s                     | 1081s                     | 1064s                     | 1370s                       | 1221s                      |

Remark: all the tests have been run using `max-processes=2` but to make S3 backups run faster, the last test used `max-processes=12`.

-----

# Conclusion

As expected, this brand new feature helps improving the backup performance when a lot a small files have to be copied.
