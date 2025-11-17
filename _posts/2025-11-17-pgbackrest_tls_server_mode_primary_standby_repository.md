---
layout: post
title: pgBackRest TLS server mode for a primary-standby setup with a repository host
---

The TLS server provides an alternative to using SSH for protocol connections to remote hosts.

In this demo setup, the repository host is named **backup-srv**, and the two PostgreSQL nodes participating in _Streaming Replication_ are **pg1-srv** and **pg2-srv**.
All nodes run on AlmaLinux 10.

If you're familiar with Vagrant, here is a simple `Vagrantfile` that provisions three virtual machines using these names:

```ruby
# Vagrantfile
Vagrant.configure(2) do |config|
    config.vm.box = 'almalinux/10'
    config.vm.provider 'libvirt' do |lv|
        lv.cpus = 1
        lv.memory = 1024
    end

    # share the default vagrant folder
    config.vm.synced_folder ".", "/vagrant", type: "nfs", nfs_udp: false

    nodes  = 'backup-srv', 'pg1-srv', 'pg2-srv'
    nodes.each do |node|
        config.vm.define node do |conf|
            conf.vm.hostname = node
        end
    end
end
```

<!--MORE-->

-----

# Installation

On all servers, begin by configuring the PGDG repositories:

```bash
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-10-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

Then install PostgreSQL on **pg1-srv** and **pg2-srv**:

```bash
sudo dnf install -y postgresql18-server
```

Create a basic PostgreSQL instance on **pg1-srv**:

```bash
sudo /usr/pgsql-18/bin/postgresql-18-setup initdb
sudo systemctl enable postgresql-18
sudo systemctl start postgresql-18
```

Install pgBackRest on every host and confirm the version:

```bash
sudo dnf install -y epel-release
sudo dnf config-manager --enable crb
sudo dnf install -y pgbackrest
```
```bash
$ pgbackrest version
pgBackRest 2.57.0
```

## Create a dedicated user on the repository host

The `pgbackrest` user will own the backups and WAL archive repository.
Although any user can own the repository, it's best not to use `postgres` to avoid confusion.

Create the user and repository directory on **backup-srv**:

```bash
sudo mkdir -p /backup_space
sudo mkdir -p /var/log/pgbackrest

sudo groupadd pgbackrest
sudo useradd -d /backup_space -M -g pgbackrest pgbackrest

sudo chown -R pgbackrest: /backup_space
sudo chown -R pgbackrest: /var/log/pgbackrest
```

For the purpose of this demo, we've created `/backup_space` to store our backups and archives locally.
The repository can be located on the various supported storage types described in the [official configuration documentation](https://pgbackrest.org/configuration.html#section-repository).

-----

# Certificate generation

The TLS server requires certificates.

A good practical example of generating certificates for PostgreSQL authentication can be found
[here](https://blog.crunchydata.com/blog/ssl-certificate-authentication-postgresql-docker-containers).

Following that approach, generate the certificates in the shared folder used by the Vagrant hosts:

```bash
sudo dnf install -y openssl
cd /vagrant
mkdir certs && cd certs
openssl req -new -x509 -days 365 -nodes -out ca.crt -keyout ca.key -subj "/CN=root-ca"
openssl req -new -nodes -out backup-srv.csr -keyout backup-srv.key -subj "/CN=backup-srv"
openssl req -new -nodes -out pg1-srv.csr -keyout pg1-srv.key -subj "/CN=pg1-srv"
openssl req -new -nodes -out pg2-srv.csr -keyout pg2-srv.key -subj "/CN=pg2-srv"
openssl x509 -req -in backup-srv.csr -days 365 -CA ca.crt -CAkey ca.key -CAcreateserial -out backup-srv.crt
openssl x509 -req -in pg1-srv.csr -days 365 -CA ca.crt -CAkey ca.key -CAcreateserial -out pg1-srv.crt
openssl x509 -req -in pg2-srv.csr -days 365 -CA ca.crt -CAkey ca.key -CAcreateserial -out pg2-srv.crt
rm -f *.csr
```

Deploy the certificates to each server (`ca.crt`, `server_name.crt`, and `server_name.key`):

```bash
# on backup-srv
cd /vagrant/certs
sudo mkdir -p /etc/pgbackrest/certs
sudo cp `hostname`.* /etc/pgbackrest/certs
sudo cp ca.crt /etc/pgbackrest/certs
sudo chown -R pgbackrest: /etc/pgbackrest/certs
sudo chmod -R og-rwx /etc/pgbackrest/certs

# on pg1-srv and pg2-srv
cd /vagrant/certs
sudo mkdir -p /etc/pgbackrest/certs
sudo cp `hostname`.* /etc/pgbackrest/certs
sudo cp ca.crt /etc/pgbackrest/certs
sudo chown -R postgres: /etc/pgbackrest/certs
sudo chmod -R og-rwx /etc/pgbackrest/certs
```

-----

# Configuration

We will now configure our stanza, named `demo`.

## Repository server configuration (backup-srv)

```ini
[global]
# repo details
repo1-path=/backup_space
repo1-retention-full=2
repo1-bundle=y
repo1-block=y

# general options
process-max=2
log-level-console=info
log-level-file=detail
compress-type=zst
start-fast=y
delta=y

# tls server options
tls-server-address=*
tls-server-cert-file=/etc/pgbackrest/certs/backup-srv.crt
tls-server-key-file=/etc/pgbackrest/certs/backup-srv.key
tls-server-ca-file=/etc/pgbackrest/certs/ca.crt
tls-server-auth=pg1-srv=demo
tls-server-auth=pg2-srv=demo

[demo]
pg1-host=pg1-srv
pg1-path=/var/lib/pgsql/18/data
pg1-host-type=tls
pg1-host-cert-file=/etc/pgbackrest/certs/backup-srv.crt
pg1-host-key-file=/etc/pgbackrest/certs/backup-srv.key
pg1-host-ca-file=/etc/pgbackrest/certs/ca.crt
```

## PostgreSQL servers (pg1-srv and pg2-srv)

```ini
[global]
repo1-host=backup-srv
repo1-host-user=pgbackrest
repo1-host-type=tls
repo1-host-cert-file=/etc/pgbackrest/certs/pg1-srv.crt
repo1-host-key-file=/etc/pgbackrest/certs/pg1-srv.key
repo1-host-ca-file=/etc/pgbackrest/certs/ca.crt

# general options
compress-type=zst
process-max=2
log-level-console=info
log-level-file=detail

# tls server options
tls-server-address=*
tls-server-cert-file=/etc/pgbackrest/certs/pg1-srv.crt
tls-server-key-file=/etc/pgbackrest/certs/pg1-srv.key
tls-server-ca-file=/etc/pgbackrest/certs/ca.crt
tls-server-auth=backup-srv=demo

[demo]
pg1-path=/var/lib/pgsql/18/data
```

*(Adjust `pg1-srv` to `pg2-srv` on **pg2-srv**.)*

The idea is that each host runs a TLS server to handle requests from the others.
For example:

* The `backup` command on the repository host acts as a TLS client to the PostgreSQL node.
* The `archive-push` command on the PostgreSQL node acts as a TLS client to the repository host.

The PGDG [packages](https://github.com/pgdg-packaging/pgdg-rpms/blob/master/rpm/redhat/main/common/pgbackrest/main/pgbackrest.spec) provide a `pgbackrest.service` systemd unit to start and stop the TLS server.

## Override the service on backup-srv

Create a drop-in override using:

```bash
sudo systemctl edit pgbackrest
```

Add:

```ini
[Service]
User=pgbackrest
Group=pgbackrest
```

Reload systemd:

```bash
sudo systemctl daemon-reload
```

Start the TLS server on all hosts:

```bash
sudo systemctl enable pgbackrest
sudo systemctl start pgbackrest
pgbackrest server-ping
```

The `server-ping` command checks the server's availability (no authentication is performed).

## Configure PostgreSQL archiving on pg1-srv

In `postgresql.conf`:

```
listen_addresses = '*'
archive_mode = on
archive_command = 'pgbackrest --stanza=demo archive-push %p'
```

Restart PostgreSQL:

```bash
sudo systemctl restart postgresql-18.service
```

## Create the stanza and validate configuration

On **backup-srv**:

```bash
sudo -iu pgbackrest pgbackrest --stanza=demo stanza-create
sudo -iu pgbackrest pgbackrest --stanza=demo check
```

-----

# Perform a backup

You can now take your first backup:

```bash
$ sudo -iu pgbackrest pgbackrest --stanza=demo --type=full backup
P00   INFO: backup command begin 2.57.0: ...
P00   INFO: execute non-exclusive backup start:
            backup begins after the requested immediate checkpoint completes
P00   INFO: backup start archive = 000000010000000000000003, lsn = 0/3000028
P00   INFO: check archive for prior segment 000000010000000000000002
P00   INFO: execute non-exclusive backup stop and wait for all WAL segments to archive
P00   INFO: backup stop archive = 000000010000000000000003, lsn = 0/3000120
P00   INFO: check archive for segment(s) 000000010000000000000003:000000010000000000000003
P00   INFO: new backup label = 20251117-134141F
P00   INFO: full backup size = 22.7MB, file total = 970
P00   INFO: backup command end: completed successfully
```

You can run the `info` command from any server correctly configured with pgBackRest.

```bash
# From the PostgreSQL node
$ sudo -iu postgres pgbackrest --stanza=demo info
stanza: demo
    status: ok
    cipher: none

    db (current)
        wal archive min/max (18): 000000010000000000000003/000000010000000000000003

        full backup: 20251117-134141F
            timestamp start/stop: 2025-11-17 13:41:41+00 / 2025-11-17 13:41:43+00
            wal start/stop: 000000010000000000000003 / 000000010000000000000003
            database size: 22.7MB, database backup size: 22.7MB
            repo1: backup size: 2.7MB

# From the backup server
$ sudo -iu pgbackrest pgbackrest --stanza=demo info
stanza: demo
    status: ok
    cipher: none

    db (current)
        wal archive min/max (18): 000000010000000000000003/000000010000000000000003

        full backup: 20251117-134141F
            timestamp start/stop: 2025-11-17 13:41:41+00 / 2025-11-17 13:41:43+00
            wal start/stop: 000000010000000000000003 / 000000010000000000000003
            database size: 22.7MB, database backup size: 22.7MB
            repo1: backup size: 2.7MB
```

-----

# Prepare the servers for Streaming Replication

On **pg1-srv**, create a replication user:

```bash
$ sudo -iu postgres psql
postgres=# CREATE ROLE replic_user WITH LOGIN REPLICATION PASSWORD 'mypwd';
```

Edit `pg_hba.conf`:

```
host   replication   replic_user   pg2-srv   scram-sha-256
```

Reload:

```bash
sudo systemctl reload postgresql-18.service
```

Configure `.pgpass` on **pg2-srv**:

```bash
sudo sh -c "echo '*:*:replication:replic_user:mypwd' >> ~postgres/.pgpass"
sudo chown postgres: ~postgres/.pgpass
sudo chmod 0600 ~postgres/.pgpass
```

-----

# Set up the standby server

Modify `/etc/pgbackrest.conf` on **pg2-srv** to include:

```ini
recovery-option=primary_conninfo=host=pg1-srv user=replic_user
```

Then verify the configuration by running the `info` command. It should print the same output as above.

Restore the backup:

```bash
$ sudo -iu postgres pgbackrest --stanza=demo --type=standby restore
P00   INFO: restore command begin 2.57.0: ...
P00   INFO: repo1: restore backup set 20251117-134141F, recovery will start at ...
P00   INFO: write updated /var/lib/pgsql/18/data/postgresql.auto.conf
P00   INFO: restore global/pg_control
            (performed last to ensure aborted restores cannot be started)
P00   INFO: restore size = 22.7MB, file total = 970
P00   INFO: restore command end: completed successfully
```

This adds the required recovery settings to `postgresql.auto.conf`:

```ini
# Recovery settings generated by pgBackRest restore
primary_conninfo = 'host=pg1-srv user=replic_user'
restore_command = 'pgbackrest --stanza=demo archive-get %f "%p"'
```

The `--type=standby` option creates the `standby.signal` needed for PostgreSQL to start in standby mode. All we have to do now is start PostgreSQL:

```bash
sudo systemctl enable postgresql-18
sudo systemctl start postgresql-18
```

If replication is configured correctly, you should see a `walsender` process on **pg1-srv**:

```bash
$ sudo -iu postgres ps -o pid,cmd fx
    PID CMD
   5768 ps -o pid,cmd fx
   5408 /usr/pgsql-18/bin/postgres -D /var/lib/pgsql/18/data/
   5409  \_ postgres: logger
   5410  \_ postgres: io worker 0
   5411  \_ postgres: io worker 1
   5412  \_ postgres: io worker 2
   5413  \_ postgres: checkpointer
   5414  \_ postgres: background writer
   5416  \_ postgres: walwriter
   5417  \_ postgres: autovacuum launcher
   5418  \_ postgres: archiver last was 000000010000000000000003.00000028.backup
   5419  \_ postgres: logical replication launcher
   5752  \_ postgres: walsender replic_user 192.168.121.28(48720) streaming 0/4001A40
   5376 /usr/bin/pgbackrest server
```

We now have a two-node cluster using _Streaming Replication_ with WAL archiving as a fallback.

-----

# Take backups from the standby server

Extend the configuration on **backup-srv**:

```ini
pg2-host=pg2-srv
pg2-path=/var/lib/pgsql/18/data
pg2-host-type=tls
pg2-host-cert-file=/etc/pgbackrest/certs/backup-srv.crt
pg2-host-key-file=/etc/pgbackrest/certs/backup-srv.key
pg2-host-ca-file=/etc/pgbackrest/certs/ca.crt
backup-standby=y
```

Now take a new backup, where the data files will come from the standby:

```bash
$ sudo -iu pgbackrest pgbackrest --stanza=demo --type=full backup
P00   INFO: backup command begin 2.57.0: ...
P00   INFO: execute non-exclusive backup start:
            backup begins after the requested immediate checkpoint completes
P00   INFO: backup start archive = 000000010000000000000005, lsn = 0/5000028
P00   INFO: wait for replay on the standby to reach 0/5000028
P00   INFO: replay on the standby reached 0/5000028
P00   INFO: check archive for prior segment 000000010000000000000004
P00   INFO: execute non-exclusive backup stop and wait for all WAL segments to archive
P00   INFO: backup stop archive = 000000010000000000000005, lsn = 0/5000120
P00   INFO: check archive for segment(s) 000000010000000000000005:000000010000000000000005
P00   INFO: new backup label = 20251117-135549F
P00   INFO: full backup size = 22.7MB, file total = 970
```

-----

# Conclusion

The TLS server offers a highly performant alternative to SSH for remote operations.
It fits perfectly in containerized environments, where it can significantly improve the user experience.
