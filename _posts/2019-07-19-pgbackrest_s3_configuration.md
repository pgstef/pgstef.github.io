---
layout: post
title: pgBackRest S3 configuration
---

[pgBackRest](http://pgbackrest.org/) is a well-known powerful backup and 
restore tool. 

While the documentation describes all the parameters, it's not always that 
simple to imagine what you can really do with it.

In this post, I will introduce some of the parameters needed to configure the 
access to an Amazon S3 bucket.

<!--MORE-->

-----

# [](#minio)MinIO

For the purpose of this demo setup, we'll install a [MinIO](https://min.io/) 
server, which is an Amazon S3 Compatible Object Storage.

To do so, I followed the guide of [centosblog.com](https://www.centosblog.com/install-configure-minio-object-storage-server-centos-linux/).

-----

## Installation

On a fresh CentOS 7 server:

```bash
$ sudo useradd -s /sbin/nologin -d /opt/minio minio
$ sudo mkdir -p /opt/minio/bin
$ sudo mkdir -p /opt/minio/data
$ sudo yum install -y wget
$ sudo wget https://dl.min.io/server/minio/release/linux-amd64/minio -O /opt/minio/bin/minio
$ sudo chmod +x /opt/minio/bin/minio
$ cat<<EOF | sudo tee "/opt/minio/minio.conf"
MINIO_VOLUMES=/opt/minio/data
MINIO_DOMAIN=minio.local
MINIO_OPTS="--certs-dir /opt/minio/certs --address :443 --compat"
MINIO_ACCESS_KEY="accessKey"
MINIO_SECRET_KEY="superSECRETkey" 
EOF
$ sudo chown -R minio:minio /opt/minio
```

MinIO is installed in `/opt/minio` with a specific system user. The domain name 
we'll use will be `minio.local` and the bucket name will be `pgbackrest`.

The data will be stored in `/opt/minio/data`.

We then have to setup the `hosts` file accordingly:

```bash
$ cat<<EOF | sudo tee "/etc/hosts"
127.0.0.1 pgbackrest.minio.local minio.local s3.eu-west-3.amazonaws.com
EOF
```

-----

## Https

Since we'll need to run MinIO in https mode to be able to work with pgBackRest, 
let's create some self-signed certificates:


```bash
$ mkdir ~/certs
$ cd ~/certs
$ openssl genrsa -out ca.key 2048
$ openssl req -new -x509 -extensions v3_ca -key ca.key -out ca.crt -days 99999 -subj "/C=BE/ST=Country/L=City/O=Organization/CN=some-really-cool-name"
$ openssl genrsa -out server.key 2048
$ openssl req -new -key server.key -out server.csr -subj "/C=BE/ST=Country/L=City/O=Organization/CN=some-really-cool-name"
$ openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 99999 -sha256

$ sudo mkdir -p -m 755 /opt/minio/certs
$ sudo cp server.crt /opt/minio/certs/public.crt
$ sudo cp server.key /opt/minio/certs/private.key
$ sudo chown -R minio:minio /opt/minio/certs
$ sudo chmod -R 644 /opt/minio/certs/public.crt
$ sudo chmod -R 644 /opt/minio/certs/private.key
$ cd ~
```

To show the `accessKey` and `secretKey` generated, use:

```bash
$ sudo grep -E 'accessKey|secretKey' /opt/minio/data/.minio.sys/config/config.json
```

For this example, we've forced those values in the `/opt/minio/minio.conf` file.

-----

## Systemd

Create the systemd service:

```bash
$ cat<<EOF | sudo tee "/etc/systemd/system/minio.service"
[Unit]
Description=Minio
Documentation=https://docs.minio.io
Wants=network-online.target
After=network-online.target
AssertFileIsExecutable=/opt/minio/bin/minio

[Service]
AmbientCapabilities=CAP_NET_BIND_SERVICE
WorkingDirectory=/opt/minio

User=minio
Group=minio

PermissionsStartOnly=true

EnvironmentFile=-/opt/minio/minio.conf
ExecStartPre=/bin/bash -c "[ -n \\"\${MINIO_VOLUMES}\\" ] || echo \\"Variable MINIO_VOLUMES not set in /opt/minio/minio.conf\\""

ExecStart=/opt/minio/bin/minio server \$MINIO_OPTS \$MINIO_VOLUMES

StandardOutput=journal
StandardError=inherit

# Specifies the maximum file descriptor number that can be opened by this process
LimitNOFILE=65536

# Disable timeout logic and wait until process is stopped
TimeoutStopSec=0

# SIGTERM signal is used to stop Minio
KillSignal=SIGTERM

SendSIGKILL=no

SuccessExitStatus=0

[Install]
WantedBy=multi-user.target
EOF

$ sudo systemctl enable minio
$ sudo systemctl start minio
$ sudo firewall-cmd --quiet --permanent --add-service=https
$ sudo firewall-cmd --quiet --reload
$ sudo systemctl status minio
```

Compared to the original service, I've added 
`AmbientCapabilities=CAP_NET_BIND_SERVICE` to allow the service to start on 
port 443. The firewall also allows the https access.

-----

## s3cmd 

To create the `pgbackrest` bucket, we'll use `s3cmd`. The package can be found 
in the `epel-release` repositories.

```bash
$ sudo yum install -y epel-release
$ sudo yum --enablerepo epel-testing install -y s3cmd

$ cat<<EOF > ~/.s3cfg
host_base = minio.local
host_bucket = pgbackrest.minio.local
bucket_location = eu-west-3
use_https = true
access_key = accessKey
secret_key = superSECRETkey
EOF

$ s3cmd mb --no-check-certificate s3://pgbackrest
Bucket 's3://pgbackrest/' created
$ sudo mkdir /opt/minio/data/pgbackrest/repo
$ sudo chown minio: /opt/minio/data/pgbackrest/repo
$ s3cmd ls --no-check-certificate s3://pgbackrest/repo
                       DIR   s3://pgbackrest/repo/
```

We've also created a `repo` sub-directory that we'll use to store 
the pgBackRest files.

-----

# [](#postgresql)PostgreSQL and pgBackRest installation

Let's install PostgreSQL and pgBackRest directly from the PGDG yum repositories:

```bash
$ sudo yum install -y https://download.postgresql.org/pub/repos/yum/11/redhat/\
rhel-7-x86_64/pgdg-centos11-11-2.noarch.rpm
$ sudo yum install -y postgresql11-server postgresql11-contrib
$ sudo yum install -y pgbackrest
```

Check that pgBackRest is correctly installed:

```bash
$ sudo -iu postgres pgbackrest
pgBackRest 2.15 - General help

Usage:
    pgbackrest [options] [command]

Commands:
    archive-get     Get a WAL segment from the archive.
    archive-push    Push a WAL segment to the archive.
    backup          Backup a database cluster.
    check           Check the configuration.
    expire          Expire backups that exceed retention.
    help            Get help.
    info            Retrieve information about backups.
    restore         Restore a database cluster.
    stanza-create   Create the required stanza data.
    stanza-delete   Delete a stanza.
    stanza-upgrade  Upgrade a stanza.
    start           Allow pgBackRest processes to run.
    stop            Stop pgBackRest processes from running.
    version         Get version.

Use 'pgbackrest help [command]' for more information.
```

Create a basic PostgreSQL cluster:

```bash
$ export PGSETUP_INITDB_OPTIONS="--data-checksums"
$ sudo /usr/pgsql-11/bin/postgresql-11-setup initdb
```

Configure archiving in the `postgresql.conf` file:

```bash
$ cat<<EOF | sudo tee -a "/var/lib/pgsql/11/data/postgresql.conf"
archive_mode = on
archive_command = 'pgbackrest --stanza=my_stanza archive-push %p'
EOF
```

Start the PostgreSQL cluster:

```bash
$ sudo systemctl start postgresql-11
```

-----

## Configure pgBackRest to point to the S3 bucket

```bash
$ cat<<EOF | sudo tee "/etc/pgbackrest.conf"
[global]
repo1-path=/repo
repo1-type=s3
repo1-s3-endpoint=minio.local
repo1-s3-bucket=pgbackrest
repo1-s3-verify-tls=n
repo1-s3-key=accessKey
repo1-s3-key-secret=superSECRETkey
repo1-s3-region=eu-west-3

repo1-retention-full=1
process-max=2
log-level-console=info
log-level-file=debug
start-fast=y
delta=y

[my_stanza]
pg1-path=/var/lib/pgsql/11/data
EOF
```

Finally, create the stanza and check that everything works fine:

```bash
$ sudo -iu postgres pgbackrest --stanza=my_stanza stanza-create
...
P00   INFO: stanza-create command end: completed successfully

$ sudo -iu postgres pgbackrest --stanza=my_stanza check
...
P00   INFO: WAL segment 000000010000000000000001 successfully stored...
P00   INFO: check command end: completed successfully
```

Let's finally take our first backup:

```bash
$ sudo -iu postgres pgbackrest --stanza=my_stanza --type=full backup
...
P00   INFO: new backup label = 20190718-194303F
P00   INFO: backup command end: completed successfully
...
```

And that's it! You're ready to go! Enjoy, test it with a real Amazon S3 bucket 
and don't hesitate to perform some performance tests with, for example, `pgbench`.

-----

# pgBackRest configuration explanations

What the documentation says:

-----

## [](#repo1-s3-endpoint)repo1-s3-endpoint

S3 repository endpoint.

The AWS end point should be valid for the selected region.

Remark: that's why a good DNS resolution is needed!

-----

## [](#repo1-s3-bucket)repo1-s3-bucket

S3 repository bucket.

S3 bucket used to store the repository.

pgBackRest repositories can be stored in the bucket root by setting 
`repo-path=/` but it is usually best to specify a prefix, such as `/repo`, so 
logs and other AWS generated content can also be stored in the bucket.

-----

## [](#repo1-s3-verify-tls)repo1-s3-verify-tls

Verify S3 server certificate.

Disables verification of the S3 server certificate. This should only be used 
for testing or other scenarios where a certificate has been self-signed.

`tl;dr`: this option would be, I think, very useful in the official 
`Amazon::S3` perl module...

-----

## [](#repo1-s3-region)repo1-s3-region

S3 repository region.

The AWS region where the bucket was created.

-----

# [](#conclusion)Conclusion

pgBackRest offers a lot of possibilities. As long as we use the default https 
port (443), we can use the S3 configurations with other S3 compatible API's 
like MinIO. Pretty much useful, right?