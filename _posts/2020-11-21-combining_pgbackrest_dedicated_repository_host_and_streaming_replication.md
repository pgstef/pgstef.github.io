---
layout: post
title: Combining pgBackRest dedicated repository host and Streaming Replication
---

[pgBackRest](http://pgbackrest.org/) is a well-known powerful backup and restore tool. It offers a lot of possibilities.

In this post, we'll see how to setup a dedicated repository host to backup a PostgreSQL 3-nodes cluster.

<!--MORE-->

-----

The repository host will be called **backup-srv** and the 3 PostgreSQL nodes in _Streaming Replication_: **pg1-srv**, 
**pg2-srv**, **pg3-srv**. All the nodes will be running on CentOS 7.

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
$ sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/\
pgdg-redhat-repo-latest.noarch.rpm
```

Then, install PostgreSQL on **pg1-srv**, **pg2-srv**, **pg3-srv**: 

```bash
$ sudo yum install -y postgresql13-server postgresql13-contrib
```

The pgBackRest package will require some dependencies located in the EPEL repository. If not already done on your
system, add it:

```bash
$ sudo yum install -y epel-release
```

Install pgBackRest and check its version:

```bash
$ sudo yum install -y pgbackrest
$ sudo -u postgres pgbackrest version
pgBackRest 2.30
```

Finally, create a basic PostgreSQL instance on **pg1-srv**:

```bash
$ export PGSETUP_INITDB_OPTIONS="--data-checksums"
$ /usr/pgsql-13/bin/postgresql-13-setup initdb
$ sudo systemctl enable postgresql-13
$ sudo systemctl start postgresql-13
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

For the purpose of this post, we've created `/backup_space` to store locally our backups and archives. The repository 
can be located on the various supported types as described in the 
[configuration official documentation](https://pgbackrest.org/configuration.html#section-repository).


## Setup password-less SSH

pgBackRest requires a password-less SSH connection to enable communication between the hosts.

Create the **backup-srv** key pair:

```bash
$ sudo -u pgbackrest ssh-keygen -N "" -t rsa -b 4096 -f /home/pgbackrest/.ssh/id_rsa
$ sudo -u pgbackrest restorecon -R /home/pgbackrest/.ssh
```

Perform the same operation on all the PostgreSQL nodes and authorize the public key from the **backup-srv**:

```bash
$ sudo -u postgres ssh-keygen -N "" -t rsa -b 4096 -f /var/lib/pgsql/.ssh/id_rsa
$ sudo -u postgres restorecon -R /var/lib/pgsql/.ssh
$ ssh root@backup-srv cat /home/pgbackrest/.ssh/id_rsa.pub | \
    sudo -u postgres tee -a /var/lib/pgsql/.ssh/authorized_keys
```

Authorize the `postgres` user public keys on the **backup-srv**:

```bash
$ ssh root@pg1-srv cat /var/lib/pgsql/.ssh/id_rsa.pub | \
    sudo -u pgbackrest tee -a /home/pgbackrest/.ssh/authorized_keys

$ ssh root@pg2-srv cat /var/lib/pgsql/.ssh/id_rsa.pub | \
    sudo -u pgbackrest tee -a /home/pgbackrest/.ssh/authorized_keys

$ ssh root@pg3-srv cat /var/lib/pgsql/.ssh/id_rsa.pub | \
    sudo -u pgbackrest tee -a /home/pgbackrest/.ssh/authorized_keys
```

To test the connection:

```bash
# From the PostgreSQL nodes
$ sudo -u postgres ssh pgbackrest@backup-srv

# From the backup server
$ sudo -u pgbackrest ssh postgres@pg1-srv
$ sudo -u pgbackrest ssh postgres@pg2-srv
$ sudo -u pgbackrest ssh postgres@pg3-srv
```

In case you'd use Vagrant boxes, it might be needed to enable the SSH password authentication to proceed with the key 
exchange:

```bash
$ sudo -i
root# sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config    
root# systemctl restart sshd.service
root# passwd
```

-----

# Configuration

We'll now prepare the configuration for our stanza called `mycluster`. 

The stanza name is totally up to you. It is often tempting to name the stanza after the primary instance but a better 
name describes the databases contained in it. Because the stanza name will be used for the primary and all replicas it 
is more appropriate to choose a name that describes the actual function of the cluster, such as app or dw, rather than 
the local instance name, such as main or prod.

Update the **backup-srv** pgBackRest configuration file:

```ini
# /etc/pgbackrest.conf
[global]
repo1-path=/backup_space
repo1-retention-full=2
log-level-console=info
log-level-file=debug
start-fast=y
repo1-cipher-type=aes-256-cbc
repo1-cipher-pass=acbd

[mycluster]
pg1-host=pg1-srv
pg1-path=/var/lib/pgsql/13/data
```

Update the **pg1-srv** pgBackRest configuration file:

```ini
# /etc/pgbackrest.conf
[global]
repo1-host=backup-srv
repo1-host-user=pgbackrest
log-level-console=info
log-level-file=debug

[mycluster]
pg1-path=/var/lib/pgsql/13/data
```

Configure archiving in the `postgresql.conf` file:

```
listen_addresses = '*'
archive_mode = on
archive_command = 'pgbackrest --stanza=mycluster archive-push %p'
```

The PostgreSQL instance must be restarted after making these changes and before performing a backup.

```bash
$ sudo systemctl restart postgresql-13.service
```

Let's finally create the stanza and check the configuration on the **backup-srv**:

```bash
$ sudo -iu pgbackrest pgbackrest --stanza=mycluster stanza-create
...
P00   INFO: stanza-create command end: completed successfully

$ sudo -iu pgbackrest pgbackrest --stanza=mycluster check
...
P00   INFO: check command end: completed successfully
```

-----

# Perform a backup

Let's take our first backup on the **backup-srv** from the **pg1-srv** primary server:

```bash
$ sudo -iu pgbackrest pgbackrest --stanza=mycluster --type=full backup
P00   INFO: backup command begin 2.30: ...
P00   INFO: execute non-exclusive pg_start_backup(): backup begins after the requested immediate checkpoint completes
P00   INFO: backup start archive = 000000010000000000000003, lsn = 0/3000060
P00   INFO: full backup size = 23.1MB
P00   INFO: execute non-exclusive pg_stop_backup() and wait for all WAL segments to archive
P00   INFO: backup stop archive = 000000010000000000000003, lsn = 0/3000138
P00   INFO: check archive for segment(s) 000000010000000000000003:000000010000000000000003
P00   INFO: new backup label = 20201121-165955F
P00   INFO: backup command end: completed successfully
P00   INFO: expire command begin 2.30: ...
P00   INFO: expire command end: completed successfully
```

The `info` command can be executed from any server where pgBackRest is correctly configured:

```bash
# From the PostgreSQL nodes
$ sudo -iu postgres pgbackrest --stanza=mycluster info
stanza: mycluster
    status: ok
    cipher: aes-256-cbc

    db (current)
        wal archive min/max (13-1): 000000010000000000000001/000000010000000000000003

        full backup: 20201121-165955F
            timestamp start/stop: 2020-11-21 16:59:55 / 2020-11-21 17:00:11
            wal start/stop: 000000010000000000000003 / 000000010000000000000003
            database size: 23.1MB, backup size: 23.1MB
            repository size: 2.9MB, repository backup size: 2.9MB

# From the backup server
$ sudo -iu pgbackrest pgbackrest --stanza=mycluster info
stanza: mycluster
    status: ok
    cipher: aes-256-cbc

    db (current)
        wal archive min/max (13-1): 000000010000000000000001/000000010000000000000003

        full backup: 20201121-165955F
            timestamp start/stop: 2020-11-21 16:59:55 / 2020-11-21 17:00:11
            wal start/stop: 000000010000000000000003 / 000000010000000000000003
            database size: 23.1MB, backup size: 23.1MB
            repository size: 2.9MB, repository backup size: 2.9MB
```

-----

# Prepare the servers for Streaming Replication

On **pg1-srv** server, create a specific user for the replication:

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
$ sudo systemctl reload postgresql-13.service
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
log-level-console=info
log-level-file=debug

[mycluster]
pg1-path=/var/lib/pgsql/13/data
recovery-option=primary_conninfo=host=pg1-srv user=replic_user
```

As you may notice, it is the same configuration as on **pg1-srv** with the extra 
[recovery-option](https://pgbackrest.org/configuration.html#section-restore/option-recovery-option).
The idea is to automatically configure the _Streaming Replication_ connection string with the restore command.

Then, make sure the configuration is correct by executing the `info` command. It should print the same output as above.

Restore the backup taken from the **pg1-srv** server on **pg2-srv** and **pg3-srv**:

```bash
$ sudo -iu postgres pgbackrest --stanza=mycluster --type=standby restore
P00   INFO: restore command begin 2.30: ...
P00   INFO: restore backup set 20201121-165955F
P00   INFO: write updated /var/lib/pgsql/13/data/postgresql.auto.conf
P00   INFO: restore global/pg_control (performed last to ensure aborted restores cannot be started)
P00   INFO: restore command end: completed successfully
```

The restore will add extra information to the `postgresql.auto.conf` file:

```ini
# Recovery settings generated by pgBackRest restore...
primary_conninfo = 'host=pg1-srv user=replic_user'
restore_command = 'pgbackrest --stanza=mycluster archive-get %f "%p"'
```

The `--type=standby` option creates the `standby.signal` needed for PostgreSQL to start in standby mode. All we have to
do now is to start the PostgreSQL instances:

```bash
$ sudo systemctl enable postgresql-13
$ sudo systemctl start postgresql-13
```

If the replication setup is correct, you should see those processes on the **pg1-srv** server:

```
# ps -ef |grep postgres
postgres 24773     1  ... usr/pgsql-13/bin/postmaster -D /var/lib/pgsql/13/data/
postgres 25144 24773  ... postgres: walsender replic_user ... streaming 0/40011B8
postgres 25146 24773  ... postgres: walsender replic_user ... streaming 0/40011B8
```

We now have a 3-nodes cluster working with _Streaming Replication_ and archives recovery as safety net.

-----

# Take backups from the standby servers

Add the following settings to the pgBackRest configuration file on **backup-srv**, in the `[mycluster]` stanza section:

```ini
pg2-host=pg2-srv
pg2-path=/var/lib/pgsql/13/data
pg3-host=pg3-srv
pg3-path=/var/lib/pgsql/13/data
backup-standby=y
```

Now, perform a backup fetching the data from the first standby server found in the configuration:

```bash
$ sudo -iu postgres pgbackrest --stanza=mycluster --type=full backup
P00   INFO: backup command begin 2.30: ..
P00   INFO: execute non-exclusive pg_start_backup(): backup begins after the requested immediate checkpoint completes
P00   INFO: backup start archive = 000000010000000000000005, lsn = 0/5000028
P00   INFO: wait for replay on the standby to reach 0/5000028
P00   INFO: replay on the standby reached 0/5000028
P00   INFO: full backup size = 23.1MB
P00   INFO: execute non-exclusive pg_stop_backup() and wait for all WAL segments to archive
P00   INFO: backup stop archive = 000000010000000000000005, lsn = 0/5000138
P00   INFO: check archive for segment(s) 000000010000000000000005:000000010000000000000005
P00   INFO: new backup label = 20201121-182045F
P00   INFO: backup command end: completed successfully
P00   INFO: expire command begin 2.30: ...
P00   INFO: expire command end: completed successfully
```

-----

# Conclusion

As you can see, the `backup` command is executed from the repository host and the `restore` command on the PostgreSQL 
nodes.

The repository host may even be configured to backup multiple PostgreSQL clusters by setting up multiple stanzas. 