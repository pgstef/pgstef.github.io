---
layout: post
title: pgBackRest SFTP support
date: 2023-05-26 12:55:00 +0200
---

SFTP support has been added in the [2.46](https://github.com/pgbackrest/pgbackrest/releases/tag/release%2F2.46) release on 22 May 2022.

In this demo setup, the SFTP host will be called **sftp-srv** and the PostgreSQL node **pg-srv**. Both nodes will be running on Rocky Linux 8.

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

    nodes  = 'sftp-srv', 'pg-srv'
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

On the database node, first configure the PGDG repositories and install PostgreSQL:

```bash
$ sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
$ sudo dnf -qy module disable postgresql
$ sudo dnf install -y postgresql15-server postgresql15-contrib
```

Create a basic PostgreSQL instance:

```bash
$ sudo PGSETUP_INITDB_OPTIONS="--data-checksums" /usr/pgsql-15/bin/postgresql-15-setup initdb
$ sudo systemctl enable postgresql-15
$ sudo systemctl start postgresql-15
```

Install pgBackRest and check its version:

```bash
$ sudo dnf install -y epel-release
$ sudo dnf install -y pgbackrest
$ pgbackrest version
pgBackRest 2.46
```

## Create a dedicated user and location on the SFTP server

The `pgbackrest` user is created to own the backups and archives repository on the SFTP server:

```bash
$ sudo groupadd pgbackrest
$ sudo adduser -g pgbackrest -n pgbackrest
$ sudo mkdir -m 750 -p /backup_space
$ sudo chown pgbackrest: /backup_space
$ sudo -u pgbackrest mkdir -m 750 -p /home/pgbackrest/.ssh
```

## Setup SSH communication between the PostgreSQL node and SFTP server

The idea is to generate a dedicated SSH key pair for the SFTP connection and use a secured passphrase with it.

First of all, generate the SSH key pair on the PostgreSQL node:

```bash
$ sudo -u postgres ssh-keygen -f /var/lib/pgsql/.ssh/id_rsa_sftp -t rsa -b 4096 -N "BeSureToGenerateAndUseASecurePassphrase"
```

Then, copy the generated public keys to the SFTP server to authorize it:

```bash
# From sftp-srv
$ ssh root@pg-srv cat /var/lib/pgsql/.ssh/id_rsa_sftp.pub| \
    sudo -u pgbackrest tee -a /home/pgbackrest/.ssh/authorized_keys
```

The connection from the PostgreSQL node should now prompt for the passphrase:

```bash
$ sudo -u postgres ssh pgbackrest@sftp-srv -i /var/lib/pgsql/.ssh/id_rsa_sftp
```

-----

# Configuration

We'll now prepare the configuration for our stanza called `demo`. 

Update the **pg-srv** pgBackRest configuration file:

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
Let's then now create the stanza and check the configuration on the **pg-srv**:

```bash
$ sudo -iu postgres pgbackrest --stanza=demo stanza-create
...
P00   INFO: stanza-create command end: completed successfully

$ sudo -iu postgres pgbackrest --stanza=demo check
...
P00   INFO: check command end: completed successfully
```

We can now take our first backup from **pg-srv**:

```bash
$ sudo -iu postgres pgbackrest --stanza=demo --type=full backup
P00   INFO: backup command begin 2.46: ...
P00   INFO: execute non-exclusive backup start: backup begins after the requested immediate checkpoint completes
P00   INFO: backup start archive = 000000010000000000000003, lsn = 0/3000028
P00   INFO: check archive for prior segment 000000010000000000000002
P00   INFO: execute non-exclusive backup stop and wait for all WAL segments to archive
P00   INFO: backup stop archive = 000000010000000000000003, lsn = 0/3000138
P00   INFO: check archive for segment(s) 000000010000000000000003:000000010000000000000003
P00   INFO: new backup label = 20230525-140532F
P00   INFO: full backup size = 22MB, file total = 966
P00   INFO: backup command end: completed successfully
```

Finally, look at the repository content using the `info` command:

```bash
$ sudo -iu postgres pgbackrest --stanza=demo info
stanza: demo
    status: ok
    cipher: none

    db (current)
        wal archive min/max (15): 000000010000000000000001/000000010000000000000003

        full backup: 20230525-140532F
            timestamp start/stop: 2023-05-25 14:05:32 / 2023-05-25 14:05:44
            wal start/stop: 000000010000000000000003 / 000000010000000000000003
            database size: 22MB, database backup size: 22MB
            repo1: backup set size: 2.9MB, backup size: 2.9MB
```

-----

# Conclusion

The SFTP repository type is pretty easy to setup but as mentioned in the official documentation, the SFTP file transfer is relatively slow.
Increasing `process-max` to parallelize file transfer and using file bundling will help but it would probably be better to use the other supported repository types if you can.

Edit: pgBackRest [2.47](https://github.com/pgbackrest/pgbackrest/releases/tag/release%2F2.47) brought a significant performance improvement of the SFTP storage driver. So it becomes a good option if you really need to use it.
