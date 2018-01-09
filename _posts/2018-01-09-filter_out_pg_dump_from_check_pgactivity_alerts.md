---
layout: post
title: Filter out pg_dump from check_pgactivity alerts
---

check_pgactivity (https://github.com/OPMDG/check_pgactivity) is designed to monitor PostgreSQL clusters from Nagios. It offers many options to measure and monitor useful performance metrics.

Imagine you have a very large database and pg_dump produce abnormal query time alerts. The upcoming release of check_pgactivity offers a way to filter out those alerts !

<!--MORE-->

-----

In the 2.4 version, it will be possible to filter out pg_dump from the `oldest_idlexact` service. Indeed, above PostgreSQL 9.2, the service supports `--exclude` to filter out connections.

The `longest_query` service will also, above PostgreSQL 9.0, support `--exclude` to filter out application names.

Let's see an example.

Launch a pg_dump of a "not-empty" database :

```bash
$ pgbench -i -s 300 blog
$ pg_dump -d blog -f tmp.dump --exclude-table=pgbench_tellers
```

We can see the pg_dump progress with pg_stat_activity :

```
postgres=# SELECT application_name, query, (current_timestamp-state_change) AS elapsed, state, wait_event from pg_stat_activity where application_name = 'pg_dump';
-[ RECORD 1 ]----+---------------------------------------------------------------------
application_name | pg_dump
query            | COPY public.pgbench_accounts (aid, bid, abalance, filler) TO stdout;
elapsed          | 00:01:26.435591
state            | active
wait_event       | ClientWrite
```

The `longest_query` service will raise an alert :

```
$ check_pgactivity --service longest_query -w 1m -c 5m
POSTGRES_LONGEST_QUERY WARNING: blog: 1m31s | 
'blog max'=91s;60;300 'blog avg'=91s;60;300 'blog #queries'=1 
'postgres max'=0s;60;300 'postgres avg'=0s;60;300 'postgres #queries'=1 
'template1 max'=NaNs;60;300 'template1 avg'=NaNs;60;300 'template1 #queries'=0
```

We can remove that alert by adding the filter :

```
$ check_pgactivity --service longest_query -w 1m -c 5m --exclude ^pg_dump
POSTGRES_LONGEST_QUERY OK: 1 running querie(s) | 
'postgres max'=0s;60;300 'postgres avg'=0s;60;300 'postgres #queries'=1 
'template1 max'=NaNs;60;300 'template1 avg'=NaNs;60;300 'template1 #queries'=0 
'blog max'=NaNs;60;300 'blog avg'=NaNs;60;300 'blog #queries'=0
```

After some time, pg_dump goes further :

```
postgres=# SELECT application_name, query, (current_timestamp-state_change) AS elapsed, state, wait_event from pg_stat_activity where application_name = 'pg_dump';
-[ RECORD 1 ]----+-----------------------------------------------------------------------------
application_name | pg_dump
query            | COPY public.pgbench_history (tid, bid, aid, delta, mtime, filler) TO stdout;
elapsed          | 00:02:38.581393
state            | idle in transaction
wait_event       | ClientRead
```

It's now the `oldest_idlexact` service that raise an alert :

```
$ check_pgactivity  --service oldest_idlexact -w 1m -c 5m
POSTGRES_OLDEST_IDLEXACT WARNING: 1 idle transaction(s), oldest idle xact on blog: 2m47s | 
'blog max'=167s;60;300 'blog avg'=167s;60;300 'blog # idle xact'=1 
'postgres max'=NaNs;60;300 'postgres avg'=NaNs;60;300 'postgres # idle xact'=0 
'template0 max'=NaNs;60;300 'template0 avg'=NaNs;60;300 'template0 # idle xact'=0 
'template1 max'=NaNs;60;300 'template1 avg'=NaNs;60;300 'template1 # idle xact'=0
```

We can also remove that alert by adding the filter :

```
$ check_pgactivity  --service oldest_idlexact -w 1m -c 5m --exclude 'pg_dump'
POSTGRES_OLDEST_IDLEXACT OK: 0 idle transaction(s)
```

-----

In addition to those features, the 2.4 release will also contain some bug fixes in `sequences_exhausted` and `backends_status` services.

The new release candidate should be available soon !