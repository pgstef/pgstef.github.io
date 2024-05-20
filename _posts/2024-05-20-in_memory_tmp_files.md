---
layout: post
title: In-memory disk for PostgreSQL temporary files
# To generate formatted date, run LC_TIME=C date +"%F %T %z"
date: 2024-05-20 09:30:00 +0200
---

Recently, while debugging a performance issue of a `CREATE INDEX` operation, I was reminded that PostgreSQL might produce temporary files when executing a parallel query, including parallel index creation, because each worker process has its own memory and might need to use disk space for sorting or hash tables.

Thanks to Peter Geoghegan answering [this pgsql-admin](https://www.postgresql.org/message-id/flat/13fa6c59-16bc-9945-f913-2e0c58d34d12@portavita.eu) email thread.

So, in order to try to speed up that index creation, I thought it would be beneficial to move those temporary files directly into memory using a tmpfs and wanted to test that theory, writing this blog post :-)

<!--MORE-->

-----

# Example

Let's first enable logging of the temporary files to evaluate the change we're planning to make:

```sql
ALTER SYSTEM SET log_temp_files TO 0;
ALTER SYSTEM SET log_min_duration_statement TO 0;
SELECT pg_reload_conf();
```

Create a test database using `pgbench` and an index on the `pgbench_accounts` table:

```bash
$ createdb bench
$ /usr/pgsql-16/bin/pgbench -i -s 100 bench
$ psql bench -c "CREATE INDEX ON pgbench_accounts (aid, filler);"
```

We can see from the logs the time it took to build the index (~5.9s) with temporary files involved:

```
LOG:  temporary file: path "base/pgsql_tmp/pgsql_tmp28501.0.fileset/0.0", size 541376512
STATEMENT:  CREATE INDEX ON pgbench_accounts (aid, filler);
LOG:  temporary file: path "base/pgsql_tmp/pgsql_tmp28501.0.fileset/1.0", size 541024256
STATEMENT:  CREATE INDEX ON pgbench_accounts (aid, filler);
LOG:  duration: 5936.468 ms  statement: CREATE INDEX ON pgbench_accounts (aid, filler);
```

Let's try to re-create the index with a higher `maintenance_work_mem` to get rid of the temporary files:

```sql
DROP INDEX pgbench_accounts_aid_filler_idx;
SET maintenance_work_mem TO '2GB';
CREATE INDEX ON pgbench_accounts (aid, filler);
```

But the temporary files are not gone, as we expected given the comment on the pgsql-admin thread mentioned above.

```
LOG:  temporary file: path "base/pgsql_tmp/pgsql_tmp28501.10.fileset/0.0", size 365936640
STATEMENT:  CREATE INDEX ON pgbench_accounts (aid, filler);
LOG:  temporary file: path "base/pgsql_tmp/pgsql_tmp28501.10.fileset/2.0", size 354754560
STATEMENT:  CREATE INDEX ON pgbench_accounts (aid, filler);
LOG:  temporary file: path "base/pgsql_tmp/pgsql_tmp28501.10.fileset/1.0", size 361439232
STATEMENT:  CREATE INDEX ON pgbench_accounts (aid, filler);
LOG:  duration: 4541.701 ms  statement: CREATE INDEX ON pgbench_accounts (aid, filler);
```

So, let's disable parallel query execution to check:

```sql
DROP INDEX pgbench_accounts_aid_filler_idx;
SET maintenance_work_mem TO '2GB';
SET max_parallel_workers TO 0;
CREATE INDEX ON pgbench_accounts (aid, filler);
```

And voilà! The temporary files are gone:

```
LOG:  duration: 4348.098 ms  statement: CREATE INDEX ON pgbench_accounts (aid, filler);
```

We got approximately the same creation time ~4.5s vs ~4.3s.
But what could we do to use both memory and parallel execution? Moving those temporary files into memory with tmpfs directories!

## Configure PostgreSQL to use tmpfs directory

To move temporary files to memory, we'll need to use the `temp_tablespaces` setting which requires to create a tablespace.
So, we'll first create that tablespace and then move it to a tmpfs location.

Let's start by creating a root directory for our tablespace:

```bash
sudo mkdir /var/pgsql_tmp
```

Now, we have to create a tablespace for those PostgreSQL temporary files:

```sql
CREATE TABLESPACE tbstmp location '/var/pgsql_tmp';
```

PostgreSQL will create a sub-directory in `/var/pgsql_tmp` and we'll need to make that one permanent if we don't want to re-create the tablespace after each and every reboot:

```bash
$ ls /var/pgsql_tmp/
PG_16_202307071
```

Finally, to make the tmpfs mount persistent, add it to `/etc/fstab`:

```
tmpfs   /var/pgsql_tmp/PG_16_202307071  tmpfs   rw,size=2G,uid=postgres,gid=postgres    0 0
```

In this example, `size=2G` configures the tmpfs instance to use up to 2GB of RAM.

After configuring `/etc/fstab`, reload systemd and mount the tmpfs instance:

```bash
sudo systemctl daemon-reload
sudo mount /var/pgsql_tmp/PG_16_202307071
```

Back to our initial example, let's now try to set `temp_tablespaces` and create the index again:

```sql
DROP INDEX pgbench_accounts_aid_filler_idx;
SET maintenance_work_mem TO '2GB';
RESET max_parallel_workers;
SET temp_tablespaces TO 'tbstmp';
CREATE INDEX ON pgbench_accounts (aid, filler);
```

The temporary files are using the in-memory tmpfs directory, which indeed speeds up the index creation (to ~3.9s):

```
LOG:  temporary file: path "pg_tblspc/16448/PG_16_202307071/pgsql_tmp/pgsql_tmp28501.11.fileset/1.0", size 361865216
STATEMENT:  CREATE INDEX ON pgbench_accounts (aid, filler);
LOG:  temporary file: path "pg_tblspc/16448/PG_16_202307071/pgsql_tmp/pgsql_tmp28501.11.fileset/0.0", size 364134400
STATEMENT:  CREATE INDEX ON pgbench_accounts (aid, filler);
LOG:  temporary file: path "pg_tblspc/16448/PG_16_202307071/pgsql_tmp/pgsql_tmp28501.11.fileset/2.0", size 356122624
STATEMENT:  CREATE INDEX ON pgbench_accounts (aid, filler);
LOG:  duration: 3977.606 ms  statement: CREATE INDEX ON pgbench_accounts (aid, filler);
```

To make that change permanent for all the temporary files, change the `temp_tablespaces` setting system-wide:

```sql
ALTER SYSTEM SET temp_tablespaces TO 'tbstmp';
```

Finally, reload the PostgreSQL configuration for the changes to take effect:

```sql
postgres=# SELECT pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres=# SELECT * FROM pg_settings WHERE name = 'temp_tablespaces';
-[ RECORD 1 ]---+-------------------------------------------------------------------
name            | temp_tablespaces
setting         | tbstmp
unit            |
category        | Client Connection Defaults / Statement Behavior
short_desc      | Sets the tablespace(s) to use for temporary tables and sort files.
extra_desc      |
context         | user
vartype         | string
source          | session
min_val         |
max_val         |
enumvals        |
boot_val        |
reset_val       | "tbstmp"
sourcefile      |
sourceline      |
pending_restart | f
```

---

# Conclusion

By following these steps, you can configure a tmpfs in-memory directory for PostgreSQL temporary files, potentially improving performance for certain operations!
Obviously, you'll need to adjust the tmpfs size according to your system's capacity and requirements...

So, what do you think about this move?
