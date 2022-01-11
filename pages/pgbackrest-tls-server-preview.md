# TLS Server

The TLS server is an alternative to using SSH for protocol connections to remote hosts.
It has been pushed in the [2.37](https://github.com/pgbackrest/pgbackrest/releases/tag/release%2F2.37) release on 3 Jan 2022.

In this demo setup, the repository host will be called **backup-srv** and the 3 PostgreSQL nodes in _Streaming Replication_:
**pg1-srv**, **pg2-srv**, **pg3-srv**. All the nodes will be running on Rocky Linux 8.

If you're familiar with Vagrant, here's a simple `Vagrantfile` to initiate 4 virtual machines using those names:

```ruby
# Vagrantfile
Vagrant.configure(2) do |config|
    config.vm.box = 'rockylinux/8'
    config.vm.provider 'libvirt' do |lv|
        lv.cpus = 1
        lv.memory = 1024
    end

    # share the default vagrant folder
    config.vm.synced_folder ".", "/vagrant", type: "nfs"

    nodes  = 'backup-srv', 'pg1-srv', 'pg2-srv', 'pg3-srv'
    nodes.each do |node|
        config.vm.define node do |conf|
            conf.vm.hostname = node
        end
    end
end
```

-----

# Installation

On all the servers, first configure the PGDG repositories:

```bash
$ sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

Then, install PostgreSQL on **pg1-srv**, **pg2-srv**, **pg3-srv**: 

```bash
$ sudo dnf -qy module disable postgresql
$ sudo dnf install -y postgresql14-server postgresql14-contrib
```

Create a basic PostgreSQL instance on **pg1-srv**:

```bash
$ export PGSETUP_INITDB_OPTIONS="--data-checksums"
$ sudo /usr/pgsql-14/bin/postgresql-14-setup initdb
$ sudo systemctl enable postgresql-14
$ sudo systemctl start postgresql-14
```

Install pgBackRest and check its version:

```bash
$ sudo dnf install -y epel-release
$ sudo dnf install -y pgbackrest
$ pgbackrest version
pgBackRest 2.37
```

## Create a dedicated user on the repository host

The `pgbackrest` user is created to own the backups and archives repository. Any user can own the repository but it is 
best not to use `postgres` to avoid confusion.

Create the user and the repository location on **backup-srv**:

```bash
$ sudo groupadd pgbackrest
$ sudo adduser -g pgbackrest -n pgbackrest
$ sudo chown pgbackrest: /var/log/pgbackrest
$ sudo mkdir /backup_space
$ sudo chown pgbackrest: /backup_space
```

For the purpose of this demo, we've created `/backup_space` to store locally our backups and archives. The repository
can be located on the various supported types as described in the 
[configuration official documentation](https://pgbackrest.org/configuration.html#section-repository).

-----

# Certificates generation

The TLS server implementation relies on certificates.

A good practical example on how to generate certificates and use it to authenticate PostgreSQL connections can be found
[here](https://blog.crunchydata.com/blog/ssl-certificate-authentication-postgresql-docker-containers).

Based on that example, let's generate our certificates:

```bash
$ mkdir certs && cd certs
$ openssl req -new -x509 -days 365 -nodes -out ca.crt -keyout ca.key -subj "/CN=root-ca"
$ openssl req -new -nodes -out backup-srv.csr -keyout backup-srv.key -subj "/CN=backup-srv"
$ openssl req -new -nodes -out pg1-srv.csr -keyout pg1-srv.key -subj "/CN=pg1-srv"
$ openssl req -new -nodes -out pg2-srv.csr -keyout pg2-srv.key -subj "/CN=pg2-srv"
$ openssl req -new -nodes -out pg3-srv.csr -keyout pg3-srv.key -subj "/CN=pg3-srv"
$ openssl x509 -req -in backup-srv.csr -days 365 -CA ca.crt -CAkey ca.key -CAcreateserial -out backup-srv.crt
$ openssl x509 -req -in pg1-srv.csr -days 365 -CA ca.crt -CAkey ca.key -CAcreateserial -out pg1-srv.crt
$ openssl x509 -req -in pg2-srv.csr -days 365 -CA ca.crt -CAkey ca.key -CAcreateserial -out pg2-srv.crt
$ openssl x509 -req -in pg3-srv.csr -days 365 -CA ca.crt -CAkey ca.key -CAcreateserial -out pg3-srv.crt
$ rm *.csr
```

Then, you have to deploy it on each server (`ca.crt` + `server_name.crt` + `server_name.key`).

Here's an example using a shared folder between the vagrant hosts:

```bash
# on backup-srv
$ sudo -u pgbackrest mkdir ~pgbackrest/certs
$ sudo cp `hostname`.* ~pgbackrest/certs
$ sudo cp ca.crt ~pgbackrest/certs
$ sudo chown -R pgbackrest: ~pgbackrest/certs
$ sudo chmod -R og-rwx ~pgbackrest/certs

# on pg1-srv, pg2-srv and pg3-srv
$ sudo -u postgres mkdir ~postgres/certs
$ sudo cp `hostname`.* ~postgres/certs
$ sudo cp ca.crt ~postgres/certs
$ sudo chown -R postgres: ~postgres/certs
$ sudo chmod -R og-rwx ~postgres/certs
```

-----

# Configuration

We'll now prepare the configuration for our stanza called `demo`. 

Update the **backup-srv** pgBackRest configuration file:

```ini
# /etc/pgbackrest.conf
[global]
# repo details
repo1-path=/backup_space
repo1-retention-full=2

# general options
process-max=2
log-level-console=info
log-level-file=debug
start-fast=y
delta=y

# tls server options
tls-server-address=*
tls-server-cert-file=/home/pgbackrest/certs/backup-srv.crt
tls-server-key-file=/home/pgbackrest/certs/backup-srv.key
tls-server-ca-file=/home/pgbackrest/certs/ca.crt
tls-server-auth=pg1-srv=demo
tls-server-auth=pg2-srv=demo
tls-server-auth=pg3-srv=demo

[demo]
pg1-host=pg1-srv
pg1-path=/var/lib/pgsql/14/data
pg1-host-type=tls
pg1-host-cert-file=certs/backup-srv.crt
pg1-host-key-file=certs/backup-srv.key
pg1-host-ca-file=certs/ca.crt
```

Update the **pg1-srv**, **pg2-srv** and **pg3-srv** pgBackRest configuration file:

```ini
# /etc/pgbackrest.conf
[global]
repo1-host=backup-srv
repo1-host-user=pgbackrest
repo1-host-type=tls
repo1-host-cert-file=/var/lib/pgsql/certs/pg1-srv.crt
repo1-host-key-file=/var/lib/pgsql/certs/pg1-srv.key
repo1-host-ca-file=/var/lib/pgsql/certs/ca.crt

# general options
process-max=2
log-level-console=info
log-level-file=debug

# tls server options
tls-server-address=*
tls-server-cert-file=/var/lib/pgsql/certs/pg1-srv.crt
tls-server-key-file=/var/lib/pgsql/certs/pg1-srv.key
tls-server-ca-file=/var/lib/pgsql/certs/ca.crt
tls-server-auth=backup-srv=demo

[demo]
pg1-path=/var/lib/pgsql/14/data
```

( Remark: obviously, adjust _pg1-srv_ by _pg2-srv_ on **pg2-srv** and _pg3-srv_ on **pg3-srv**. )

The idea is to have a TLS server running on each host to serve the request coming from the other.
In example, the `backup` command running on the repository host will act as client for the TLS server running on the
PostgreSQL node. The `archive-push` command running on the PostgreSQL node will act as client for the TLS server running
on the repository host.

The [`server`](https://pgbackrest.org/command.html#command-server) command can be used to start the TLS server and will
run until terminated by the SIGINT signal (Control+C).

If not done by the PGDG [package](https://git.postgresql.org/gitweb/?p=pgrpms.git;a=blob;f=rpm/redhat/main/common/pgbackrest/main/pgbackrest.spec)
already, create a service file on the **backup-srv** server:

```ini
# /etc/systemd/system/pgbackrest.service
[Unit]
Description=pgBackRest Server
Documentation=https://pgbackrest.org/configuration.html
After=network.target
StartLimitIntervalSec=0

[Service]
Type=simple
User=pgbackrest
Restart=always
RestartSec=1
ExecStart=/usr/bin/pgbackrest server
ExecReload=kill -HUP $MAINPID
KillMode=mixed
KillSignal=SIGINT

[Install]
WantedBy=multi-user.target
```

Start the server and check it's running:

```bash
$ sudo systemctl daemon-reload
$ sudo systemctl enable pgbackrest
$ sudo systemctl start pgbackrest
$ sudo systemctl status pgbackrest
$ pgbackrest server-ping
```

The `server-ping` command serves as an aliveness check only since no authentication is attempted.

Don't forget to terminate/restart the process when the `tls-*` configuration is changed.

Now, apply the same steps (service configuration and start) on the on **pg1-srv**, **pg2-srv** and **pg3-srv** using this configuration:

```ini
# /etc/systemd/system/pgbackrest.service
[Unit]
Description=pgBackRest Server
Documentation=https://pgbackrest.org/configuration.html
After=network.target
StartLimitIntervalSec=0

[Service]
Type=simple
User=postgres
Restart=always
RestartSec=1
ExecStart=/usr/bin/pgbackrest server
ExecReload=kill -HUP $MAINPID
KillMode=mixed
KillSignal=SIGINT

[Install]
WantedBy=multi-user.target
```

Now, let's configure PostgreSQL archiving in the `postgresql.conf` file (on **pg1-srv**):

```
listen_addresses = '*'
archive_mode = on
archive_command = 'pgbackrest --stanza=demo archive-push %p'
```

The PostgreSQL instance must be restarted after making these changes and before performing a backup.

```bash
$ sudo systemctl restart postgresql-14.service
```

Let's finally create the stanza and check the configuration on **backup-srv**:

```bash
$ sudo -iu pgbackrest pgbackrest --stanza=demo stanza-create
...
P00   INFO: stanza-create command end: completed successfully

$ sudo -iu pgbackrest pgbackrest --stanza=demo check
...
P00   INFO: check command end: completed successfully
```

-----

# Perform a backup

If everything is working correctly, we can now take our first backup:

```bash
$ sudo -iu pgbackrest pgbackrest --stanza=demo --type=full backup
P00   INFO: backup command begin 2.37: ..
P00   INFO: execute non-exclusive pg_start_backup(): backup begins after the requested
                                                             immediate checkpoint completes
P00   INFO: backup start archive = 000000010000000000000002, lsn = 0/2000028
P00   INFO: check archive for segment 000000010000000000000002
P00   INFO: execute non-exclusive pg_stop_backup() and wait for all WAL segments to archive
P00   INFO: backup stop archive = 000000010000000000000003, lsn = 0/3000088
P00   INFO: check archive for segment(s) 000000010000000000000002:000000010000000000000003
P00   INFO: new backup label = 20220111-095427F
P00   INFO: full backup size = 25.2MB, file total = 951
P00   INFO: backup command end: completed successfully
```

The `info` command can then be executed from any server where pgBackRest is correctly configured:

```bash
# From the PostgreSQL node
$ sudo -iu postgres pgbackrest --stanza=demo info
stanza: demo
    status: ok
    cipher: none

    db (current)
        wal archive min/max (14): 000000010000000000000001/000000010000000000000003

        full backup: 20220111-095427F
            timestamp start/stop: 2022-01-11 09:54:27 / 2022-01-11 09:54:35
            wal start/stop: 000000010000000000000002 / 000000010000000000000003
            database size: 25.2MB, database backup size: 25.2MB
            repo1: backup set size: 3.2MB, backup size: 3.2MB

# From the backup server
$ sudo -iu pgbackrest pgbackrest --stanza=demo info
stanza: demo
    status: ok
    cipher: none

    db (current)
        wal archive min/max (14): 000000010000000000000001/000000010000000000000003

        full backup: 20220111-095427F
            timestamp start/stop: 2022-01-11 09:54:27 / 2022-01-11 09:54:35
            wal start/stop: 000000010000000000000002 / 000000010000000000000003
            database size: 25.2MB, database backup size: 25.2MB
            repo1: backup set size: 3.2MB, backup size: 3.2MB
```

-----

# Prepare the servers for Streaming Replication

On **pg1-srv**, create a specific user for the replication:

```bash
$ sudo -iu postgres psql
postgres=# CREATE ROLE replic_user WITH LOGIN REPLICATION PASSWORD 'mypwd';
```

Configure `pg_hba.conf`:

```
host   replication   replic_user   pg2-srv   scram-sha-256
host   replication   replic_user   pg3-srv   scram-sha-256
```

Reload configuration:

```bash
$ sudo systemctl reload postgresql-14.service
```

Configure `~postgres/.pgpass` on **pg2-srv** and **pg3-srv**:

```bash
$ echo "*:*:replication:replic_user:mypwd" >> ~postgres/.pgpass
$ chown postgres: ~postgres/.pgpass
$ chmod 0600 ~postgres/.pgpass
```

-----

# Setup the standby servers

Adjust `/etc/pgbackrest.conf` on **pg2-srv** and **pg3-srv** to add the [recovery-option](https://pgbackrest.org/configuration.html#section-restore/option-recovery-option).
The idea is to automatically configure the _Streaming Replication_ connection string with the restore command.

```ini
recovery-option=primary_conninfo=host=pg1-srv user=replic_user
```

Then, make sure the configuration is correct by executing the `info` command. It should print the same output as above.

Restore the backup taken from the **pg1-srv** server on **pg2-srv** and **pg3-srv**:

```bash
$ sudo -iu postgres pgbackrest --stanza=demo --type=standby restore
P00   INFO: restore command begin 2.37: ...
P00   INFO: repo1: restore backup set 20220111-095427F, recovery will start at ...
P00   INFO: write updated /var/lib/pgsql/14/data/postgresql.auto.conf
P00   INFO: restore global/pg_control
(performed last to ensure aborted restores cannot be started)
P00   INFO: restore size = 25.2MB, file total = 951
P00   INFO: restore command end: completed successfully
```

The restore will add extra information to the `postgresql.auto.conf` file:

```ini
# Recovery settings generated by pgBackRest restore on ...
primary_conninfo = 'host=pg1-srv user=replic_user'
restore_command = 'pgbackrest --stanza=demo archive-get %f "%p"'
```

The `--type=standby` option creates the `standby.signal` needed for PostgreSQL to start in standby mode. All we have to
do now is to start the PostgreSQL instances:

```bash
$ sudo systemctl enable postgresql-14
$ sudo systemctl start postgresql-14
```

If the replication setup is correct, you should see those processes on the **pg1-srv** server:

```
# ps -ef |grep postgres
postgres 1735    1 ... /usr/pgsql-14/bin/postmaster -D /var/lib/pgsql/14/data/
postgres 1947 1735 ... postgres: walsender replic_user ... streaming 0/4000CE0
postgres 1948 1735 ... postgres: walsender replic_user ... streaming 0/4000CE0
```

We now have a 3-nodes cluster working with _Streaming Replication_ and archives recovery as safety net.

-----

# Take backups from the standby servers

Add the following settings to the pgBackRest configuration file on **backup-srv**, in the `[demo]` stanza section:

```ini
pg2-host=pg2-srv
pg2-path=/var/lib/pgsql/14/data
pg2-host-type=tls
pg2-host-cert-file=certs/backup-srv.crt
pg2-host-key-file=certs/backup-srv.key
pg2-host-ca-file=certs/ca.crt
pg3-host=pg3-srv
pg3-path=/var/lib/pgsql/14/data
pg3-host-type=tls
pg3-host-cert-file=certs/backup-srv.crt
pg3-host-key-file=certs/backup-srv.key
pg3-host-ca-file=certs/ca.crt
backup-standby=y
```

Now, perform a backup fetching the data from the first standby server found in the configuration:

```bash
$ sudo -iu pgbackrest pgbackrest --stanza=demo --type=full backup
P00   INFO: backup command begin 2.37: ...
P00   INFO: execute non-exclusive pg_start_backup(): backup begins after the requested
                                                             immediate checkpoint completes
P00   INFO: backup start archive = 000000010000000000000005, lsn = 0/5000028
P00   INFO: wait for replay on the standby to reach 0/5000028
P00   INFO: replay on the standby reached 0/5000028
P00   INFO: check archive for prior segment 000000010000000000000004
P00   INFO: execute non-exclusive pg_stop_backup() and wait for all WAL segments to archive
P00   INFO: backup stop archive = 000000010000000000000005, lsn = 0/5000138
P00   INFO: check archive for segment(s) 000000010000000000000005:000000010000000000000005
P00   INFO: new backup label = 20220111-100417F
P00   INFO: full backup size = 25.2MB, file total = 951
P00   INFO: backup command end: completed successfully
```
