---
layout: post
title: Combining pgBackRest dedicated repository host and EDB Postgres Advanced Server
---

[pgBackRest](http://pgbackrest.org/) is a well-known powerful backup and restore tool. It offers a lot of possibilities.

In this post, we'll briefly see how to setup a dedicated repository host to backup an _Advanced Server_ 3-nodes cluster.

<!--MORE-->

-----

The repository host will be called **backup-srv** and the 3 _Advanced Server_ nodes in _Streaming Replication_: **pg1-srv**, 
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

First of all, pgBackRest and _Advanced Server_ will both require some dependencies located in the EPEL repository.
If not already done on your system, add it:

```bash
$ sudo yum install -y epel-release
```

Then, to install _Advanced Server_ on **pg1-srv**, **pg2-srv** and **pg3-srv**, please refer to the 
[official documentation](https://www.enterprisedb.com/edb-docs).

In short, install the EDB repository configuration package:

```bash
$ sudo yum -y install https://yum.enterprisedb.com/edbrepos/edb-repo-latest.noarch.rpm
```

Replace the `USERNAME:PASSWORD` variable in the following command with the username and password of a registered EDB user:

```bash
$ sudo sed -i "s@<username>:<password>@USERNAME:PASSWORD@" /etc/yum.repos.d/edb.repo
```

Update the cache and install _Advanced Server_:

```bash
$ sudo yum makecache
$ sudo yum -y install edb-as13-server
```

On **backup-srv**, update the cache and install _Advanced Server_ libs:

```bash
$ sudo yum makecache
$ sudo yum -y install edb-as13-server-libs
```

Now, install pgBackRest and check its version:

```bash
$ sudo yum install -y pgbackrest
$ sudo -u postgres pgbackrest version
pgBackRest 2.31
```

If the latest release package couldn't be found in the EDB repository, you might need to configure the PGDG yum repositories:

```bash
$ sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/\
pgdg-redhat-repo-latest.noarch.rpm
```

Finally, create a basic _Advanced Server_ instance on **pg1-srv**:

```bash
$ sudo -i
root# PGSETUP_INITDB_OPTIONS="-E UTF-8 --data-checksums" /usr/edb/as13/bin/edb-as-13-setup initdb
root# systemctl enable edb-as-13
root# systemctl start edb-as-13
```

## Create a dedicated user on the repository host

The `pgbackrest` user is created to own the backups and archives repository. Any user can own the repository but it is 
best not to use `postgres` or `enterprisedb` to avoid confusion.

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

## Change directories ownership

Since we'll execute pgBackRest commands using the `enterprisedb` system-user on **pg1-srv**, **pg2-srv** and 
**pg3-srv**, change the pgBackRest default directories ownership:

```bash
$ sudo chown -R enterprisedb: /var/lib/pgbackrest
$ sudo chown -R enterprisedb: /var/log/pgbackrest
$ sudo chown -R enterprisedb: /var/spool/pgbackrest
```

## Setup password-less SSH

pgBackRest requires a password-less SSH connection to enable communication between the hosts.

Create the **backup-srv** key pair:

```bash
$ sudo -u pgbackrest ssh-keygen -N "" -t rsa -b 4096 -f /home/pgbackrest/.ssh/id_rsa
$ sudo -u pgbackrest restorecon -R /home/pgbackrest/.ssh
```

Perform the same operation on all the _Advanced Server_ nodes and authorize the public key from the **backup-srv**:

```bash
$ sudo -u enterprisedb ssh-keygen -N "" -t rsa -b 4096 -f /var/lib/edb/.ssh/id_rsa
$ sudo -u enterprisedb restorecon -R /var/lib/edb/.ssh
$ ssh root@backup-srv cat /home/pgbackrest/.ssh/id_rsa.pub | \
    sudo -u enterprisedb tee -a /var/lib/edb/.ssh/authorized_keys
```

Authorize the `enterprisedb` user public keys on the **backup-srv**:

```bash
$ ssh root@pg1-srv cat /var/lib/edb/.ssh/id_rsa.pub | \
    sudo -u pgbackrest tee -a /home/pgbackrest/.ssh/authorized_keys

$ ssh root@pg2-srv cat /var/lib/edb/.ssh/id_rsa.pub | \
    sudo -u pgbackrest tee -a /home/pgbackrest/.ssh/authorized_keys

$ ssh root@pg3-srv cat /var/lib/edb/.ssh/id_rsa.pub | \
    sudo -u pgbackrest tee -a /home/pgbackrest/.ssh/authorized_keys
```

To test the connection:

```bash
# From the Advanced Server nodes
$ sudo -u enterprisedb ssh pgbackrest@backup-srv

# From the backup server
$ sudo -u pgbackrest ssh enterprisedb@pg1-srv
$ sudo -u pgbackrest ssh enterprisedb@pg2-srv
$ sudo -u pgbackrest ssh enterprisedb@pg3-srv
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

We'll now prepare the configuration for our stanza called `demo`. 

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
delta=y

[demo]
pg1-host=pg1-srv
pg1-host-user=enterprisedb
pg1-path=/var/lib/edb/as13/data
```

Update the **pg1-srv** pgBackRest configuration file:

```ini
# /etc/pgbackrest.conf
[global]
repo1-host=backup-srv
repo1-host-user=pgbackrest
log-level-console=info
log-level-file=debug
delta=y

[demo]
pg1-path=/var/lib/edb/as13/data
pg1-user=enterprisedb
pg1-port=5444 
```

Configure archiving in the `postgresql.conf` file:

```
listen_addresses = '*'
archive_mode = on
archive_command = 'pgbackrest --stanza=demo archive-push %p'
```

The _Advanced Server_ instance must be restarted after making these changes and before performing a backup.

```bash
$ sudo systemctl restart edb-as-13
```

Let's finally create the stanza and check the configuration on the **backup-srv**:

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

Let's take our first backup on the **backup-srv** from the **pg1-srv** primary server:

```bash
$ sudo -iu pgbackrest pgbackrest --stanza=demo --type=full backup
P00   INFO: backup command begin 2.31: ...
P00   INFO: execute non-exclusive pg_start_backup(): backup begins after the requested immediate checkpoint completes
P00   INFO: backup start archive = 000000010000000000000004, lsn = 0/4000028
P00   INFO: full backup size = 49.9MB
P00   INFO: execute non-exclusive pg_stop_backup() and wait for all WAL segments to archive
P00   INFO: backup stop archive = 000000010000000000000004, lsn = 0/4000138
P00   INFO: check archive for segment(s) 000000010000000000000004:000000010000000000000004
P00   INFO: new backup label = 20210114-160602F
P00   INFO: backup command end: completed successfully
P00   INFO: expire command begin 2.31: ...
P00   INFO: expire command end: completed successfully
```

The `info` command can be executed from any server where pgBackRest is correctly configured:

```bash
# From the Advanced Server nodes
$ sudo -iu enterprisedb pgbackrest --stanza=demo info
stanza: demo
    status: ok
    cipher: none

    db (current)
        wal archive min/max (13-1): 000000010000000000000002/000000010000000000000004

        full backup: 20210114-160602F
            timestamp start/stop: 2021-01-14 16:06:02 / 2021-01-14 16:06:28
            wal start/stop: 000000010000000000000004 / 000000010000000000000004
            database size: 49.9MB, backup size: 49.9MB
            repository size: 8MB, repository backup size: 8MB

# From the backup server
$ sudo -iu pgbackrest pgbackrest --stanza=demo info
stanza: demo
    status: ok
    cipher: none

    db (current)
        wal archive min/max (13-1): 000000010000000000000002/000000010000000000000004

        full backup: 20210114-160602F
            timestamp start/stop: 2021-01-14 16:06:02 / 2021-01-14 16:06:28
            wal start/stop: 000000010000000000000004 / 000000010000000000000004
            database size: 49.9MB, backup size: 49.9MB
            repository size: 8MB, repository backup size: 8MB
```

-----

# Prepare the servers for Streaming Replication

On the **pg1-srv** server, create a specific user for the replication:

```bash
$ sudo -iu enterprisedb psql -d postgres
postgres=# CREATE ROLE replic_user WITH LOGIN REPLICATION PASSWORD 'mypwd';
```

Configure `pg_hba.conf`:

```
host   replication   replic_user   pg2-srv   md5
host   replication   replic_user   pg3-srv   md5
```

Reload configuration:

```bash
$ sudo systemctl reload edb-as-13
```

Configure `~enterprisedb/.pgpass` on **pg2-srv** and **pg3-srv**:

```bash
$ echo "*:*:replication:replic_user:mypwd" >> ~enterprisedb/.pgpass
$ chown enterprisedb: ~enterprisedb/.pgpass
$ chmod 0600 ~enterprisedb/.pgpass
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
delta=y

[demo]
pg1-path=/var/lib/edb/as13/data
pg1-user=enterprisedb
pg1-port=5444
recovery-option=primary_conninfo=host=pg1-srv user=replic_user port=5444
```

As you may notice, it is the same configuration as on **pg1-srv** with the extra 
[recovery-option](https://pgbackrest.org/configuration.html#section-restore/option-recovery-option).
The idea is to automatically configure the _Streaming Replication_ connection string with the restore command.

Then, make sure the configuration is correct by executing the `info` command. It should print the same output as above.

Restore the backup taken from the **pg1-srv** server on **pg2-srv** and **pg3-srv**:

```bash
$ sudo -iu enterprisedb pgbackrest --stanza=demo --type=standby --no-delta restore
P00   INFO: restore command begin 2.31: ...
P00   INFO: restore backup set 20210114-160602F
P00   INFO: write updated /var/lib/edb/as13/data/postgresql.auto.conf
P00   INFO: restore global/pg_control (performed last to ensure aborted restores cannot be started)
P00   INFO: restore command end: completed successfully
```

The restore will add extra information to the `postgresql.auto.conf` file:

```ini
# Recovery settings generated by pgBackRest restore...
primary_conninfo = 'host=pg1-srv user=replic_user port=5444'
restore_command = 'pgbackrest --stanza=mycluster archive-get %f "%p"'
```

The `--type=standby` option creates the `standby.signal` needed for _Advanced Server_ to start in standby mode. All we have to
do now is to start the _Advanced Server_ instances:

```bash
$ sudo systemctl enable edb-as-13
$ sudo systemctl start edb-as-13
```

If the replication setup is correct, you should see those processes on the **pg1-srv** server:

```
$ sudo -iu enterprisedb ps -o pid,cmd fx
  PID CMD
 2868 ps -o pid,cmd fx
  653 /usr/edb/as13/bin/edb-postmaster -D /var/lib/edb/as13/data
  ... 
 2781  \_ postgres: walsender replic_user ... streaming ...
 2784  \_ postgres: walsender replic_user ... streaming ...
```

We now have a 3-nodes cluster working with _Streaming Replication_ and archives recovery as safety net.

-----

# Take backups from the standby servers

Add the following settings to the pgBackRest configuration file on **backup-srv**, in the `[demo]` stanza section:

```ini
pg2-host=pg2-srv
pg2-host-user=enterprisedb
pg2-path=/var/lib/edb/as13/data
pg3-host=pg3-srv
pg3-host-user=enterprisedb
pg3-path=/var/lib/edb/as13/data
backup-standby=y
```

Now, perform a backup fetching the data from the first standby server found in the configuration:

```bash
$ sudo -iu pgbackrest pgbackrest --stanza=demo --type=full backup
P00   INFO: backup command begin 2.31: ...
P00   INFO: execute non-exclusive pg_start_backup(): backup begins after the requested immediate checkpoint completes
P00   INFO: backup start archive = ..., lsn = ...
P00   INFO: wait for replay on the standby to reach ...
P00   INFO: replay on the standby reached ...
P00   INFO: full backup size = 49.9MB
P00   INFO: execute non-exclusive pg_stop_backup() and wait for all WAL segments to archive
P00   INFO: backup stop archive = ..., lsn = ...
P00   INFO: check archive for segment(s) ...
P00   INFO: new backup label = ...
P00   INFO: backup command end: completed successfully
P00   INFO: expire command begin 2.31: ...
P00   INFO: expire command end: completed successfully
```

-----

# Conclusion

As you can see, the `backup` command is executed from the repository host and the `restore` command on the _Advanced Server_ 
nodes.

The repository host may even be configured to backup multiple _Advanced Server_ clusters by setting up multiple stanzas. 