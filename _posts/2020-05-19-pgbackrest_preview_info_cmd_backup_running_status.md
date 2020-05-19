---
layout: post
title: pgBackRest preview - Info command and backup/expire running status
---

[pgBackRest](http://pgbackrest.org/) is a well-known powerful backup and 
restore tool. The 2.26 version has been released on Apr 20, 2020. New features 
have been developed since then.

Today, let's have a look at: [add backup/expire running status to the info command](https://github.com/pgbackrest/pgbackrest/commit/e92eb709).

```
This is implemented by checking for a backup lock on the host where info is running so there are a few limitations:

* It is not currently possible to know which command is running: backup, expire, or stanza-*. 
The stanza commands are very unlikely to be running so it's pretty safe to guess backup/expire. 
Command information may be added to the lock file to improve the accuracy of the reported command.

* If the info command is run on a host that is not participating in the backup, e.g. a standby, then there will be no backup lock. 
This seems like a minor limitation since running info on the repo or primary host is preferred.
```

<!--MORE-->

-----

# Explanation

The info command will try to acquire a `lockTypeBackup` lock. This type of 
lock is originally acquired by backup, expire, or stanza-* commands.

The result will be displayed in two ways. First, the status of the text output 
will receive the `backup/expire running` extra message. Then, the JSON output 
will have a specific lock part in the status report. There, we'll know if the 
`backup` lock is held or not. This will allow easier future improvements.

Let's now see some examples.

-----

# Local backup

The first example scenario is a really simple one: local backup.

Before any backup, we typically get this information:

```bash
$ pgbackrest --stanza=my_stanza info
stanza: my_stanza
    status: error (no valid backups)
    cipher: none
    ...

$ pgbackrest --stanza=my_stanza --output=json info
[{
	"archive":...,
	"backup":[],
	"cipher":"none",
	"db":...,
	"name":"my_stanza",
	"status":
	{
		"code":2,
		"lock":
		{
			"backup":{"held":false}
		},
		"message":"no valid backups"
	}
}]
```

Remark here the `lock` part of the JSON output.

Now, let's take a backup and see how the info command react:

```bash 
$ pgbackrest --stanza=my_stanza info
stanza: my_stanza
    status: error (no valid backups, backup/expire running)
    cipher: none
    ...

$ pgbackrest --stanza=my_stanza --output=json info
[{
	"archive":...,
	"backup":[],
	"cipher":"none",
	"db":...,
	"name":"my_stanza",
	"status":
	{
		"code":2,
		"lock":
		{
			"backup":{"held":true}
		},
		"message":"no valid backups"
	}
}]
```

As explained before, the text status is modified and we see the `backup` lock 
held in the JSON output.

After the backup, the text output gets back to the usual situation:

```bash
$ pgbackrest --stanza=my_stanza info
stanza: my_stanza
    status: ok
    cipher: none
    ...

$ pgbackrest --stanza=my_stanza --output=json info
[{
	"archive":...,
	"backup":...,
	"cipher":"none",
	"db":...,
	"name":"my_stanza",
	"status":
	{
		"code":0,
		"lock":
		{
			"backup":{"held":false}
		},
		"message":"ok"
	}
}]
```

Finally, here's an example with a new backup running:

```bash
$ pgbackrest --stanza=my_stanza info
stanza: my_stanza
    status: ok (backup/expire running)
    cipher: none
    ...

$ pgbackrest --stanza=my_stanza --output=json info
[{
	"archive":...,
	"backup":...,
	"cipher":"none",
	"db":...,
	"name":"my_stanza",
	"status":
	{
		"code":0,
		"lock":
		{
			"backup":{"held":true}
		},
		"message":"ok"
	}
}]
```

-----

# Remote backup example

Let's now see how this new feature react with the remote backup configuration.

Even if the backup command runs on the remote host, we still find processes on 
the `pgsql` server itself like:

```bash
pgbackrest ... --remote-type=pg --stanza=my_stanza backup:remote
```

So, running the info command on the `pgsql` server returns:

```bash
$ pgbackrest --stanza=my_stanza info
stanza: my_stanza
    status: ok (backup/expire running)
    cipher: none
    ...

$ pgbackrest --stanza=my_stanza --output=json info
[{
	"archive":...,
	"backup":...,
	"cipher":"none",
	"db":...,
	"name":"my_stanza",
	"status":
	{
		"code":0,
		"lock":
		{
			"backup":{"held":true}
		},
		"message":"ok"
	}
}]
```

On the remote host, we will find processes like:

```bash
pgbackrest --stanza=my_stanza backup --type=full
ssh ... postgres@pgsql-srv pgbackrest ... --remote-type=pg --stanza=my_stanza backup:remote
pgbackrest ... --remote-type=pg --stanza=my_stanza --type=full backup:local
```

This means the backup command runs locally but connects through SSH to the 
`pgsql` server to get the data.

Then, if we execute the info command on the remote host, we get the same information:

```bash
$ pgbackrest --stanza=my_stanza info
stanza: my_stanza
    status: ok (backup/expire running)
    cipher: none
    ...

$ pgbackrest --stanza=my_stanza --output=json info
[{
	"archive":...,
	"backup":...,
	"cipher":"none",
	"db":...,
	"name":"my_stanza",
	"status":
	{
		"code":0,
		"lock":
		{
			"backup":{"held":true}
		},
		"message":"ok"
	}
}]
```

-----

# Conclusion

This feature is pretty basic and may be improved in the future but already 
provides us a good hint. 

Pay attention to the release notes when the new version will be available, 
interesting new features are planned!