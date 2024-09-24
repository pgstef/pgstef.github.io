---
layout: post
title: pgBackRest dedicated backup host
date: 2024-09-25 10:00:00 +0200
draft: true
---

As I mentioned in my last blog post, as your cluster grows with multiple standby servers and potentially automated failover (using tools like Patroni), it becomes more practical to set up a dedicated repository host, also known as a dedicated backup server. This backup server can then trigger backups and automatically select the appropriate node in case of failover, eliminating the need for manual intervention.

In this post, I'll show you how easy it is to add a repository host to an existing cluster. I'll also give you a sneak peek at a new feature expected to be included in the next pgBackRest release 😉

<!--MORE-->

-----

# Example setup for repository host

## Initial situation

In this example, we pick up from where we left off [last time](https://pgstef.github.io/2024/09/23/pgbackrest_backups_from_the_standby_server.html): a primary server (`pg1`) with a standby (`pg2`), both already configured to use pgBackRest (with an NFS mount) for backups taken from the standby. Now, we will add a new node, `repo1`, to take over pgBackRest backups.

The pgBackRest [user guide](https://pgbackrest.org/user-guide-rhel.html#repo-host) provides a comprehensive overview of how to set up a repository host. Since pgBackRest needs to interact with local processes on each node, we must enable communication between the hosts, either through passwordless SSH or [TLS with client certificates](https://pgbackrest.org/user-guide-rhel.html#repo-host/config). While SSH is generally easier to set up, TLS offers better performance. If you're interested in an example of the TLS setup, I wrote [this blog post](https://pgstef.github.io/2022/02/21/pgbackrest_tls_server.html) when the feature was first introduced.

## Installation

Let's return to our repository host setup. The first step, of course, is to install pgBackRest:

```bash
$ sudo dnf install pgbackrest -y
```

Any user can own the repository, but it's best to avoid using the `postgres` user (if it exists) to prevent confusion. Instead, let’s create a dedicated system user for this purpose:

```bash
$ sudo groupadd pgbackrest
$ sudo adduser -g pgbackrest -n pgbackrest
$ sudo chown -R pgbackrest: /var/log/pgbackrest/
```

The SSH setup is up to you, but usually it is as simple as creating SSH keys and authorize them on the other nodes. Example:

```bash
# From repo1
[pgbackrest@repo1 ~]$ ssh-keygen -f /home/pgbackrest/.ssh/id_rsa -t rsa -b 4096 -N ""
[pgbackrest@repo1 ~]$ ssh-copy-id postgres@pg1
[pgbackrest@repo1 ~]$ ssh postgres@pg1 hostname
[pgbackrest@repo1 ~]$ ssh-copy-id postgres@pg2
[pgbackrest@repo1 ~]$ ssh postgres@pg2 hostname

# From pg1
[postgres@pg1 ~]$ ssh-copy-id pgbackrest@repo1
[postgres@pg1 ~]$ ssh pgbackrest@repo1 hostname

# From pg2
[postgres@pg2 ~]$ ssh-copy-id pgbackrest@repo1
[postgres@pg2 ~]$ ssh pgbackrest@repo1 hostname
```

The repository can be stored on any supported [repository type](https://pgbackrest.org/configuration.html#section-repository/option-repo-type). In this example, we'll create a local directory on the backup server (also known as the repository host, `repo1`):

```bash
$ sudo -u pgbackrest mkdir /shared/repo1
```

_Remark:_ You can reuse the previous repository (where you already have archives and backups) if you don't want to start from scratch. Just ensure it's accessible from the `repo1` host and that the `pgbackrest` user has full access to it.

## Configuration

Now, let's move on to configuring pgBackRest on `repo1`:

```ini
[global]
repo1-path=/shared/repo1
repo1-retention-full=4
repo1-bundle=y
repo1-block=y
start-fast=y
log-level-console=info
log-level-file=detail
delta=y
process-max=2
compress-type=zst
backup-standby=y

[mycluster]
pg1-path=/var/lib/pgsql/16/data
pg1-host=pg1
pg1-host-user=postgres
pg2-path=/var/lib/pgsql/16/data
pg2-host=pg2
pg2-host-user=postgres
```

The global section here includes options primarily used to control the behavior of the backup command and specify where to store the backups and archives. The `[mycluster]` stanza section, on the other hand, defines the locations of all the PostgreSQL nodes within the cluster.

Speaking about the PostgreSQL nodes, let's now update the pgBackRest configuration on `pg1` and `pg2`:

```ini
[global]
repo1-host=repo1
repo1-host-user=pgbackrest
log-level-console=info
log-level-file=detail
compress-type=zst

[mycluster]
pg1-path=/var/lib/pgsql/16/data
```

Since the backup command will be executed on `repo1`, the key point is to define the location of the locally running database in the `[mycluster]` stanza section and configure how to reach the repository host in the global section. Additionally, options for controlling archiving (`archive-push`/`archive-get`) and restore commands should also be specified here.

Next, initialize the backup repository by running the `stanza-create` command on the repository host (`repo1`). After that, run the `check` command to ensure that the archiving process is working correctly and that WAL archives are successfully being pushed to the new repository:

```bash
$ sudo -iu pgbackrest pgbackrest --stanza=mycluster stanza-create
INFO: stanza-create command begin 2.53.1: ...
INFO: stanza-create for stanza 'mycluster' on repo1
INFO: stanza-create command end: completed successfully

$ sudo -iu pgbackrest pgbackrest --stanza=mycluster check
INFO: check command begin 2.53.1: ...
INFO: check repo1 (standby)
INFO: switch wal not performed because this is a standby
INFO: check repo1 configuration (primary)
INFO: check repo1 archive for WAL (primary)
INFO: WAL segment 000000010000000000000057 successfully archived to '...' on repo1
INFO: check command end: completed successfully
```

You can also run the `check` command on the primary (it won't have any effect on the standby server):

```bash
$ sudo -iu postgres pgbackrest --stanza=mycluster check
```

_Remark:_ If you chose to reuse the existing repository, there's no need to re-run the `stanza-create` command, as the repository is already initialized. However, you can still run it, and the `stanza-create` command will simply verify that everything is fine:

```bash
$ sudo -iu pgbackrest pgbackrest --stanza=mycluster stanza-create
INFO: stanza-create command begin 2.53.1: ...
INFO: stanza-create for stanza 'mycluster' on repo1
INFO: stanza 'mycluster' already exists on repo1 and is valid
INFO: stanza-create command end: completed successfully
```

## Backups

Let's now take a backup from the repository host (`repo1`):

```bash
$ sudo -iu pgbackrest pgbackrest backup --stanza=mycluster --type=full
INFO: backup command begin 2.53.1: ...
INFO: execute non-exclusive backup start: backup begins after the requested immediate checkpoint completes
INFO: backup start archive = 000000010000000000000059, lsn = 0/59000028
INFO: wait for replay on the standby to reach 0/59000028
INFO: replay on the standby reached 0/59000028
INFO: check archive for prior segment 000000010000000000000058
INFO: execute non-exclusive backup stop and wait for all WAL segments to archive
INFO: backup stop archive = 000000010000000000000059, lsn = 0/59000138
INFO: check archive for segment(s) 000000010000000000000059:000000010000000000000059
INFO: new backup label = 20240923-142704F
INFO: full backup size = 1.5GB, file total = 1281
INFO: backup command end: completed successfully
```

What happens if the standby server is down?

```bash
$ sudo -iu pgbackrest pgbackrest backup --stanza=mycluster --type=full
INFO: backup command begin 2.53.1: ...
WARN: unable to check pg2: [DbConnectError] raised from remote-0 ssh protocol on 'pg2':
    unable to connect to 'dbname='postgres' port=5432':
        connection to server on socket "/run/postgresql/.s.PGSQL.5432" failed: No such file or directory
    Is the server running locally and accepting connections on that socket?
ERROR: [056]: unable to find standby cluster - cannot proceed
INFO: backup command end: aborted with exception [056]
```

If the standby server is down and the configuration is set to take backups from the standby (`backup-standby=y`), the backup command will fail. In such cases, you can trigger a backup from the primary by using the `--no-backup-standby` option in the command line:

```bash
$ sudo -iu pgbackrest pgbackrest backup --stanza=mycluster --type=full --no-backup-standby
INFO: backup command begin 2.53.1: ...
WARN: unable to check pg2: [DbConnectError] raised from remote-0 ssh protocol on 'pg2':
    unable to connect to 'dbname='postgres' port=5432':
        connection to server on socket "/run/postgresql/.s.PGSQL.5432" failed: No such file or directory
    Is the server running locally and accepting connections on that socket?
INFO: execute non-exclusive backup start: backup begins after the requested immediate checkpoint completes
INFO: backup start archive = 00000001000000000000005C, lsn = 0/5C000028
INFO: check archive for prior segment 00000001000000000000005B
INFO: execute non-exclusive backup stop and wait for all WAL segments to archive
INFO: backup stop archive = 00000001000000000000005C, lsn = 0/5C000100
INFO: check archive for segment(s) 00000001000000000000005C:00000001000000000000005C
INFO: new backup label = 20240923-143128F
INFO: full backup size = 1.5GB, file total = 1281
INFO: backup command end: completed successfully
```

## New option preview

To avoid adjusting the configuration or manually triggering backups, an upcoming pgBackRest release will introduce a new option: [`backup-standby=prefer`](https://github.com/pgbackrest/pgbackrest/commit/a42629f87ab70ffaee8d1241eb50c5ea7154a87d). This option will allow backups to be taken from the standby if available, or automatically fall back to the primary if the standby is down.

To test this new feature, I've installed the development version of pgBackRest on all nodes:

```bash
$ pgbackrest version
pgBackRest 2.54dev
```

What would happen if I only upgraded the pgBackRest version on the repository host (`repo1`) but not on the PostgreSQL nodes (`pg1` and `pg2`)?

```bash
$ sudo -u pgbackrest pgbackrest backup --stanza=mycluster --type=full --no-backup-standby
INFO: backup command begin 2.54dev: ...
WARN: unable to check pg1: [ProtocolError] expected value '2.54dev' for greeting key 'version' but got '2.53.1'
      HINT: is the same version of pgBackRest installed on the local and remote host?
WARN: unable to check pg2: [ProtocolError] expected value '2.54dev' for greeting key 'version' but got '2.53.1'
      HINT: is the same version of pgBackRest installed on the local and remote host?
ERROR: [056]: unable to find primary cluster - cannot proceed
      HINT: are all available clusters in recovery?
INFO: backup command end: aborted with exception [056]
```

That's probably the main drawback about using repository hosts: the pgBackRest version installed on the repository host must exactly match the version installed on the PostgreSQL hosts.

Finally, let's test the new option:

```bash
$ sudo -u pgbackrest pgbackrest backup --stanza=mycluster --type=full --backup-standby=prefer
INFO: backup command begin 2.54dev: ...
WARN: unable to check pg2: [DbConnectError] raised from remote-0 ssh protocol on 'pg2':
    unable to connect to 'dbname='postgres' port=5432':
        connection to server on socket "/run/postgresql/.s.PGSQL.5432" failed: No such file or directory
    Is the server running locally and accepting connections on that socket?
WARN: unable to find a standby to perform the backup, using primary instead
INFO: execute non-exclusive backup start: backup begins after the requested immediate checkpoint completes
INFO: backup start archive = 00000001000000000000005E, lsn = 0/5E000028
INFO: check archive for prior segment 00000001000000000000005D
INFO: execute non-exclusive backup stop and wait for all WAL segments to archive
INFO: backup stop archive = 00000001000000000000005E, lsn = 0/5E000138
INFO: check archive for segment(s) 00000001000000000000005E:00000001000000000000005E
INFO: new backup label = 20240923-144946F
INFO: full backup size = 1.5GB, file total = 1281
INFO: backup command end: completed successfully
```

---

# Conclusion

Setting up a dedicated pgBackRest repository host makes managing backups easier and reduces the load on your PostgreSQL nodes, especially as your cluster grows and you add more standby servers or automated failover tools.

Whether you're reusing an existing repository or starting fresh, the setup process is really straightforward. Plus, with features like `backup-standby=prefer` coming in the next release, backups will be more flexible, allowing for automatic fallback to the primary when the standby is down. PgBackRest keeps getting better with each new version, so keep an eye on the release radar (and release notes) to stay updated on the latest features and improvements!
