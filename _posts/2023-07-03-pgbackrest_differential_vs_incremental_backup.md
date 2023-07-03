---
layout: post
title: pgBackRest differential vs incremental backup
date: 2023-07-03 10:25:00 +0200
---

One of the most frequent questions I get is to actually explain the difference between differential and incremental backups in pgBackRest.

As everyone might easily imagine, an incremental backup will only copy the database files (to keep it simple) that have been modified since the last successful backup.
If you have a full backup every Sunday, and then an incremental every other day, the backup from Wednesday will be based on the one from Tuesday.
That also means that if the backup from Monday or Tuesday is broken/corrupted (for any reason), you might not be able to recover from your Wednesday backup.

Now, what is a differential backup then? It is an incremental backup _but_ it will be based on the last successful **full** backup.
So if you have a full backup every Sunday, and then a differential every other day, the backup from Wednesday will be based on the one from Sunday!
It would then doesn't matter if the other differential backups get broken/corrupted, but since you'll always compare what has been modified since Sunday, the size of the differential backups will be bigger over time.

Let's have a concrete example.

<!--MORE-->

-----

# Example

As you probably have understood now, incremental or differential changes the backup it refers to.

Let's set up a database and take some incremental and differential backups:

```bash
# Initialization
$ /usr/pgsql-16/bin/pgbench -i -s 65

# Incremental backups
$ pgbackrest --stanza=demo backup --type=full
$ /usr/pgsql-16/bin/pgbench -n -b simple-update -t 100
$ pgbackrest --stanza=demo backup --type=incr
$ /usr/pgsql-16/bin/pgbench -n -b simple-update -t 100
$ pgbackrest --stanza=demo backup --type=incr

# Differential backups
$ pgbackrest --stanza=demo backup --type=full
$ /usr/pgsql-16/bin/pgbench -n -b simple-update -t 100
$ pgbackrest --stanza=demo backup --type=diff
$ /usr/pgsql-16/bin/pgbench -n -b simple-update -t 100
$ pgbackrest --stanza=demo backup --type=diff
```

Now, look at the repository content using the info command:

```bash
$ pgbackrest --stanza=demo info

# Incremental
full backup: 20230702-142534F
incr backup: 20230702-142534F_20230702-142548I
    backup reference list: 20230702-142534F
incr backup: 20230702-142534F_20230702-142601I
    backup reference list: 20230702-142534F, 20230702-142534F_20230702-142548I

# Differential
full backup: 20230702-142626F
diff backup: 20230702-142626F_20230702-142639D
    backup reference list: 20230702-142626F
diff backup: 20230702-142626F_20230702-142652D
    backup reference list: 20230702-142626F
```

As I mentioned before, the second incremental backup `20230702-142534F_20230702-142601I` will contain the files that have been modified since the previous one.
To be able to restore it, we'd need to restore all the files that haven't changed from the initial full backup `20230702-142534F` and also all the files modified before the first incremental
`20230702-142534F_20230702-142548I` that haven't changed since then. That's actually the information you get in the `backup reference list` section.

Let's have a look at the backup manifest content:

```bash
# First incremental
$ cat 20230702-142534F_20230702-142548I/backup.manifest |grep '"reference":"20230702-142534F"' |wc -l
969

# Second incremental
$ cat 20230702-142534F_20230702-142601I/backup.manifest |grep '"reference":"20230702-142534F"' |wc -l
969
$ cat 20230702-142534F_20230702-142601I/backup.manifest |grep '"reference":"20230702-142534F_20230702-142548I"' |wc -l
3
```

So, in this case, to be able to restore the second incremental backup, we'd need 3 files from the first incremental. If for any reason, one of those 3 files gets corrupted, we might not be able to successfully recover.

And that's the reason why there are differential backups. As you've seen in the `backup reference list` section, the differential backups only need the files from the full backup.

-----

# Conclusion

I hope this short post will now give you a better understanding of what incremental and differential backups mean for pgBackRest.

It is also important to remember that WAL archives are a key part of the backups. They are needed for consistency and let you perform _Point-In-Time Recovery_. But also, as long as you have all the WAL archives needed, in case your latest backup gets corrupted, you should be able to recover from a previous backup: the recovery would only be slower given it would have more WAL entries to replay.
