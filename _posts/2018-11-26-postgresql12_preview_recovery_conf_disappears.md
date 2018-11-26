---
layout: post
title: PostgreSQL 12 preview - recovery.conf disappears
draft: true
---

To be able in the future to have (for example) more dynamic reconfiguration 
around recovery (changing the primary_conninfo or other settings at runtime,...), 
PostgreSQL needs the infrastructure to deal with that. 

The first step, mostly to avoid having to duplicate the GUC logic, results on 
the following patch.

<!--MORE-->

-----

# [](#introduction)Introduction

On 25th of November 2018, Peter Eisentraut committed 
[Integrate recovery.conf into postgresql.conf](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commitdiff;h=2dedf4d9a899b36d1a8ed29be5efbd1b31a8fe85):

```
recovery.conf settings are now set in postgresql.conf (or other GUC
sources).  Currently, all the affected settings are PGC_POSTMASTER;
this could be refined in the future case by case.

Recovery is now initiated by a file recovery.signal.  Standby mode is
initiated by a file standby.signal.  The standby_mode setting is
gone.  If a recovery.conf file is found, an error is issued.

The trigger_file setting has been renamed to promote_trigger_file as
part of the move.

The documentation chapter "Recovery Configuration" has been integrated
into "Server Configuration".

pg_basebackup -R now appends settings to postgresql.auto.conf and
creates a standby.signal file.

Author: Fujii Masao <masao.fujii@gmail.com>
Author: Simon Riggs <simon@2ndquadrant.com>
Author: Abhijit Menon-Sen <ams@2ndquadrant.com>
Author: Sergei Kornilov <sk@zsrv.org>
Discussion: https://www.postgresql.org/message-id/flat/607741529606767@web3g.yandex.ru/
```

Let's compare a simple example between PostgreSQL 11 and 12.

-----

## [](#initialize-replication-on-v11)Initialize replication on v11

With a default postgresql11-server installation on CentOS 7, let's start
archiving on our primary server:

```
$ mkdir /var/lib/pgsql/11/archives
$ echo "archive_mode = 'on'" >> /var/lib/pgsql/11/data/postgresql.conf
$ echo "archive_command = 'cp %p /var/lib/pgsql/11/archives/%f'" \
>> /var/lib/pgsql/11/data/postgresql.conf
# systemctl start postgresql-11.service
```

Check if the archiver process is working:

```
$ psql -c "SELECT pg_switch_wal();"
 pg_switch_wal 
---------------
 0/16AC7D0
(1 row)

$ ps -ef |grep postgres|grep archiver 
... postgres: archiver last was 000000010000000000000001

$ ls -l /var/lib/pgsql/11/archives/
total 16384
-rw-------. 1 postgres postgres 16777216 Nov 26 09:30 000000010000000000000001
```

Create a base copy for our secondary server:

```
$ pg_basebackup --pgdata=/var/lib/pgsql/11/replicated_data -P
24502/24502 kB (100%), 1/1 tablespace
```

Configure the recovery.conf file as following:

```
$ cat recovery.conf 
standby_mode = 'on'
primary_conninfo = 'port=5432'
restore_command = 'cp /var/lib/pgsql/11/archives/%f %p'
recovery_target_timeline = 'latest'
```

Change the default port and start:

```
$ echo 'port = 5433' >> /var/lib/pgsql/11/replicated_data/postgresql.conf
$ /usr/pgsql-11/bin/pg_ctl -D /var/lib/pgsql/11/replicated_data/ start
```

If the replication is correctly setup, you should see those processes:

```
postgres 10950     1  ... /usr/pgsql-11/bin/postmaster -D /var/lib/pgsql/11/data/
postgres 10958 10950  ... postgres: archiver   last was 000000010000000000000004
postgres 11595 10950  ... postgres: walsender postgres [local] streaming 0/5000140
...
postgres 11586     1  ... /usr/pgsql-11/bin/postgres -D /var/lib/pgsql/11/replicated_data
postgres 11588 11586  ... postgres: startup   recovering 000000010000000000000005
postgres 11594 11586  ... postgres: walreceiver   streaming 0/5000140
```

We now have a local 2-nodes cluster working with Streaming Replication and 
archives recovery as safety net.

To stop the cluster:

```
$ /usr/pgsql-11/bin/pg_ctl -D /var/lib/pgsql/11/replicated_data stop
# systemctl stop postgresql-11.service 
```

-----

## [](#recovery-conf-explanation)Recovery.conf explanation

What's important here are those parameters:

* standby_mode

> Specifies whether to start the PostgreSQL server as a standby. 
> If this parameter is on, the server will not stop recovery when the end of 
> archived WAL is reached, but will keep trying to continue recovery by fetching 
> new WAL segments using restore_command and/or by connecting to the primary 
> server as specified by the primary_conninfo setting.

* primary_conninfo

> Specifies a connection string to be used for the standby server to connect 
> with the primary.

* restore_command

> The local shell command to execute to retrieve an archived segment of the 
> WAL file series. This parameter is required for archive recovery, but optional 
> for streaming replication.

* recovery_target_timeline

> Specifies recovering into a particular timeline. The default is to recover 
> along the same timeline that was current when the base backup was taken. 
> Setting this to latest recovers to the latest timeline found in the archive,
> which is useful in a standby server.

For a complete information, you can refer to the 
[standby-settings](https://www.postgresql.org/docs/current/standby-settings.html), 
[archive-recovery-settings](https://www.postgresql.org/docs/current/archive-recovery-settings.html) 
and [recovery-target-settings](https://www.postgresql.org/docs/current/recovery-target-settings.html) 
official documentation.

-----

## [](#same-example-with-v12)Same example with v12

Get PostgreSQL sources and build the v12 version for this specific commit:

```
# mkdir /opt/build_postgresql
# cd /opt/build_postgresql/
# git clone git://git.postgresql.org/git/postgresql.git
# cd postgresql/
# git checkout 2dedf4d9a899b36d1a8ed29be5efbd1b31a8fe85
# ./configure
# make
# make install
$ /usr/local/pgsql/bin/initdb -D /var/lib/pgsql/12/data_build
```

Configure the archiver process:

```
$ mkdir /var/lib/pgsql/12/archives
$ echo "archive_mode = 'on'" >> /var/lib/pgsql/12/data_build/postgresql.conf
$ echo "archive_command = 'cp %p /var/lib/pgsql/12/archives/%f'" \
>> /var/lib/pgsql/12/data_build/postgresql.conf
$ echo "unix_socket_directories = '/var/run/postgresql, /tmp'" \
>> /var/lib/pgsql/12/data_build/postgresql.conf
$ /usr/local/pgsql/bin/pg_ctl -D /var/lib/pgsql/12/data_build start
```

Check:

```
$ psql -c "SELECT pg_switch_wal();"
 pg_switch_wal 
---------------
 0/20000B0
(1 row)

$ ps -ef |grep postgres|grep archiver 
... postgres: archiver   last was 000000010000000000000002
```

Create the base copy:

```
$ pg_basebackup --pgdata=/var/lib/pgsql/12/replicated_data -P
24534/24534 kB (100%), 1/1 tablespace
$ echo 'port = 5433' >> /var/lib/pgsql/12/replicated_data/postgresql.conf
```

Here comes the part modified by this patch. Most of the parameters for the 
recovery and standby mode has been moved directly to the main configuration file.

```
$ echo "primary_conninfo = 'port=5432'" \
>> /var/lib/pgsql/12/replicated_data/postgresql.conf
$ echo "restore_command = 'cp /var/lib/pgsql/12/archives/%f %p'" \
>> /var/lib/pgsql/12/replicated_data/postgresql.conf
$ echo "recovery_target_timeline = 'latest'" \
>> /var/lib/pgsql/12/replicated_data/postgresql.conf
```

If you simply want to start a recovery process (f.e. restore a backup), you 
need to create a file named **recovery.signal** in the data directory.

Here, we want to set up a standby server. We'll then need to create a file 
named **standby.signal**.

```
$ touch /var/lib/pgsql/12/replicated_data/standby.signal
$ /usr/local/pgsql/bin/pg_ctl -D /var/lib/pgsql/12/replicated_data start
```

If the replication is correctly setup, you should see those processes:

```
postgres  9033     1  ... /usr/local/pgsql/bin/postgres -D /var/lib/pgsql/12/data_build  
postgres  9039  9033  ... postgres: archiver   last was 000000010000000000000006
postgres 11608  9033  ... postgres: walsender postgres [local] streaming 0/7000060
...
postgres 11599     1  ... /usr/local/pgsql/bin/postgres -D /var/lib/pgsql/12/replicated_data
postgres 11600 11599  ... postgres: startup   recovering 000000010000000000000007 
postgres 11607 11599  ... postgres: walreceiver   streaming 0/7000060
```

To stop the cluster:

```
$ /usr/local/pgsql/bin/pg_ctl -D /var/lib/pgsql/12/replicated_data stop
$ /usr/local/pgsql/bin/pg_ctl -D /var/lib/pgsql/12/data_build stop
```

-----

## [](#pgbasebackup-behavior)pg_basebackup behavior

In version 11, the **-R, --write-recovery-conf** options write the recovery.conf 
file. This patch change this behavior by appending settings to postgresql.auto.conf.

This part is actually raising some concerns in the community. 
For example, after a complete restore, the recovery.conf file was moved to 
recovery.done. This will not be the case anymore.

-----

# [](#conclusion)Conclusion

This commit can have a big impact on your backup tools and procedures. 

Until the official v12 release, a lot of changes may still happen on this topic. 

The best advice is, like always, to have an attentive look to the release notes.