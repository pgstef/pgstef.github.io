---
layout: post
title: pgBackRest SFTP support
---

SFTP support has been added in the [2.46](https://github.com/pgbackrest/pgbackrest/releases/tag/release%2F2.46) release on 22 May 2022.

In this demo setup, the SFTP host will be called **sftp-srv** and the 2 PostgreSQL nodes in _Streaming Replication_: **pg1-srv**, **pg2-srv**. All the nodes will be running on Rocky Linux 8.

If you're familiar with Vagrant, here's a simple `Vagrantfile` to initiate 3 virtual machines using those names:

```ruby
# Vagrantfile
Vagrant.configure(2) do |config|
    config.vm.box = 'rockylinux/8'
    config.vm.provider 'libvirt' do |lv|
        lv.cpus = 1
        lv.memory = 1024
    end
    config.vm.synced_folder ".", "/vagrant", disabled: true

    nodes  = 'sftp-srv', 'pg1-srv', 'pg2-srv'
    nodes.each do |node|
        config.vm.define node do |conf|
            conf.vm.hostname = node
        end
    end
end
```

When using Vagrant boxes, it might be needed to enable the SSH password authentication to proceed with SSH key exchange:

```bash
$ sudo -i
root# sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config    
root# systemctl restart sshd.service
root# passwd
```

<!--MORE-->

-----

# Installation

On all the database nodes, first configure the PGDG repositories and install PostgreSQL:

```bash
$ sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
$ sudo dnf -qy module disable postgresql
$ sudo dnf install -y postgresql15-server postgresql15-contrib
```

Create a basic PostgreSQL instance on **pg1-srv**:

```bash
$ sudo PGSETUP_INITDB_OPTIONS="--data-checksums" /usr/pgsql-15/bin/postgresql-15-setup initdb
$ sudo systemctl enable postgresql-15
$ sudo systemctl start postgresql-15
```

Install pgBackRest and check its version on both PostgreSQL nodes:

```bash
$ sudo dnf install -y epel-release
$ sudo dnf install -y pgbackrest
$ pgbackrest version
pgBackRest 2.46
```

## Create a dedicated user and location on the SFTP server

The `pgbackrest` user is created to own the backups and archives repository on **sftp-srv**:

```bash
$ sudo groupadd pgbackrest
$ sudo adduser -g pgbackrest -n pgbackrest
$ sudo mkdir -m 750 -p /backup_space
$ sudo chown pgbackrest: /backup_space
$ sudo -u pgbackrest mkdir -m 750 -p /home/pgbackrest/.ssh
```

## Setup SSH communication between PostgreSQL nodes and SFTP server

The idea is to generate a dedicated SSH key pair for the SFTP connection and use a secured passphrase with it.

First of all, generate the SSH key pair on each PostgreSQL node:

```bash
$ sudo -u postgres ssh-keygen -f /var/lib/pgsql/.ssh/id_rsa_sftp -t rsa -b 4096 -N "BeSureToGenerateAndUseASecurePassphrase"
```

Then, copy the generated public keys to **sftp-srv** to authorize it:

```bash
$ ssh root@pg1-srv cat /var/lib/pgsql/.ssh/id_rsa_sftp.pub| \
    sudo -u pgbackrest tee -a /home/pgbackrest/.ssh/authorized_keys

$ ssh root@pg2-srv cat /var/lib/pgsql/.ssh/id_rsa_sftp.pub | \
    sudo -u pgbackrest tee -a /home/pgbackrest/.ssh/authorized_keys
```

The connection from the PostgreSQL nodes should now prompt for the passphrase:

```bash
$ sudo -u postgres ssh pgbackrest@sftp-srv -i /var/lib/pgsql/.ssh/id_rsa_sftp
```

## Setup password-less SSH between PostgreSQL nodes

To be able to take backups from the standby server, pgBackRest requires a direct communication between the PostgreSQL hosts.
One possibility to achieve that is to setup a password-less SSH connection.

Create the ssh key pair and authorize the public keys generated:

```bash
$ sudo -u postgres ssh-keygen -N "" -t rsa -b 4096 -f /var/lib/pgsql/.ssh/id_rsa
$ sudo -u postgres restorecon -R /var/lib/pgsql/.ssh

# On pg1-srv
$ ssh root@pg2-srv cat /var/lib/pgsql/.ssh/id_rsa.pub | \
    sudo -u postgres tee -a /var/lib/pgsql/.ssh/authorized_keys

# On pg2-srv
$ ssh root@pg1-srv cat /var/lib/pgsql/.ssh/id_rsa.pub | \
    sudo -u postgres tee -a /var/lib/pgsql/.ssh/authorized_keys
```

To test the connection:

```bash
# From pg2-srv
$ sudo -u postgres ssh postgres@pg1-srv

# From pg1-srv
$ sudo -u postgres ssh postgres@pg2-srv
```

**_Remark:_** You could setup and use a TLS server connection rather than SSH.

-----

# Configuration

We'll now prepare the configuration for our stanza called `demo`. 

Update the **pg1-srv** pgBackRest configuration file:

```ini
# /etc/pgbackrest.conf
[global]
repo1-bundle=y
repo1-type=sftp
repo1-path=/backup_space
repo1-sftp-host=sftp-srv
repo1-sftp-host-key-hash-type=sha1
repo1-sftp-host-user=pgbackrest
repo1-sftp-private-key-file=/var/lib/pgsql/.ssh/id_rsa_sftp
repo1-sftp-private-key-passphrase=BeSureToGenerateAndUseASecurePassphrase

repo1-retention-full=2
start-fast=y
log-level-console=info
log-level-file=debug
delta=y
process-max=2

[demo]
pg1-path=/var/lib/pgsql/15/data
```

Configure archiving in the `postgresql.conf` file:

```
listen_addresses = '*'
archive_mode = on
archive_command = 'pgbackrest --stanza=demo archive-push %p'
```

The PostgreSQL instance must be restarted after making these changes.

```bash
$ sudo systemctl restart postgresql-15.service
```

When using SFTP, pgBackRest commands are run like with other repository types (NFS, S3,...).
Let's then now create the stanza and check the configuration on the **pg1-srv**:

```bash
$ sudo -iu postgres pgbackrest --stanza=demo stanza-create
...
P00   INFO: stanza-create command end: completed successfully

$ sudo -iu postgres pgbackrest --stanza=demo check
...
P00   INFO: check command end: completed successfully
```

We can now take our first backup from **pg1-srv**:

```bash
$ sudo -iu postgres pgbackrest --stanza=demo --type=full backup
P00   INFO: backup command begin 2.46: ...
P00   INFO: execute non-exclusive backup start: backup begins after the requested immediate checkpoint completes
P00   INFO: backup start archive = 000000010000000000000003, lsn = 0/3000028
P00   INFO: check archive for prior segment 000000010000000000000002
P00   INFO: execute non-exclusive backup stop and wait for all WAL segments to archive
P00   INFO: backup stop archive = 000000010000000000000003, lsn = 0/3000138
P00   INFO: check archive for segment(s) 000000010000000000000003:000000010000000000000003
P00   INFO: new backup label = 20230523-141139F
P00   INFO: full backup size = 22MB, file total = 966
P00   INFO: backup command end: completed successfully
```

## Prepare the servers for Streaming Replication

On **pg1-srv**, create a specific user for the replication:

```bash
$ sudo -iu postgres psql
postgres=# CREATE ROLE replic_user WITH LOGIN REPLICATION PASSWORD 'mypwd';
```

Configure `pg_hba.conf`:

```
host replication replic_user pg2-srv scram-sha-256
```

Reload configuration:

```bash
$ sudo systemctl reload postgresql-15.service
```

Configure `~postgres/.pgpass` on **pg2-srv**:

```bash
$ echo "*:*:replication:replic_user:mypwd" >> ~postgres/.pgpass
$ chown postgres: ~postgres/.pgpass
$ chmod 0600 ~postgres/.pgpass
```

## Setup the standby server

Configure `/etc/pgbackrest.conf` on **pg2-srv**:

```ini
[global]
repo1-bundle=y
repo1-type=sftp
repo1-path=/backup_space
repo1-sftp-host=sftp-srv
repo1-sftp-host-key-hash-type=sha1
repo1-sftp-host-user=pgbackrest
repo1-sftp-private-key-file=/var/lib/pgsql/.ssh/id_rsa_sftp
repo1-sftp-private-key-passphrase=BeSureToGenerateAndUseASecurePassphrase

repo1-retention-full=2
start-fast=y
log-level-console=info
log-level-file=debug
delta=y
process-max=2

[demo]
pg1-path=/var/lib/pgsql/15/data
pg2-host=pg1-srv
pg2-path=/var/lib/pgsql/15/data
backup-standby=y
recovery-option=primary_conninfo=host=pg1-srv user=replic_user
```

Thanks to the `backup-standby=y` and `pg2-*` settings (pointing to the primary server), we will also be able to take backups from **pg2-srv** later.

To make sure pgBackRest is correctly set up here, use the `info` command to list the content of the repository:

```bash
$ sudo -iu postgres pgbackrest --stanza=demo info
stanza: demo
    status: ok
    cipher: none

    db (current)
        wal archive min/max (15): 000000010000000000000001/000000010000000000000003

        full backup: 20230523-141139F
            timestamp start/stop: 2023-05-23 14:11:39 / 2023-05-23 14:11:51
            wal start/stop: 000000010000000000000003 / 000000010000000000000003
            database size: 22MB, database backup size: 22MB
            repo1: backup set size: 2.9MB, backup size: 2.9MB
```

Restore the backup taken from the **pg1-srv** server on **pg2-srv**:

```bash
$ sudo -iu postgres pgbackrest --stanza=demo --type=standby restore --no-delta
P00   INFO: restore command begin 2.46: ...
P00   INFO: repo1: restore backup set 20230523-141139F, recovery will start at 2023-05-23 14:11:39
P00   INFO: write updated /var/lib/pgsql/15/data/postgresql.auto.conf
P00   INFO: restore global/pg_control (performed last to ensure aborted restores cannot be started)
P00   INFO: restore size = 22MB, file total = 966
P00   INFO: restore command end: completed successfully
```

The restore will add extra information to the `postgresql.auto.conf` file:

```ini
# Recovery settings generated by pgBackRest restore...
primary_conninfo = 'host=pg1-srv user=replic_user'
restore_command = 'pgbackrest --stanza=demo archive-get %f "%p"'
```

The `--type=standby` option creates the `standby.signal` needed for PostgreSQL to start in standby mode.
All we have to do now is to start the PostgreSQL instances:

```bash
$ sudo systemctl enable postgresql-15
$ sudo systemctl start postgresql-15
```

If the replication setup is correct, you should see those processes on the **pg1-srv** server:

```
$ ps -ef |grep postgres
postgres 34939     1  ... /usr/pgsql-15/bin/postmaster -D /var/lib/pgsql/15/data/
postgres 35422 34939  ... postgres: walsender replic_user ... streaming 0/4001420
```

We now have a PostgreSQL cluster working with _Streaming Replication_ and archives recovery as safety net.

Let's finally take a backup from this standby server:

```bash
$ sudo -iu postgres pgbackrest --stanza=demo --type=full backup
P00   INFO: backup command begin 2.46: ...
P00   INFO: execute non-exclusive backup start: backup begins after the requested immediate checkpoint completes
P00   INFO: backup start archive = 000000010000000000000005, lsn = 0/5000028
P00   INFO: wait for replay on the standby to reach 0/5000028
P00   INFO: replay on the standby reached 0/5000028
P00   INFO: check archive for prior segment 000000010000000000000004
P00   INFO: execute non-exclusive backup stop and wait for all WAL segments to archive
P00   INFO: backup stop archive = 000000010000000000000005, lsn = 0/5000138
P00   INFO: check archive for segment(s) 000000010000000000000005:000000010000000000000005
P00   INFO: new backup label = 20230523-143125F
P00   INFO: full backup size = 22MB, file total = 966
P00   INFO: backup command end: completed successfully
```

-----

# Conclusion

As mentioned in the official documentation, SFTP file transfer is relatively slow.
Increasing `process-max` to parallelize file transfer and using file bundling will help but it would probably be better to ...

<!--
# Steps to build pgBackRest from sources if needed
sudo dnf -y install git wget
sudo dnf -y --enablerepo=powertools install make gcc postgresql15-devel openssl-devel libxml2-devel lz4-devel libzstd-devel bzip2-devel libyaml-devel libssh2-devel
sudo -i
mkdir -p /build
wget -q -O - https://github.com/pgbackrest/pgbackrest/archive/release/2.46.tar.gz | tar zx -C /build
cd /build/pgbackrest-release-2.46/src
export PATH=$PATH:/usr/pgsql-15/bin
./configure && make
-->
