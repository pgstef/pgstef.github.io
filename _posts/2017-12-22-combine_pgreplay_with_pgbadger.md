---
layout: post
title: combine pgreplay with pgBadger
---

pgreplay, written by Laurenz Albe, reads a PostgreSQL log file, extracts the
SQL statements and executes them in the same order and with the original
timing against a PostgreSQL database.

While pgreplay will find out if your database application will encounter
performance problems, it does not provide a lot of help in the analysis of
the cause of these problems.  Combine pgreplay with a specialized analysis
program like pgBadger (https://github.com/dalibo/pgbadger) for that.

<!--MORE-->

-----

In case of a PostgreSQL major version update, it may be useful to replay a test scenario on the new version.

The test case bellow will create a scenario with version 9.6 and replay it on version 10.

On a CentOS 6 server, let's assume a newly installed PostgreSQL 9.6 instance :

```bash
# yum install https://download.postgresql.org/pub/repos/yum/9.6/redhat/rhel-6-x86_64/pgdg-centos96-9.6-3.noarch.rpm
# yum install postgresql96-server
# service postgresql-9.6 initdb
```

Usual `postgresql.conf` configuration for pgBadger :

```
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
lc_messages='C'
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0
log_autovacuum_min_duration = 0
```

Add the pgreplay specific part :

```
log_destination = 'csvlog'
log_min_messages = error
log_min_error_statement = log
log_statement = 'all'
```

Start service :

```bash
# service postgresql-9.6 start
```

Let's initialize a `bench` database with 1.000.000 tuples in the biggest table and save it :

```bash
# su - postgres
$ export PATH=/usr/pgsql-9.6/bin:$PATH
$ createdb bench
$ pgbench -i -s 10 bench
$ pg_dump -d bench -Fc -f bench.dump
```

Let's now create a test scenario using pgbench :

```bash
$ pgbench -c 1 -T 300 bench
```

Install pgreplay :

```bash
# yum groupinstall "Development tools"
# yum install postgresql96-devel
# wget https://github.com/laurenz/pgreplay/archive/PGREPLAY_1_3_0.tar.gz
# tar -xzf PGREPLAY_1_3_0.tar.gz
# cd pgreplay-PGREPLAY_1_3_0/
# ./configure --with-postgres=/usr/pgsql-9.6/bin/
# make
# make install
```

Generate a the replay file :

```bash
$ pgreplay -f -c -o pgreplay.file /var/lib/pgsql/9.6/data/pg_log/postgresql-*.csv 
```

Lets install PostgreSQL 10 :

```bash
# service postgresql-9.6 stop
# yum install https://download.postgresql.org/pub/repos/yum/10/redhat/rhel-6-x86_64/pgdg-centos10-10-1.noarch.rpm
# yum install postgresql10-server
# service postgresql-10 initdb
# service postgresql-10 start
```

Import the database we saved earlier and replay the scenario :

```bash
$ export PATH=/usr/pgsql-10/bin:$PATH
$ createdb bench
$ pg_restore -d bench bench.dump
$ pgreplay -r -j pgreplay.file
```

-----

Because both pgreplay and pgBadger target performance questions, it would be
nice if they could operate on the same log files.  But pgBadger needs the
statement to be logged along with its duration, which is logged at the end of
query execution, whereas pgreplay needs it at execution start time, so they
cannot use the same log file.

If you use CSV logging, there is a workaround with which you can turn a
log file for pgBadger into a log file for pgreplay.

Let's get back to our test case on version 9.6.

In the `postgresql.conf`, change :

```
log_statement = 'none'
log_min_duration_statement = 0
```

Stop version 10 if running, clean the logs and start version 9.6 :

```bash
# service postgresql-10 stop
# rm -rf /var/lib/pgsql/9.6/data/pg_log/*
# service postgresql-9.6 start
```

Play a test scenario using pgbench :

```bash
$ pgbench -c 1 -T 300 bench
```

To make the log compatible for pgreplay, we'll need to :

* Create a postgres_log table

```sql
CREATE TABLE postgres_log
(
  log_time timestamp(3) with time zone,
  user_name text,
  database_name text,
  process_id integer,
  connection_from text,
  session_id text,
  session_line_num bigint,
  command_tag text,
  session_start_time timestamp with time zone,
  virtual_transaction_id text,
  transaction_id bigint,
  error_severity text,
  sql_state_code text,
  message text,
  detail text,
  hint text,
  internal_query text,
  internal_query_pos integer,
  context text,
  query text,
  query_pos integer,
  location text,
  application_name text,
  PRIMARY KEY (session_id, session_line_num)
);
```

* Import the log file into this table

```sql
COPY postgres_log FROM '/var/lib/pgsql/9.6/data/pg_log/postgresql-log.csv' WITH CSV;
```

* Update some data if needed and export the content to a new csv file 

```sql
UPDATE postgres_log SET
  log_time = log_time - CAST(substring(message FROM E'\\d+.\\d* ms') AS interval),
  message = regexp_replace(message, E'^duration: \\d+.\\d* ms  ', '')
WHERE error_severity = 'LOG' AND message ~ E'^duration: \\d+.\\d* ms  ';
```

```sql
COPY (SELECT
        to_char(log_time, 'YYYY-MM-DD HH24:MI:SS.MS TZ'),
        user_name, database_name, process_id, connection_from,
        session_id, session_line_num, command_tag, session_start_time,
        virtual_transaction_id, transaction_id, error_severity,
        sql_state_code, message, detail, hint, internal_query,
        internal_query_pos, context, query, query_pos, location, application_name
      FROM postgres_log ORDER BY log_time, session_line_num)
  TO '/var/lib/pgsql/9.6/data/pg_log/pgreplay.csv' WITH CSV;
```

Finally, generate the replay file and play it :

```bash
$ pgreplay -f -c -o pgreplay.file /var/lib/pgsql/9.6/data/pg_log/pgreplay.csv
$ pgreplay -r -j pgreplay.file
```

If everything goes correctly, you'll get some replay statistics :

```
Speed factor for replay: 1.000
Total run time: 3 minutes 46.340 seconds
Maximum lag behind schedule: 2 seconds
Calls to the server: 753096
(3327.273 calls per second)
Total number of connections: 2
Maximum number of concurrent connections: 1
Average number of concurrent connections: 1.000
Average session idle percentage: 38.650%
SQL statements executed: 753092
(0 or 0.000% of these completed with error)
Maximum number of concurrent SQL statements: 1
Average number of concurrent SQL statements: 0.613
Average SQL statement duration: 0.000 seconds
Maximum SQL statement duration: 0.099 seconds
Statement duration histogram:
  0    to 0.02 seconds: 99.997%
  0.02 to 0.1  seconds: 0.003%
  0.1  to 0.5  seconds: 0.000%
  0.5  to 2    seconds: 0.000%
     over 2    seconds: 0.000%
```

-----

In conclusion, pgreplay might be handy but need some trials to validate your procedure.

Hope you enjoyed this first blog post :-)