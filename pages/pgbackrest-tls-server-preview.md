# TLS Server

The TLS server is an alternative to using SSH for protocol connections to remote hosts.
It has been pushed in the [main branch](https://github.com/pgbackrest/pgbackrest/commit/ccc255d3e05d8ce2b6ac251d1498f71b04098a86)
on 18 Oct 2021.

In this demo setup, the repository host will be called **backup-srv** and the 3 PostgreSQL nodes in _Streaming Replication_:
**pg1-srv**, **pg2-srv**, **pg3-srv**. All the nodes will be running on CentOS 7.

If you're familiar with Vagrant, here's a simple `Vagrantfile` to initiate 4 virtual machines using those names:

```ruby
# Vagrantfile
Vagrant.configure(2) do |config|
    config.vm.box = 'centos/7'
    config.vm.provider 'libvirt' do |lv|
        lv.cpus = 1
        lv.memory = 1024
    end

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

On all the servers, first configure the PGDG yum repositories:


```bash
$ sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

Then, install PostgreSQL on **pg1-srv**, **pg2-srv**, **pg3-srv**: 

```bash
$ sudo yum install -y postgresql14-server postgresql14-contrib
```

Create a basic PostgreSQL instance on **pg1-srv**:

```bash
$ export PGSETUP_INITDB_OPTIONS="--data-checksums"
$ sudo /usr/pgsql-14/bin/postgresql-14-setup initdb
$ sudo systemctl enable postgresql-14
$ sudo systemctl start postgresql-14
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

## Build pgBackRest from sources

We'll first install the pgBackRest package to fetch the dependencies and create the usual directories. Then, we'll build
pgBackRest from sources to fetch the latest modifications (including the TLS Server initial implementation).

As the PGDG packagers explained it [here](https://yum.postgresql.org/news/devel-rpms-require-a-new-repository/), we'll
need some dependencies from extra repositories:

```bash
$ sudo yum install -y epel-release centos-release-scl
```

Install pgBackRest and check its version:

```bash
$ sudo yum install -y pgbackrest
$ pgbackrest version
pgBackRest 2.35
```

Install build prerequisites and build from sources:

```bash
$ sudo yum install -y git make gcc openssl-devel libxml2-devel lz4-devel libzstd-devel bzip2-devel libyaml-devel
$ sudo yum -y install postgresql14-devel
$ git clone https://github.com/pgbackrest/pgbackrest.git --single-branch ~/build/pgbackrest
$ cd ~/build/pgbackrest/src && ./configure CPPFLAGS='-I /usr/pgsql-14/include' LDFLAGS='-L/usr/pgsql-14/lib' && make
```

Replace pgBackRest executable installed earlier by the one we've just built:

```bash
$ sudo mv -f ~/build/pgbackrest/src/pgbackrest /usr/bin
$ pgbackrest version
pgBackRest 2.36dev
```

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
$ sudo -u pgbackrest cp `hostname`.* ~pgbackrest/certs
$ sudo -u pgbackrest cp ca.crt ~pgbackrest/certs
$ sudo chmod -R og-rwx ~pgbackrest/certs

# on pg1-srv, pg2-srv and pg3-srv
$ sudo -u postgres mkdir ~postgres/certs
$ sudo -u postgres cp `hostname`.* ~postgres/certs
$ sudo -u postgres cp ca.crt ~postgres/certs
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
tls-server-address=0.0.0.0
tls-server-port=8432
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

Update the **pg1-srv** pgBackRest configuration file:

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
tls-server-address=0.0.0.0
tls-server-port=8432
tls-server-cert-file=/var/lib/pgsql/certs/pg1-srv.crt
tls-server-key-file=/var/lib/pgsql/certs/pg1-srv.key
tls-server-ca-file=/var/lib/pgsql/certs/ca.crt
tls-server-auth=backup-srv=demo

[demo]
pg1-path=/var/lib/pgsql/14/data
```

The idea is to have a TLS server running on each host to serve the request coming from the other.
In example, the `backup` command running on the repository host will act as client for the TLS server running on the
PostgreSQL node. The `archive-push` command running on the PostgreSQL node will act as client for the TLS server running
on the repository host.

Now that we prepared the configuration files, let's start the TLS server:

```bash
# on backup-srv
$ sudo -iu pgbackrest pgbackrest server-start --log-level-console=error &
$ sudo -iu pgbackrest pgbackrest server-ping

# on pg1-srv, pg2-srv and pg3-srv
$ sudo -iu postgres pgbackrest server-start --log-level-console=error &
$ sudo -iu postgres pgbackrest server-ping
```

The `server-start` command will run until terminated by the SIGINT signal (Control+C). It's up to you to keep it running
(e.g. as background process) and make sure it is actually running. Process monitoring becomes very important in this case.
The `server-ping` command serves as an aliveness check only since no authentication is attempted.

Don't forget to terminate/restart the process when the `tls-*` configuration is changed.

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
P00   INFO: backup command begin 2.36dev: ...
P00   INFO: execute non-exclusive pg_start_backup(): backup begins after the requested
                                                             immediate checkpoint completes
P00   INFO: backup start archive = 000000010000000000000004, lsn = 0/4000028
P00   INFO: execute non-exclusive pg_stop_backup() and wait for all WAL segments to archive
P00   INFO: backup stop archive = 000000010000000000000004, lsn = 0/4000138
P00   INFO: check archive for segment(s) 000000010000000000000004:000000010000000000000004
P00   INFO: new backup label = 20211020-092215F
P00   INFO: full backup size = 25MB, file total = 951
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
        wal archive min/max (14): 000000010000000000000001/000000010000000000000004

        full backup: 20211020-092215F
            timestamp start/stop: 2021-10-20 09:22:15 / 2021-10-20 09:22:21
            wal start/stop: 000000010000000000000004 / 000000010000000000000004
            database size: 25MB, database backup size: 25MB
            repo1: backup set size: 3.1MB, backup size: 3.1MB

# From the backup server
$ sudo -iu pgbackrest pgbackrest --stanza=demo info
stanza: demo
    status: ok
    cipher: none

    db (current)
        wal archive min/max (14): 000000010000000000000001/000000010000000000000004

        full backup: 20211020-092215F
            timestamp start/stop: 2021-10-20 09:22:15 / 2021-10-20 09:22:21
            wal start/stop: 000000010000000000000004 / 000000010000000000000004
            database size: 25MB, database backup size: 25MB
            repo1: backup set size: 3.1MB, backup size: 3.1MB
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

Configure `/etc/pgbackrest.conf` on **pg2-srv** and **pg3-srv**:

```ini
[global]
repo1-host=backup-srv
repo1-host-user=pgbackrest
repo1-host-type=tls
repo1-host-cert-file=/var/lib/pgsql/certs/pg2-srv.crt
repo1-host-key-file=/var/lib/pgsql/certs/pg2-srv.key
repo1-host-ca-file=/var/lib/pgsql/certs/ca.crt

# general options
process-max=2
log-level-console=info
log-level-file=debug

# tls server options
tls-server-address=0.0.0.0
tls-server-port=8432
tls-server-cert-file=/var/lib/pgsql/certs/pg2-srv.crt
tls-server-key-file=/var/lib/pgsql/certs/pg2-srv.key
tls-server-ca-file=/var/lib/pgsql/certs/ca.crt
tls-server-auth=backup-srv=demo

[demo]
pg1-path=/var/lib/pgsql/14/data
recovery-option=primary_conninfo=host=pg1-srv user=replic_user
```
( Remark: obviously, adjust _pg2-srv_ by _pg3-srv_ on **pg3-srv**. )

We've also added the [recovery-option](https://pgbackrest.org/configuration.html#section-restore/option-recovery-option).
The idea is to automatically configure the _Streaming Replication_ connection string with the restore command.

Then, make sure the configuration is correct by executing the `info` command. It should print the same output as above.

Restore the backup taken from the **pg1-srv** server on **pg2-srv** and **pg3-srv**:

```bash
$ sudo -iu postgres pgbackrest --stanza=demo --type=standby restore
P00   INFO: restore command begin 2.36dev: ...
P00   INFO: repo1: restore backup set 20211020-092215F, recovery will start at ...
P00   INFO: write updated /var/lib/pgsql/14/data/postgresql.auto.conf
P00   INFO: restore global/pg_control
(performed last to ensure aborted restores cannot be started)
P00   INFO: restore size = 25MB, file total = 951
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
postgres 31691     1  ... /usr/pgsql-14/bin/postmaster -D /var/lib/pgsql/14/data/
postgres 32093 31691  ... postgres: walsender replic_user ... streaming 0/5000DC8
postgres 32094 31691  ... postgres: walsender replic_user ... streaming 0/5000DC8
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
P00   INFO: backup command begin 2.36dev: ...
P00   INFO: execute non-exclusive pg_start_backup(): backup begins after the requested
                                                             immediate checkpoint completes
P00   INFO: backup start archive = 000000010000000000000006, lsn = 0/6000028
P00   INFO: wait for replay on the standby to reach 0/6000028
P00   INFO: replay on the standby reached 0/6000028
P00   INFO: execute non-exclusive pg_stop_backup() and wait for all WAL segments to archive
P00   INFO: backup stop archive = 000000010000000000000006, lsn = 0/6000100
P00   INFO: check archive for segment(s) 000000010000000000000006:000000010000000000000006
P00   INFO: new backup label = 20211020-095839F
P00   INFO: full backup size = 25MB, file total = 951
P00   INFO: backup command end: completed successfully
```
