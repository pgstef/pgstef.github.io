---
layout: default
title: Réplication et sauvegarde PITR avec pitrery
permalink: /2018-07-30-replication_et_sauvegarde_pitr_avec_pitrery/
---

# Configurer la réplication

Installer PostgreSQL 9.6 sur les 2 serveurs (server1 et server2) :

```
# yum install https://download.postgresql.org/pub/repos/yum/9.6/redhat/rhel-7-x86_64/pgdg-centos96-9.6-3.noarch.rpm
# yum install postgresql96-server
# firewall-cmd --permanent --add-service=postgresql
# firewall-cmd --reload
```

Initialiser et démarrer PostgreSQL sur server1 :

```
# /usr/pgsql-9.6/bin/postgresql96-setup initdb
# systemctl enable postgresql-9.6
# systemctl start postgresql-9.6
```

Configurer `postgresql.conf` :

```
##Global
listen_addresses = '*'

##Réplication
wal_level = replica
max_wal_senders = 5
wal_log_hints = on
hot_standby = on
hot_standby_feedback = on

##Archivage
archive_mode = on
archive_command = 'true'
```

Créer un utilisateur pour la réplication :

```
# sudo -iu postgres psql
postgres=# CREATE ROLE replicator WITH LOGIN REPLICATION PASSWORD 'motdepasse';
```

Configurer `pg_hba.conf` (et interdire que l'instance réplique sur elle-même) :

```
host replication replicator server1 reject
host replication replicator server2 md5
```

Configurer `~postgres/.pgpass` sur les 2 serveurs :

```
# echo "*:*:replication:replicator:motdepasse" >> ~postgres/.pgpass
# chown postgres:postgres ~postgres/.pgpass
# chmod 0600 ~postgres/.pgpass
```

Redémarrer PostgreSQL pour prendre en compte le nouveau paramétrage :

```
# systemctl restart postgresql-9.6
```

Initialiser instance secondaire sur server2 :

```
# sudo -iu postgres
$ pg_basebackup -h server1 -U replicator \
   --pgdata /var/lib/pgsql/9.6/data \
   --xlog-method=stream \
   --progress
```

Configurer `recovery.conf` :

```
standby_mode = 'on'
primary_conninfo = 'host=server1 user=replicator'
recovery_target_timeline = 'latest'
```

Modifier `pg_hba.conf` pour interdire que l'instance ne réplique sur elle-même :

```
host replication replicator server1 md5
host replication replicator server2 reject
```

Démarrer PostgreSQL :

```
# systemctl enable postgresql-9.6
# systemctl start postgresql-9.6
```

Pour vérifier que la réplication est fonctionnelle

- sur server2 :

```
# ps -ef |grep postgres |grep "wal receiver"
postgres 18086 18078  0 09:41 ?        00:00:00 postgres: wal receiver process   streaming 0/4000140
```

- sur server1 :

```
# ps -ef |grep postgres |grep "wal sender"
postgres 18265 18239  0 09:41 ?        00:00:00 postgres: wal sender process replicator 192.168.122.219(40366) streaming 0/4000140

# sudo -iu postgres psql
postgres=# SELECT
	client_addr AS client, usename AS user, application_name AS name,
	state, sync_state AS mode,
	(pg_xlog_location_diff(pg_current_xlog_location(),sent_location) / 1024)::bigint as pending,
	(pg_xlog_location_diff(sent_location,write_location) / 1024)::bigint as write,
	(pg_xlog_location_diff(write_location,flush_location) / 1024)::bigint as flush,
	(pg_xlog_location_diff(flush_location,replay_location) / 1024)::bigint as replay,
	(pg_xlog_location_diff(pg_current_xlog_location(),replay_location))::bigint / 1024 as total_lag
FROM pg_stat_replication;
```

# Configurer la sauvegarde PITR

Supposons le répertoire `/mnt/backups` monté sur les 2 serveurs et disponible en RW pour l'utilisateur système `postgres`.

Installer et configurer pitrery sur les 2 serveurs :

```
# yum install pitrery
# cat /etc/pitrery/pitrery.conf
PGDATA="/var/lib/pgsql/9.6/data"
BACKUP_DIR="/mnt/backups/pitrery"
PURGE_KEEP_COUNT=1
STORAGE="rsync"
```

Configurer l'archivage dans `postgresql.conf` :

```
archive_command = '/usr/bin/archive_xlog %p'
```

Vérifier le bon fonctionnement de l'archivage sur server1 :

```
# systemctl reload postgresql-9.6.service
# sudo -iu postgres mkdir -p /mnt/backups/pitrery/archived_xlog
# sudo -iu postgres pitrery check
# sudo -iu postgres psql -c "SELECT pg_switch_xlog();"
```

Générer une sauvegarde :

```
# sudo -iu postgres pitrery backup
```

Modifier le `recovery.conf` de server2 :

```
restore_command = '/usr/bin/restore_xlog %f %p'
```

Après redémarrage, cela permettra à server2 de rattraper automatiquement son retard sur server1 en rejouant les wal archivés en cas de problème avec la réplication.


# Promouvoir proprement l'instance primaire sur server2

Sur server1 :

```
# systemctl stop postgresql-9.6.service
# LC_MESSAGES=C /usr/pgsql-9.6/bin/pg_controldata /var/lib/pgsql/9.6/data | grep -E '(Database cluster state)|(REDO location)'
Database cluster state:               shut down
Latest checkpoint's REDO location:    0/47000028
```

Sur server2 :

```
# sudo -iu postgres psql -c "CHECKPOINT"
# LC_MESSAGES=C /usr/pgsql-9.6/bin/pg_controldata /var/lib/pgsql/9.6/data | grep -E '(Database cluster state)|(REDO location)'
Database cluster state:               in archive recovery
Latest checkpoint's REDO location:    0/47000028
```

Il faut absolument que le statut de `Database cluster state` soit à `in archive recovery` et que les valeurs retournées dans `Latest checkpoint location` soient strictement identiques pour qu'il n'y ait pas de perte de données. 

Si tel est le cas, promouvoir le service via et en vérifier la bonne exécution depuis les logs :

```
# sudo -iu postgres /usr/pgsql-9.6/bin/pg_ctl -D /var/lib/pgsql/9.6/data promote
```

Pour réintégrer server1 comme instance secondaire, y créer le `recovery.conf` avant de redémarrer l'instance :

```
standby_mode = 'on'
primary_conninfo = 'user=replicator host=server2'
recovery_target_timeline = 'latest'
restore_command = '/usr/bin/restore_xlog %f %p'
```

Pour vérifier l'état de l'instance locale :

```
# sudo -iu postgres psql -c "SELECT pg_is_in_recovery();"
```

# Reconstruire l'instance secondaire à partir d'une sauvegarde pitrery

Supposons l'instance primaire démarrée et fonctionnelle sur server1 et une sauvegarde pitrery correcte.

Sur server 2 :

```
# sudo -iu postgres pitrery list -v
List of local backups
----------------------------------------------------------------------
Directory:
  /mnt/backups/pitrery/2018.07.30_11.20.11
  space used: 403M
  storage: rsync
Minimum recovery target time:
  2018-07-30 11:20:11 CEST
PGDATA:
  pg_default 402 MB
  pg_global 497 kB

# sudo -iu postgres pitrery restore
INFO: searching backup directory
INFO: retrieving the PostgreSQL version of the backup
INFO: PostgreSQL version: 9.6
INFO: searching for tablespaces information
INFO: 
INFO: backup directory:
INFO:   /mnt/backups/pitrery/2018.07.30_11.20.11
INFO: 
INFO: destinations directories:
INFO:   PGDATA -> /var/lib/pgsql/9.6/data
INFO: 
INFO: recovery configuration:
INFO:   target owner of the restored files: postgres
INFO:   restore_command = 'restore_xlog -C /etc/pitrery/pitrery.conf %f %p'
INFO: 
INFO: creating /var/lib/pgsql/9.6/data with permission 0700
INFO: transferring PGDATA to /var/lib/pgsql/9.6/data with rsync
INFO: transfer of PGDATA successful
INFO: preparing pg_xlog directory
INFO: preparing recovery.conf file
INFO: done
INFO: 
INFO: please check directories and recovery.conf before starting the cluster
INFO: and do not forget to update the configuration of pitrery if needed
INFO: 
```

Configurer `recovery.conf` :

```
standby_mode = 'on'
primary_conninfo = 'host=server1 user=replicator'
recovery_target_timeline = 'latest'
restore_command = '/usr/bin/restore_xlog %f %p'
```

Modifier `pg_hba.conf` pour interdire que l'instance ne réplique sur elle-même :

```
host replication replicator server1 md5
host replication replicator server2 reject
```

Démarrer l'instance et en vérifier le bon démarrage depuis les logs :

```
# systemctl start postgresql-9.6.service
```

Pour finir, vérifier que la réplication est fonctionnelle comme vu précédemment.