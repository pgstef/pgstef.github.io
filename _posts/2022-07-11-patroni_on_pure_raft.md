---
layout: post
title: Patroni on pure Raft
date: 2022-07-11 09:00:00 +0200
---

Since September 2020 and its 2.0 release, [Patroni](https://github.com/zalando/patroni) is able to rely on the pysyncobj module in order to use python Raft implementation as DCS.

In this post, we will setup a demo cluster to illustrate that feature.

<!--MORE-->

-----

# Installation

For this demo, we will install 3 PostgreSQL nodes in _Streaming Replication_, running on Rocky Linux 8.

If you're familiar with Vagrant, here's a simple `Vagrantfile` to initiate 3 virtual machines:

```ruby
# Vagrantfile
Vagrant.configure(2) do |config|
    config.vm.box = 'rockylinux/8'
    config.vm.provider 'libvirt' do |lv|
        lv.cpus = 1
        lv.memory = 1024
    end
    config.vm.synced_folder ".", "/vagrant", disabled: true

    nodes = 'srv1', 'srv2', 'srv3'
    nodes.each do |node|
        config.vm.define node do |conf|
            conf.vm.hostname = node
        end
    end

    config.vm.provision "shell", inline: <<-SHELL
        #-----------------------------
        sudo dnf install -y bind-utils
        #-----------------------------
    SHELL
end
```

## PostgreSQL

First of all, let's install PostgreSQL on all the nodes:

```bash
$ sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
$ sudo dnf -qy module disable postgresql
$ sudo dnf install -y postgresql14-server postgresql14-contrib
$ sudo systemctl disable postgresql-14
```

Patroni will bootstrap (create) the initial PostgreSQL cluster and be in charge of starting the service, so be sure `systemctl` is disabled for the PostgreSQL service.

## Watchdog

> Watchdog devices are software or hardware mechanisms that will reset the whole system when they do not get a keepalive heartbeat within a specified timeframe. This adds an additional layer of fail safe in case usual Patroni split-brain protection mechanisms fail.

Patroni will be the component interacting with the watchdog device. Set the permissions of the software watchdog:

```bash
$ cat <<EOF | sudo tee /etc/udev/rules.d/99-watchdog.rules
KERNEL=="watchdog", OWNER="postgres", GROUP="postgres"
EOF
$ sudo sh -c 'echo "softdog" >> /etc/modules-load.d/softdog.conf'
$ sudo modprobe softdog
$ sudo chown postgres: /dev/watchdog
```

## Patroni

Install Patroni and its Raft dependencies:

```bash
$ sudo dnf install -y python39
$ sudo -iu postgres pip3 install --user --upgrade pip
$ sudo -iu postgres pip3 install --user setuptools_rust
$ sudo -iu postgres pip3 install --user psycopg[binary]>=3.0.0
$ sudo -iu postgres pip3 install --user patroni[raft]
```
**_Remark:_** since December 2021 and its version 2.1.2, Patroni supports psycopg3.

Since we installed Patroni for the `postgres` user, let's add its location to the user PATH:

```bash
$ sudo -u postgres sh -c 'echo "export PATH=\"/var/lib/pgsql/.local/bin:\$PATH\"" >> ~/.bash_profile'
$ sudo -iu postgres patroni --version
$ sudo -iu postgres syncobj_admin --help
```

Create the data directory for Raft:

```bash
$ sudo mkdir /var/lib/raft
$ sudo chown postgres: /var/lib/raft
```

-----

# Patroni configuration

We will need to define the list of Patroni nodes participating in the Raft consensus cluster.
To fetch it dynamically, you can use this simple shell script (where `srv1 srv2 srv3` are the 3 Patroni hosts):

```bash
# Fetch the IP addresses of all Patroni hosts
MY_IP=$(hostname -I | awk ' {print $1}')
patroni_nodes=( srv1 srv2 srv3 )
i=0
for node in "${patroni_nodes[@]}"
do
  i=$i+1
  target_ip=$(dig +short $node)
  if [[ "$target_ip" = "$MY_IP" ]]; then
    continue
  fi
  target_array[$i]="'$target_ip:5010'"
done
RAFT_PARTNER_ADDRS=$(printf ",%s" "${target_array[@]}")
export RAFT_PARTNER_ADDRS="[${RAFT_PARTNER_ADDRS:1}]"
echo "partner_addrs: $RAFT_PARTNER_ADDRS"
```

Let us now define the Patroni configuration in `/etc/patroni.yml`:

```bash
$ CLUSTER_NAME="demo-cluster-1"
$ MY_NAME=$(hostname --short)
$ MY_IP=$(hostname -I | awk ' {print $1}')
$ cat <<EOF | sudo tee /etc/patroni.yml
scope: $CLUSTER_NAME
namespace: /db/
name: $MY_NAME

restapi:
  listen: "0.0.0.0:8008"
  connect_address: "$MY_IP:8008"
  authentication:
    username: patroni
    password: mySupeSecretPassword

raft:
  data_dir: /var/lib/raft
  self_addr: "$MY_IP:5010"
  partner_addrs: $RAFT_PARTNER_ADDRS

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: false
      use_slots: true
      parameters:
        archive_mode: "on"
        archive_command: "/bin/true"

  initdb:
  - encoding: UTF8
  - data-checksums
  - auth-local: peer
  - auth-host: scram-sha-256

  pg_hba:
  - host replication replicator 0.0.0.0/0 scram-sha-256
  - host all all 0.0.0.0/0 scram-sha-256

  # Some additional users which needs to be created after initializing new cluster
  users:
    admin:
      password: admin%
      options:
        - createrole
        - createdb

postgresql:
  listen: "0.0.0.0:5432"
  connect_address: "$MY_IP:5432"
  data_dir: /var/lib/pgsql/14/data
  bin_dir: /usr/pgsql-14/bin
  pgpass: /tmp/pgpass0
  authentication:
    replication:
      username: replicator
      password: confidential
    superuser:
      username: postgres
      password: my-super-password
    rewind:
      username: rewind_user
      password: rewind_password
  parameters:
    unix_socket_directories: '/var/run/postgresql,/tmp'

watchdog:
  mode: required
  device: /dev/watchdog
  safety_margin: 5

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
EOF
```

Except for `$MY_IP`, `$MY_NAME` and `$RAFT_PARTNER_ADDRS` which are related to the local host, the `patroni.yml` configuration should be the same on all Patroni nodes.

Depending on the Patroni installation source, create the [systemd file](https://github.com/zalando/patroni/blob/master/extras/startup-scripts/patroni.service) if not done during the installation and start the Patroni service:

```bash
$ cat <<EOF | sudo tee /etc/systemd/system/patroni.service
[Unit]
Description=Runners to orchestrate a high-availability PostgreSQL
After=syslog.target network.target

[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=python3 /var/lib/pgsql/.local/bin/patroni /etc/patroni.yml
ExecReload=/bin/kill -s HUP \$MAINPID
KillMode=process
TimeoutSec=30
Restart=no

[Install]
WantedBy=multi-user.target
EOF

$ sudo systemctl daemon-reload
$ sudo systemctl enable patroni
$ sudo systemctl start patroni
```

To check the Raft cluster status, use the `syncobj_admin` command:

```bash
$ sudo -iu postgres syncobj_admin -conn localhost:5010 -status
```

To list the members of the cluster, use the `patronictl` command:

```bash
$ sudo -iu postgres patronictl -c /etc/patroni.yml topology
+--------+-----------------+---------+---------+----+-----------+
| Member | Host            | Role    | State   | TL | Lag in MB |
+ Cluster: demo-cluster-1 (7117556723320621508) ----+-----------+
| srv2   | 192.168.121.12  | Leader  | running |  1 |           |
| + srv1 | 192.168.121.126 | Replica | running |  1 |         0 |
| + srv3 | 192.168.121.194 | Replica | running |  1 |         0 |
+--------+-----------------+---------+---------+----+-----------+
```

-----

# Database connection

Instead of connecting directly to the database server, it is possible to setup HAProxy so the application will be connecting to the proxy instead, which will then forward the request to PostgreSQL. When HAproxy is used for this, it is also possible to route read-only requests to one or more replicas, for load balancing. HAproxy can be installed as an independent server but it can also be installed on the application server or the database server itself.

Another possibility is to use PostgreSQL [client libraries](https://www.postgresql.org/docs/current/libpq-connect.html#LIBPQ-CONNSTRING) like `libpq` and `jdbc` which support client connection fail-over. The connection string contains multiple servers (eg: `host=srv1,srv2,srv3`) and the client library loops over the available hosts to find a connection that is available and capable of read-write or read-only operations. This capability allows clients to follow the primary cluster during a switchover.

Example:

```bash
$ psql "host=srv1,srv2,srv3 dbname=postgres user=admin target_session_attrs=read-write" -c "SELECT pg_is_in_recovery();"
 pg_is_in_recovery
-------------------
 f
(1 row)

$ psql "host=srv1,srv2,srv3 dbname=postgres user=admin target_session_attrs=read-only" -c "SELECT pg_is_in_recovery();"
 pg_is_in_recovery
-------------------
 t
(1 row)

$ psql "host=srv1,srv2,srv3 dbname=postgres user=admin target_session_attrs=read-only" -c "\conninfo"
You are connected to database "postgres" as user "admin" on host "srv2" (address "192.168.121.36") at port "5432".
```
-----

# Automatic failover test

> By default Patroni will set up the watchdog to expire 5 seconds before TTL expires. With the default setup of `loop_wait=10` and `ttl=30` this gives HA loop at least 15 seconds (ttl - safety_margin - loop_wait) to complete before the system gets forcefully reset. By default accessing DCS is configured to time out after 10 seconds. This means that when DCS is unavailable, for example due to network issues, Patroni and PostgreSQL will have at least 5 seconds (ttl - safety_margin - loop_wait - retry_timeout) to come to a state where all client connections are terminated.

Simply run `pgbench` on the leader node and disconnect the VM network interface for a few seconds to notice that a failover may happen very (too?) quickly!

Example:

```
09:22:45,326 INFO: no action. I am (srv1), a secondary, and following a leader (srv2)
09:22:55,333 INFO: no action. I am (srv1), a secondary, and following a leader (srv2)
09:23:05,355 INFO: Got response from srv3 http://192.168.121.194:8008/patroni: {"state": "running", ...}
09:23:07,268 WARNING: Request failed to srv2: GET http://192.168.121.12:8008/patroni (...)
09:23:07,280 INFO: Software Watchdog activated with 25 second timeout, timing slack 15 seconds
09:23:07,319 INFO: promoted self to leader by acquiring session lock
09:23:07 srv1 python3[27101]: server promoting
09:23:07,340 INFO: cleared rewind state after becoming the leader
09:23:08,760 INFO: no action. I am (srv1), the leader with the lock
```

When the network interface comes back up on `srv2`, if it received additional data, the replication might be broken:

```
FATAL:  could not start WAL streaming: ERROR:  requested starting point 0/D9000000 on timeline 1 is not in this server's history
  DETAIL:  This server's history forked from timeline 1 at 0/CDBDA4A8.
LOG:  new timeline 2 forked off current database system timeline 1 before current recovery point 0/D9683DC8
```

Since we didn't configure Patroni to use `pg_rewind`, the replication lag might grow very quickly:

```bash
$ sudo -iu postgres patronictl -c /etc/patroni.yml list
+--------+-----------------+---------+---------+----+-----------+
| Member | Host            | Role    | State   | TL | Lag in MB |
+ Cluster: demo-cluster-1 (7117556723320621508) ----+-----------+
| srv1   | 192.168.121.126 | Leader  | running |  2 |           |
| srv2   | 192.168.121.12  | Replica | running |  1 |       169 |
| srv3   | 192.168.121.194 | Replica | running |  2 |         0 |
+--------+-----------------+---------+---------+----+-----------+
```

Hopefully, we defined a `maximum_lag_on_failover` to prevent the failover on the failing standby:

```bash
$ sudo -iu postgres patronictl -c /etc/patroni.yml switchover --candidate srv2 --force
Current cluster topology
+--------+-----------------+---------+---------+----+-----------+
| Member | Host            | Role    | State   | TL | Lag in MB |
+ Cluster: demo-cluster-1 (7117556723320621508) ----+-----------+
| srv1   | 192.168.121.126 | Leader  | running |  2 |           |
| srv2   | 192.168.121.12  | Replica | running |  1 |      1046 |
| srv3   | 192.168.121.194 | Replica | running |  2 |         0 |
+--------+-----------------+---------+---------+----+-----------+
Switchover failed, details: 503, Switchover failed
```

From Patroni logs:

```
INFO: Member srv2 exceeds maximum replication lag
WARNING: manual failover: no healthy members found, failover is not possible
```

We have to reinitialize the failing standby:

```bash
$ sudo -iu postgres patronictl -c /etc/patroni.yml reinit demo-cluster-1 srv2
+--------+-----------------+---------+---------+----+-----------+
| Member | Host            | Role    | State   | TL | Lag in MB |
+ Cluster: demo-cluster-1 (7117556723320621508) ----+-----------+
| srv1   | 192.168.121.126 | Leader  | running |  2 |           |
| srv2   | 192.168.121.12  | Replica | running |  1 |      1575 |
| srv3   | 192.168.121.194 | Replica | running |  2 |         0 |
+--------+-----------------+---------+---------+----+-----------+
Are you sure you want to reinitialize members srv2? [y/N]: y
Success: reinitialize for member srv2
```

From Patroni logs:

```
INFO: Removing data directory: /var/lib/pgsql/14/data
INFO: Lock owner: srv1; I am srv2
INFO: reinitialize in progress
...
```

The `reinit` step is by default performed using `pg_basebackup` without the fast checkpoint mode. So, depending on the checkpoint configuration and database size, it may take a lot of time.

-----

# Conclusion

It is very important to understand the parameters affecting the automatic failover kick-off and the consequences of a switchover/failover, or even the impact of not using `pg_rewind`.

As usual, testing its own configuration is important and once in production what's even more important is to have a good monitoring and alerting system!
