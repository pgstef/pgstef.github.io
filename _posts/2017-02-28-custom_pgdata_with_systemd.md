---
layout: post
title: Custom PGDATA with systemd
draft: true
---

By default, on CentOS 7, the PostgreSQL v10 data directory is located in /var/lib/pgsql/10/data.

Here's a simple trick to easily place it somewhere else without using symbolic links.

<!--MORE-->

-----

First of all, install PostgreSQL 10:

```bash
# yum install -y https://download.postgresql.org/pub/repos/yum/10/redhat/rhel-7-x86_64/pgdg-centos10-10-2.noarch.rpm
# yum install -y postgresql10-server
```

If you wish to place your data in (e.g.) /pgdata/10/data, create the directory with the good rights:

```bash
# mkdir -p /pgdata/10/data
# chown -R postgres:postgres /pgdata
```

Then, customize the systemd service:

```bash
# cat <<EOF > /etc/systemd/system/postgresql-10.service
.include /lib/systemd/system/postgresql-10.service

[Service]
Environment=PGDATA=/pgdata/10/data
EOF
```

Reload systemd:

```bash
# systemctl daemon-reload
```

Initialize the PostgreSQL data directory:

```bash
# /usr/pgsql-10/bin/postgresql-10-setup initdb
```

Start and enable the service:

```bash
# systemctl enable postgresql-10
# systemctl start postgresql-10
```

And Voila! It's just that simple.