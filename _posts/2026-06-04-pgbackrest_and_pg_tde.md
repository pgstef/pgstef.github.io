---
layout: post
title: Does pgBackRest work with pg_tde?
date: 2026-06-04 09:10:00 +0200
---

Percona Transparent Data Encryption for PostgreSQL (`pg_tde`) is an open-source PostgreSQL extension that provides Transparent Data Encryption (TDE) to protect data at rest. `pg_tde` ensures that data stored on disk is encrypted and cannot be read without the proper encryption keys, even if someone gains access to the physical storage media.

A few months ago, Percona published a [blog post](https://percona.community/blog/2026/03/10/running-pgbackrest-with-pg_tde-a-practical-percona-walkthrough/) describing how pgBackRest can be used with encrypted data, although not all features are supported. In that example, they pass decrypted WAL files to the pgBackRest archiving command and state that asynchronous archiving is not supported because (1) it would copy encrypted WAL segments and (2) the `restore_command` would attempt to re-encrypt the archived WAL files.

Hallway-track discussions at conferences about this limitation gave me the idea to test it myself, as I suspected that pgBackRest could handle encrypted WAL segments transparently. Let's take a closer look.

<!--MORE-->

-----

# Install PostgreSQL and `pg_tde`

First of all, I'm going to install PostgreSQL and the `pg_tde` extension on a Debian 12 test server. As `pg_tde` currently requires a specific PostgreSQL patch to work, the extension is bundled as a component of [Percona Server for PostgreSQL](https://docs.percona.com/postgresql/18/installing.html).

```bash
sudo apt-get install -y wget gnupg2 curl lsb-release
sudo wget https://repo.percona.com/apt/percona-release_latest.generic_all.deb
sudo dpkg -i percona-release_latest.generic_all.deb
sudo apt-get update
sudo percona-release setup ppg-18
sudo apt install percona-postgresql-18
sudo apt-get install -y percona-pg-tde18
```

There have been many recent discussions about joining efforts to develop a TDE implementation directly in PostgreSQL core, especially during PGConf.dev in Vancouver last month. Until that work is completed, it may be worthwhile to extend PostgreSQL's capabilities so that extensions like this can run without requiring a patched PostgreSQL fork.

Next, we need to load the library and restart PostgreSQL on our test server:

```sql
ALTER SYSTEM SET shared_preload_libraries = 'pg_tde';
```

```bash
sudo systemctl restart postgresql@18-main.service
```

As I'm planning to use pgbench to simulate some activity, let's create a `bench` database and enable the `pg_tde` extension in there:

```sql
postgres=# CREATE DATABASE bench;
CREATE DATABASE

postgres=# \c bench
You are now connected to database "bench" as user "postgres".

bench=# CREATE EXTENSION pg_tde;
CREATE EXTENSION
```

One of the most important—and often most complex—aspects of using TDE is integrating a [key management](https://docs.percona.com/pg-tde/global-key-provider-configuration/overview.html) system.

---

# Using OpenBao as a key provider

For this test, I'm going to configure `pg_tde` to use OpenBao as a global key provider for managing encryption keys securely.
To do so, I'm going to run OpenBao in [Docker](https://hub.docker.com/r/openbao/openbao).

```bash
docker run \
  --name openbao \
  --detach \
  --volume /openbao/config:/openbao/config \
  --publish 8200:8200 \
  openbao/openbao
```

## OpenBao setup

* Create a policy file

```bash
cat > pg-tde-policy.hcl <<EOF
path "secret/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
EOF
```

This defines an OpenBao access policy granting full read/write access to everything under the `secret/` path (the KV mount).
This is where `pg_tde` will store and retrieve its encryption keys.

* Copy the policy into the container:

```bash
docker cp pg-tde-policy.hcl openbao:/tmp/pg-tde-policy.hcl
```

This command copies the file from the host into the running container so the `bao` CLI inside can read it.

* Write the policy to OpenBao:

```bash
docker exec -e VAULT_ADDR='http://127.0.0.1:8200' -e VAULT_TOKEN='XXX' openbao bao policy write pg-tde-policy /tmp/pg-tde-policy.hcl
```

This registers the policy under the name `pg-tde-policy`.

* Create a scoped token:

```bash
docker exec -e VAULT_ADDR='http://127.0.0.1:8200' -e VAULT_TOKEN='XXX' openbao bao token create \
  -policy=pg-tde-policy \
  -display-name=pg-tde \
  -no-default-policy \
  -period=0
```

This follows the principle of least privilege: PostgreSQL gets a token that can only access the key store, not the root token.

* Store the token for PostgreSQL to use on the test server:

```bash
echo -n 'YYY' > /etc/postgresql/18/main/token_file
chmod 600 /etc/postgresql/18/main/token_file
```

## PostgreSQL and OpenBao integration

* Let's register our OpenBao server as a global key provider:

```sql
SELECT pg_tde_add_global_key_provider_vault_v2(
    'my-openbao-provider',
    'http://192.168.121.1:8200',
    'secret',
    '/etc/postgresql/18/main/token_file',
    NULL
);
```

`secret` is the mount point where the keyring should store the keys.
`/etc/postgresql/18/main/token_file` is the path to the file that contains an access token with read and write access to the above mount point.

* Configure a default principal key for the server using a global key provider (this key is used by all databases that do not have their own encryption keys configured):

```sql
SELECT pg_tde_create_key_using_global_key_provider(
    'keytest1',
    'my-openbao-provider'
);

SELECT pg_tde_set_default_key_using_global_key_provider(
    'keytest1',
    'my-openbao-provider'
);
```

`pg_tde_create_key_using_global_key_provider()` creates a principal key at a global key provider.
`pg_tde_set_default_key_using_global_key_provider()` sets the default principal key and rotates the internal encryption key if one is already configured.

Note that if a default principal key is already configured for the server, WAL encryption uses it automatically.

* Enable WAL encryption and restart PostgreSQL:

```sql
ALTER SYSTEM SET pg_tde.wal_encrypt = on;
```

```bash
sudo systemctl restart postgresql@18-main.service
```

## Initialize database content and verify encryption

Let's initialize the pgbench tables (approx 1.5 GB):

```bash
pgbench -i -s 100 bench
```

Let's now encrypt those tables:

```sql
ALTER TABLE pgbench_accounts SET ACCESS METHOD tde_heap;
ALTER TABLE pgbench_branches SET ACCESS METHOD tde_heap;
ALTER TABLE pgbench_history SET ACCESS METHOD tde_heap;
ALTER TABLE pgbench_tellers SET ACCESS METHOD tde_heap;
```

It is now easy to check if encryption is turned on:

```sql
SHOW pg_tde.wal_encrypt;
SELECT pg_tde_is_encrypted('pgbench_accounts');
SELECT pg_tde_is_encrypted('pgbench_branches');
SELECT pg_tde_is_encrypted('pgbench_history');
SELECT pg_tde_is_encrypted('pgbench_tellers');
```

---

# pgBackRest setup and test

Now comes the fun part: installing pgBackRest. To do so, I'm going to use the PGDG APT repository.

```bash
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
sudo apt install -y pgbackrest
```

Check that you have the latest version installed:

```bash
$ pgbackrest version
pgBackRest 2.58.0
```

I'm going to store the repository in an S3-compatible bucket running in a Docker container alongside OpenBao. Here is the full configuration I used:

```ini
[global]
repo1-type=s3
repo1-s3-endpoint=192.168.122.1
repo1-s3-region=eu-central-1
repo1-s3-bucket=pgbackrest-tde
repo1-s3-key=XXX
repo1-s3-key-secret=YYY
repo1-s3-uri-style=path
repo1-storage-verify-tls=n
repo1-path=/repo1
repo1-retention-full=1
repo1-bundle=y
repo1-block=y

start-fast=y
delta=y
process-max=2
log-level-console=info
log-level-file=detail
compress-type=zst
archive-header-check=n
checksum-page=n

[tdedemo]
pg1-path=/var/lib/postgresql/18/main
pg-version-force=18
```

A few settings are worth highlighting:

`repo1-bundle=y` helps reduce the number of small files, which is generally more efficient, especially when using object storage such as S3.

`repo1-block=y` turns on block-level incremental backup.

`compress-type=zst`: we generally recommend using Zstandard compression, but Percona recommends `compress-type=none` here because compressing encrypted data blocks provides almost no storage benefit while still adding CPU overhead.

`pg_tde` encrypts data pages and WAL segments, so the content looks like random bytes to pgBackRest. As a result, WAL header checks and data-page checksum validation may either fail or be meaningless because pgBackRest cannot interpret the encrypted content. Therefore, we need to set `archive-header-check=n` and `checksum-page=n`.

Because `pg_tde` runs on Percona Server for PostgreSQL, we have to use `pg-version-force=18` to force the PostgreSQL version and skip the automatic detection.

Now that pgBackRest is configured, let's enable WAL archiving and restart PostgreSQL:

```sql
ALTER SYSTEM SET archive_mode = 'on';
ALTER SYSTEM SET archive_command = 'pgbackrest --stanza=tdedemo archive-push %p';
```

```bash
sudo systemctl restart postgresql@18-main.service
```

Initialize the repository content:

```bash
$ pgbackrest --stanza=tdedemo stanza-create
P00   INFO: stanza-create command begin 2.58.0: ...
P00   INFO: stanza-create for stanza 'tdedemo' on repo1
P00   INFO: stanza-create command end: completed successfully
```

Take a first backup:

```bash
$ pgbackrest --stanza=tdedemo --type=full backup
P00   INFO: backup command begin 2.58.0: ...
P00   INFO: execute backup start: backup begins after the requested immediate checkpoint completes
P00   INFO: backup start archive = 0000000100000000000000B6, lsn = 0/B6000028
P00   INFO: check archive for prior segment 0000000100000000000000B5
P00   INFO: execute backup stop and wait for all WAL segments to archive
P00   INFO: backup stop archive = 0000000100000000000000B6, lsn = 0/B6000158
P00   INFO: check archive for segment(s) 0000000100000000000000B6:0000000100000000000000B6
P00   INFO: new backup label = 20260603-130645F
P00   INFO: full backup size = 1.5GB, file total = 1278
P00   INFO: backup command end: completed successfully
```

```bash
$ pgbackrest --stanza=tdedemo info
stanza: tdedemo
    status: ok
    cipher: none

    db (current)
        wal archive min/max (18): 0000000100000000000000B6/0000000100000000000000B6

        full backup: 20260603-130645F
            timestamp start/stop: 2026-06-03 13:06:45+00 / 2026-06-03 13:06:58+00
            wal start/stop: 0000000100000000000000B6 / 0000000100000000000000B6
            database size: 1.5GB, database backup size: 1.5GB
            repo1: backup size: 1.5GB
```

In many pgBackRest deployments, compression is enabled to reduce backup size. However, when using `pg_tde`, database pages are already encrypted before pgBackRest processes them, as shown in the example above. Encryption randomizes the data blocks, making traditional compression algorithms ineffective. For this reason, we'll disable compression to avoid unnecessary CPU overhead.

We also want to test asynchronous archiving, so let's add that option too:

```ini
compress-type=none
archive-async=y
```

Now, let's run pgbench, take a new backup, and simulate a recovery scenario.

```bash
$ pgbackrest --stanza=tdedemo --type=full backup
P00   INFO: backup command begin 2.58.0: ...
P00   INFO: execute backup start: backup begins after the requested immediate checkpoint completes
P00   INFO: backup start archive = 0000000100000000000000CC, lsn = 0/CC000118
P00   INFO: check archive for prior segment 0000000100000000000000CB
P00   INFO: execute backup stop and wait for all WAL segments to archive
P00   INFO: backup stop archive = 0000000100000000000000D4, lsn = 0/D42CE038
P00   INFO: check archive for segment(s) 0000000100000000000000CC:0000000100000000000000D4
P00   INFO: new backup label = 20260603-132748F
P00   INFO: full backup size = 1.5GB, file total = 1284
P00   INFO: backup command end: completed successfully
```

```bash
$ pgbackrest --stanza=tdedemo info
stanza: tdedemo
    status: ok
    cipher: none

    db (current)
        wal archive min/max (18): 0000000100000000000000CC/0000000100000000000000E0

        full backup: 20260603-132748F
            timestamp start/stop: 2026-06-03 13:27:48+00 / 2026-06-03 13:28:01+00
            wal start/stop: 0000000100000000000000CC / 0000000100000000000000D4
            database size: 1.5GB, database backup size: 1.5GB
            repo1: backup size: 1.5GB
```

Let's now create a named restore point and delete data from the `pgbench_tellers` table:

```sql
SELECT pg_create_restore_point('RP1');
BEGIN;
SELECT pg_current_wal_lsn(), current_timestamp;
DELETE FROM pgbench_tellers;
COMMIT;
```

The `DELETE` operation returned `DELETE 1000`, confirming that 1,000 rows were removed.

Let's stop PostgreSQL and restore the latest backup using the named recovery target `RP1`:

```bash
sudo systemctl stop postgresql@18-main.service
```

```bash
$ pgbackrest --stanza=tdedemo restore --type=name --target='RP1'
P00   INFO: restore command begin 2.58.0: ...
P00   INFO: repo1: restore backup set 20260603-132748F, recovery will start at 2026-06-03 13:27:48
P00   INFO: remove invalid files/links/paths from '/var/lib/postgresql/18/main'
P00   INFO: write updated /var/lib/postgresql/18/main/postgresql.auto.conf
P00   INFO: restore global/pg_control (performed last to ensure aborted restores cannot be started)
P00   INFO: restore size = 1.5GB, file total = 1284
P00   INFO: restore command end: completed successfully
```

Check the generated recovery settings and start PostgreSQL:

```bash
$ cat /var/lib/postgresql/18/main/postgresql.auto.conf
...
# Recovery settings generated by pgBackRest restore on 2026-06-03 14:25:19
restore_command = 'pgbackrest --stanza=tdedemo archive-get %f "%p"'
recovery_target_name = 'RP1'
```

```bash
sudo systemctl start postgresql@18-main.service
```

Check the logs to make sure recovery reached the target:

```bash
$ tail /var/log/postgresql/postgresql-18-main.log
P00   INFO: archive-get command begin 2.58.0: ...
P00   INFO: found 000000010000000100000007 in the archive asynchronously
P00   INFO: archive-get command end: completed successfully
LOG:  restored log file "000000010000000100000007" from archive
LOG:  recovery stopping at restore point "RP1", time ...
LOG:  pausing at the end of recovery
HINT:  Execute pg_wal_replay_resume() to promote.
```

Complete the recovery and verify that the 1,000 deleted rows have been restored:

```sql
SELECT pg_wal_replay_resume();
```

```bash
$ tail /var/log/postgresql/postgresql-18-main.log
LOG:  archive recovery complete
LOG:  database system is ready to accept connections
P00   INFO: archive-push command begin 2.58.0: ...
P00   INFO: pushed WAL file '00000002.history' to the archive asynchronously
P00   INFO: archive-push command end: completed successfully
```

```sql
bench=# SELECT count(*) FROM pgbench_tellers;
 count
-------
  1000
(1 row)
```

This confirms that asynchronous archiving works as expected in this setup, and that compression provides little benefit.

Let's take some incremental backups while pgbench is running and look at the `info` command output:

```bash
$ pgbackrest info
stanza: tdedemo
    status: ok
    cipher: none

    db (current)
        wal archive min/max (18): 0000000100000000000000CC/000000020000000100000025

        full backup: 20260603-132748F
            timestamp start/stop: 2026-06-03 13:27:48+00 / 2026-06-03 13:28:01+00
            wal start/stop: 0000000100000000000000CC / 0000000100000000000000D4
            database size: 1.5GB, database backup size: 1.5GB
            repo1: backup size: 1.5GB

        incr backup: 20260603-132748F_20260603-144243I
            timestamp start/stop: 2026-06-03 14:42:43+00 / 2026-06-03 14:42:57+00
            wal start/stop: 000000020000000100000008 / 000000020000000100000008
            database size: 1.5GB, database backup size: 1.5GB
            repo1: backup size: 1.5GB
            backup reference total: 1 full

        incr backup: 20260603-132748F_20260603-144322I
            timestamp start/stop: 2026-06-03 14:43:22+00 / 2026-06-03 14:43:27+00
            wal start/stop: 00000002000000010000000F / 000000020000000100000013
            database size: 1.5GB, database backup size: 1.5GB
            repo1: backup size: 560.1MB
            backup reference total: 1 full, 1 incr

        incr backup: 20260603-132748F_20260603-144348I
            timestamp start/stop: 2026-06-03 14:43:48+00 / 2026-06-03 14:43:59+00
            wal start/stop: 00000002000000010000001F / 000000020000000100000025
            database size: 1.5GB, database backup size: 1.5GB
            repo1: backup size: 1.1GB
            backup reference total: 1 full, 2 incr
```

Looking at the reported `backup size` values, pgbench should not have generated enough changes to justify such large incremental backups. Encryption has a significant impact here, greatly reducing the storage savings that block-level incremental backups normally provide.

---

# Conclusion

pgBackRest works well with `pg_tde` and handles encrypted data largely transparently. Both data files and WAL segments remain encrypted when pgBackRest stores them.

The features tested here worked after setting `archive-header-check=n` to disable WAL header checks and `checksum-page=n` to disable data-page checksum validation.

Encrypted data does not compress well, so we can save CPU by disabling compression with `compress-type=none`. Encrypted data also reduces the effectiveness of block-level incremental backups, so it is better to leave that disabled as well with `repo1-block=n`.

My final test configuration looks like this:

```ini
[global]
repo1-type=s3
repo1-s3-endpoint=192.168.122.1
repo1-s3-region=eu-central-1
repo1-s3-bucket=pgbackrest-tde
repo1-s3-key=XXX
repo1-s3-key-secret=YYY
repo1-s3-uri-style=path
repo1-storage-verify-tls=n
repo1-path=/repo1
repo1-retention-full=1
repo1-bundle=y
repo1-block=n

start-fast=y
delta=y
process-max=2
log-level-console=info
log-level-file=detail
compress-type=none
archive-header-check=n
checksum-page=n
archive-async=y

[tdedemo]
pg1-path=/var/lib/postgresql/18/main
pg-version-force=18
```

Naturally, you'll need to adjust the repository configuration to match your own environment.

You can also add an additional encryption layer using `repo1-cipher-type` and `repo1-cipher-pass`. However, keep in mind that `pg_tde` already encrypts both data files and WAL segments before pgBackRest stores them. Repository-level encryption may still be useful if your backup policy requires a separate encryption layer, but it is not needed to prevent pgBackRest from storing plaintext database contents.

**TL;DR**: Contrary to the limitation described in Percona's blog post, asynchronous archiving worked correctly with `pg_tde` in this test setup. pgBackRest was able to archive and restore encrypted WAL segments without requiring the custom decryption and re-encryption wrappers described there.
