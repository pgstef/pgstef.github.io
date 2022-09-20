---
layout: post
title: pgBackRest 2.41 released
date: 2022-09-20 17:30:00 +0200
---

With pgBackRest [2.41](https://github.com/pgbackrest/pgbackrest/releases/tag/release/2.41) just released, a new feature called **backup annotations** is now available.
Let's see in this blog post what this is about.

<!--MORE-->

-----

# Backup annotations

This new feature adds an `--annotation` option to the **backup** command.
We can now attach informative key/value pairs to the backup and the option may be used multiple times to attach multiple annotations.

Example:

```
$ pgbackrest backup --stanza=X --type=full \
  --annotation=comment="this is our initial backup" \
  --annotation=some-other-key="any text you'd like"
```

Internally, the initial annotation is stored in the shared backup information:

```
$ pgbackrest repo-get backup/X/backup.info |grep annotation
..."backup-annotation":{"comment":"this is our initial backup","some-other-key":"any text you'd like"}...
```

That allows to show those annotations in the **info** command.

First, in the _text output_, for a specific backup `--set`:

```
$ pgbackrest info --stanza=X --set=20220920-140720F
...
full backup: 20220920-140720F
    timestamp start/stop: 2022-09-20 14:07:20 / 2022-09-20 14:07:24
    wal start/stop: 00000002000000000000002F / 00000002000000000000002F
    lsn start/stop: 0/2F000028 / 0/2F000100
    database size: 185.3MB, database backup size: 185.3MB
    repo1: backup set size: 13.7MB, backup size: 13.7MB
    database list: bench (16394), postgres (13631)
    annotation(s)
        comment: this is our initial backup
        some-other-key: any text you'd like

$ pgbackrest info --stanza=X --set=20220920-140740F
...
full backup: 20220920-140740F
    timestamp start/stop: 2022-09-20 14:07:40 / 2022-09-20 14:07:46
    wal start/stop: 000000020000000000000031 / 000000020000000000000031
    lsn start/stop: 0/31000028 / 0/31000138
    database size: 185.3MB, database backup size: 185.3MB
    repo1: backup set size: 13.7MB, backup size: 13.7MB
    database list: bench (16394), postgres (13631)
    annotation(s)
        comment: just another backup
```

Then also in the global _json output_:

```bash
$ pgbackrest info --stanza=X --output=json
...{"annotation":{"comment":"this is our initial backup","some-other-key":"any text you'd like"}...
...{"annotation":{"comment":"just another backup"}...
```

pgBackRest 2.41 also adds support for the `--set` option for the _json output_ of the **info** command!

```
$ pgbackrest info --stanza=X --set=20220920-140720F --output=json
...{"annotation":{"comment":"this is our initial backup","some-other-key":"any text you'd like"}...

$ pgbackrest info --stanza=X --set=20220920-140740F --output=json
...{"annotation":{"comment":"just another backup"}...
```

Finally, there's a new **annotate** command to add, modify and remove annotations.

Example:

```
$ pgbackrest annotate --stanza=X --set=20220920-140740F \
  --annotation=comment="another brick in the wall"
$ pgbackrest info --stanza=X --set=20220920-140740F
...
    annotation(s)
        comment: another brick in the wall
```

The keys may also be modified or removed:

```
$ pgbackrest annotate --stanza=X --set=20220920-140740F \
  --annotation=comment= --annotation=new-comment="just a backup"
$ pgbackrest info --stanza=X --set=20220920-140740F
...
annotation(s)
    new-comment: just a backup
```

It is now your turn to update your pgBackRest version and try this new feature :-)

