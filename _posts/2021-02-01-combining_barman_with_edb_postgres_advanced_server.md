---
layout: post
title: Combining Barman and EDB Postgres Advanced Server
---

[Barman](https://www.pgbarman.org) allows you to implement disaster recovery solutions for databases with high 
requirements of business continuity.

Traditionally, Barman has always operated remotely via SSH, taking advantage of `rsync` for physical backup operations.
Version 2.0 introduces native support for _Streaming Replication_ backup operations, via `pg_basebackup`.

Choosing one of these two methods is a decision you will need to make. The [official documentation](http://docs.pgbarman.org) 
deeply describes how to implement it and covers a lot of important general considerations. 

In this post, we'll briefly see how to setup a dedicated Barman server to backup an _Advanced Server_ 3-nodes cluster, 
using the SSH method.

<!--MORE-->

-----

The Barman dedicated server will be called **backup-srv** and the 3 _Advanced Server_ nodes in _Streaming Replication_: 
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

First of all, Barman and _Advanced Server_ will both require some dependencies located in the EPEL repository.
If not already done already on your system, add it:

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

Now, install Barman and check its version:

```bash
$ sudo yum install -y barman
$ sudo -iu barman barman -v
2.12
```

On the _Advanced Server_ nodes, to be able to use the `barman-wal-archive` command for WAL archiving, install the following package:

```bash
$ sudo yum install -y barman-cli
$ barman-wal-archive --version
barman-wal-archive 2.12
```

If the latest release packages couldn't be found in the already installed repositories, you might need to configure the 
[2ndQuadrant Public RPM repository](https://rpm.2ndquadrant.com/) as stated in the official documentation.

Finally, create a basic _Advanced Server_ instance on **pg1-srv**:

```bash
$ sudo -i
root# PGSETUP_INITDB_OPTIONS="-E UTF-8 --data-checksums" /usr/edb/as13/bin/edb-as-13-setup initdb
root# systemctl enable edb-as-13
root# systemctl start edb-as-13
```

## Setup password-less SSH

SSH key exchange is a very common practice that is used to implement secure password-less connections between users on 
different machines, and it's needed to use rsync for WAL archiving and for backups.

Create the **backup-srv** key pair:

```bash
$ sudo -u barman ssh-keygen -N "" -t rsa -b 4096 -f /var/lib/barman/.ssh/id_rsa
$ sudo -u barman restorecon -R /var/lib/barman/.ssh
```

Perform the same operation on all the _Advanced Server_ nodes and authorize the public key from the **backup-srv**:

```bash
$ sudo -u enterprisedb ssh-keygen -N "" -t rsa -b 4096 -f /var/lib/edb/.ssh/id_rsa
$ sudo -u enterprisedb restorecon -R /var/lib/edb/.ssh
$ ssh root@backup-srv cat /var/lib/barman/.ssh/id_rsa.pub | \
    sudo -u enterprisedb tee -a /var/lib/edb/.ssh/authorized_keys
```

Authorize the `enterprisedb` user public keys on the **backup-srv**:

```bash
$ ssh root@pg1-srv cat /var/lib/edb/.ssh/id_rsa.pub | \
    sudo -u barman tee -a /var/lib/barman/.ssh/authorized_keys

$ ssh root@pg2-srv cat /var/lib/edb/.ssh/id_rsa.pub | \
    sudo -u barman tee -a /var/lib/barman/.ssh/authorized_keys

$ ssh root@pg3-srv cat /var/lib/edb/.ssh/id_rsa.pub | \
    sudo -u barman tee -a /var/lib/barman/.ssh/authorized_keys
```

To test the connection:

```bash
# From the Advanced Server nodes
$ sudo -u enterprisedb ssh barman@backup-srv

# From the backup server
$ sudo -u barman ssh enterprisedb@pg1-srv
$ sudo -u barman ssh enterprisedb@pg2-srv
$ sudo -u barman ssh enterprisedb@pg3-srv
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

On the **pg1-srv** server, create a specific user for the backups:

```bash
$ sudo -iu enterprisedb psql -d postgres
postgres=# CREATE ROLE barman WITH LOGIN SUPERUSER PASSWORD 'mypwd';
```

Configure `pg_hba.conf`:

```
host   postgres   barman   backup-srv   md5
```

We'll now prepare the **backup-srv** configuration file. Create a new file, called `demo.conf`, in `/etc/barman.d` 
directory, with the following content:

```ini
[demo]
description =  "Example of Advanced Server (via Ssh)"
ssh_command = ssh enterprisedb@pg1-srv
conninfo = host=pg1-srv user=barman dbname=postgres
backup_method = rsync
backup_options = concurrent_backup
reuse_backup = link
archiver = on
```

Configure `~barman/.pgpass`:

```bash
$ echo "*:*:postgres:barman:mypwd" >> ~barman/.pgpass
$ chown barman: ~barman/.pgpass
$ chmod 0600 ~barman/.pgpass
```

By default, the backups and archives are stored locally in the `barman_home` directory: `/var/lib/barman`.

Try the WAL archiving command on **pg1-srv**:

```bash
$ sudo -iu enterprisedb barman-wal-archive backup-srv demo DUMMY --test
Ready to accept WAL files for the server demo
```

Configure archiving in the `postgresql.conf` file:

```
listen_addresses = '*'
archive_mode = on
archive_command = 'barman-wal-archive backup-srv demo %p'
```

The _Advanced Server_ instance must be restarted after making these changes and before performing a backup.

```bash
$ sudo systemctl restart edb-as-13
```

Check the configuration and that WAL archiving is working from **backup-srv**:

```bash
$ sudo -iu barman  barman check demo
Server demo:
	PostgreSQL: OK
	superuser or standard user with backup privileges: OK
	wal_level: OK
	directories: OK
	retention policy settings: OK
	backup maximum age: OK (no last_backup_maximum_age provided)
	compression settings: OK
	failed backups: OK (there are 0 failed backups)
	minimum redundancy requirements: OK (have 0 backups, expected at least 0)
	ssh: OK (PostgreSQL server)
	systemid coherence: OK (no system Id stored on disk)
	archive_mode: OK
	archive_command: OK
	continuous archiving: OK
	archiver errors: OK

$ sudo -iu barman barman switch-wal --force --archive demo
The WAL file ... has been closed on server 'demo'
Waiting for the WAL file ... from server 'demo' (max: 30 seconds)
Processing xlog segments from file archival for demo
	...
```

-----

# Perform a backup

Let's take our first backup on the **backup-srv** from the **pg1-srv** primary server:

```bash
$ sudo -iu barman barman backup demo --wait
Starting backup using rsync-concurrent method for server demo in /var/lib/barman/demo/base/20210115T133837
Backup start at LSN: 0/5000028 (000000010000000000000005, 00000028)
This is the first backup for server demo
WAL segments preceding the current backup have been found:
	000000010000000000000002 from server demo has been removed
	000000010000000000000003 from server demo has been removed
Starting backup copy via rsync/SSH for 20210115T133837
Copy done (time: 1 second)
This is the first backup for server demo
Asking PostgreSQL server to finalize the backup.
Backup size: 49.9 MiB. Actual size on disk: 49.9 MiB (-0.00% deduplication ratio).
Backup end at LSN: 0/5000138 (000000010000000000000005, 00000138)
Backup completed (start time: 2021-01-15 13:38:37.321112, elapsed time: 4 seconds)
Waiting for the WAL file 000000010000000000000005 from server 'demo'
Processing xlog segments from file archival for demo
	000000010000000000000004
	000000010000000000000005
	000000010000000000000005.00000028.backup
```

Get the backups list:

```bash
$ sudo -iu barman barman list-backup demo
demo 20210115T133837 - Fri Jan 15 13:38:39 2021 - Size: 65.9 MiB - WAL Size: 0 B
```

Show more information about that backup:

```bash
$ sudo -iu barman barman show-backup demo 20210115T133837
Backup 20210115T133837:
  Server Name            : demo
  System Id              : 6917964532224614077
  Status                 : DONE
  PostgreSQL Version     : 130001
  PGDATA directory       : /var/lib/edb/as13/data

  Base backup information:
    Disk usage           : 49.9 MiB (65.9 MiB with WALs)
    Incremental size     : 49.9 MiB (-0.00%)
    Timeline             : 1
    Begin WAL            : 000000010000000000000005
    End WAL              : 000000010000000000000005
    WAL number           : 1
    Begin time           : 2021-01-15 13:38:37.226367+00:00
    End time             : 2021-01-15 13:38:39.656877+00:00
    Copy time            : 1 second
    Estimated throughput : 36.0 MiB/s
    Begin Offset         : 40
    End Offset           : 312
    Begin LSN           : 0/5000028
    End LSN             : 0/5000138

  WAL information:
    No of files          : 0
    Disk usage           : 0 B
    Last available       : 000000010000000000000005

  Catalog information:
    Retention Policy     : not enforced
    Previous Backup      : - (this is the oldest base backup)
    Next Backup          : - (this is the latest base backup)
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

On **backup-srv**, restore the backup taken from the **pg1-srv** on **pg2-srv** and **pg3-srv**.

Example for **pg2-srv**:

```bash
$ sudo -iu barman barman recover demo latest /var/lib/edb/as13/data --standby-mode --remote-ssh-command="ssh enterprisedb@pg2-srv"
Starting remote restore for server demo using backup 20210115T133837
Destination directory: /var/lib/edb/as13/data
Remote command: ssh enterprisedb@pg2-srv
Copying the base backup.
Copying required WAL segments.
Generating archive status files
Generating recovery configuration
Identify dangerous settings in destination directory.

IMPORTANT
These settings have been modified to prevent data losses
postgresql.auto.conf line 4: archive_command = false
Recovery completed (start time: 2021-01-15 15:35:32.143974, elapsed time: 5 seconds)

Your PostgreSQL server has been successfully prepared for recovery!
```

Add the _Streaming Replication_ settings to `postgresql.auto.conf`:

```
primary_conninfo = 'host=pg1-srv user=replic_user port=5444'
```

All we have to do now is to start the PostgreSQL instances:

```bash
$ sudo systemctl enable edb-as-13
$ sudo systemctl start edb-as-13
```

If the replication setup is correct, you should see those processes on the **pg1-srv** server:

```
$ sudo -iu enterprisedb ps -o pid,cmd fx
  PID CMD
27643 ps -o pid,cmd fx
27226 /usr/edb/as13/bin/edb-postmaster -D /var/lib/edb/as13/data
  ...
27255  \_ postgres: walsender replic_user ... streaming ...
27639  \_ postgres: walsender replic_user ... streaming ...
```

We now have a 3-nodes cluster working with _Streaming Replication_ and archives recovery as safety net.

-----

# Conclusion

As you can see, backups and restores are executed from the dedicated backup server. This server may even be configured 
to backup multiple _Advanced Server_ clusters by setting up multiple configuration files. 