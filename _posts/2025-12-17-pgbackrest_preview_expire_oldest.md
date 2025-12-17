---
layout: post
title: pgBackRest preview - simplifying manual expiration of oldest backups
date: 2025-12-17 11:35:00 +0100
---

A useful new feature was introduced on 11 December 2025: **Allow expiration of the oldest full backup regardless of current retention**.
Details are available in commit [bf2b276](https://github.com/pgbackrest/pgbackrest/commit/bf2b276dc011464b13039630870934fd42ca473c).

Before this change, it was awkward to expire only the oldest full backup while leaving the existing retention settings untouched.
Users had to temporarily adjust retention (or write a script) to achieve the same result.
Expiring the oldest full backup is particularly helpful when your backup repository is running low on disk space.

Let's see how this works in practice with a simple example.

<!--MORE-->

-----

# Example of expiring the oldest full backup

Let's use a very basic demo setup.

```bash
$ pgbackrest info
...
wal archive min/max (18): 000000010000000000000056/000000010000000300000038

full backup: 20251217-095505F
    wal start/stop: 000000010000000000000056 / 000000010000000000000056

incr backup: 20251217-095505F_20251217-095807I
    wal start/stop: 00000001000000010000001B / 00000001000000010000001B
    backup reference total: 1 full

full backup: 20251217-095902F
    wal start/stop: 000000010000000100000090 / 000000010000000100000090

incr backup: 20251217-095902F_20251217-100124I
    wal start/stop: 000000010000000200000013 / 000000010000000200000013
    backup reference total: 1 full
```

This gives us two full backups, each of them with a linked incremental backup. Note that I have deliberately reduced the output of the `info` command to the information relevant to our test case.

```bash
$ pgbackrest expire --stanza=demo --dry-run
P00   INFO: [DRY-RUN] expire command begin 2.57.0: ...
P00   INFO: [DRY-RUN] repo1: 18-1 no archive to remove
P00   INFO: [DRY-RUN] expire command end: completed successfully
```

If we run the `expire` command, nothing is expired, as I have set `repo1-retention-full=2`, meaning that two full backups should be kept.

Let's now imagine that your backup repository is running out of disk space and you want to remove the oldest backup to free up some space. How would you do that?

Once you have identified the backup set name to expire (`20251217-095505F` in this case), you can run:

```bash
$ pgbackrest expire --stanza=demo --set=20251217-095505F --dry-run
P00   INFO: [DRY-RUN] expire command begin 2.57.0: ...
P00   INFO: [DRY-RUN] repo1: expire adhoc backup set 20251217-095505F, 20251217-095505F_20251217-095807I
P00   INFO: [DRY-RUN] repo1: remove expired backup 20251217-095505F_20251217-095807I
P00   INFO: [DRY-RUN] repo1: remove expired backup 20251217-095505F
P00   INFO: [DRY-RUN] expire command end: completed successfully
```

The `expire --set` option allows you to remove a backup set and all dependent backups (in this case, the incremental one), but the WAL archives are not expired.
Indeed, WAL archives older than the oldest backup are not expired until the retention policy has been met.
This is done to ensure that streaming replicas still have access to older WAL segments that they may need to catch up.

Until this new feature was introduced, the only available option was to adjust the retention settings. In this case, you would count the number of full backups (two), subtract one, and then adjust `--repo1-retention-full=1`.

```bash
$ pgbackrest expire --stanza=demo --repo1-retention-full=1 --dry-run
P00   INFO: [DRY-RUN] expire command begin 2.57.0: ...
P00   INFO: [DRY-RUN] repo1: expire full backup set 20251217-095505F, 20251217-095505F_20251217-095807I
P00   INFO: [DRY-RUN] repo1: remove expired backup 20251217-095505F_20251217-095807I
P00   INFO: [DRY-RUN] repo1: remove expired backup 20251217-095505F
P00   INFO: [DRY-RUN] repo1: 18-1 remove archive, start = 0000000100000000, stop = 00000001000000010000008F
P00   INFO: [DRY-RUN] expire command end: completed successfully
```

If you compare both outputs, you will see that the WAL archives would now be expired. This is a perfectly valid approach, but let's be honest: it is not very user-friendly, as it requires knowing the current retention settings and adjusting them manually.

```bash
$ pgbackrest version
pgBackRest 2.58.0dev
```

The new feature introduces a `--oldest` option for the `expire` command, which expires the oldest full backup set that can be removed (that is, as long as at least one newer full backup remains). This is equivalent to manually decrementing the retention by one, but it is computed automatically. All backups related to the expired full backup set (both differential and incremental) are also expired.
When this option is used, archive retention is also temporarily adjusted so that WAL archives associated with the expired backups can be removed in the same run.

```bash
$ pgbackrest expire --stanza=demo --oldest
P00   INFO: expire command begin 2.58.0dev: ...
P00   INFO: repo1: expire full backup set 20251217-095505F, 20251217-095505F_20251217-095807I
P00   INFO: repo1: remove expired backup 20251217-095505F_20251217-095807I
P00   INFO: repo1: remove expired backup 20251217-095505F
P00   INFO: repo1: 18-1 remove archive, start = 0000000100000000, stop = 00000001000000010000008F
P00   INFO: expire command end: completed successfully
```

This option should be particularly useful for users who want to expire the earliest backup in order to free up space. Since, in this scenario, users will almost always want the corresponding WAL archives to be expired as well, the option internally adjusts archive retention to remove WAL archives that precede the oldest retained full backup.

This option only works if at least one full backup would remain:

```bash
$ pgbackrest info
...
wal archive min/max (18): 000000010000000100000090/000000010000000300000038

full backup: 20251217-095902F
    wal start/stop: 000000010000000100000090 / 000000010000000100000090

incr backup: 20251217-095902F_20251217-100124I
    wal start/stop: 000000010000000200000013 / 000000010000000200000013
    backup reference total: 1 full

$ pgbackrest expire --stanza=demo --oldest
P00   WARN: repo1: expire oldest requested but no eligible full backup to expire
```

-----

# Conclusion

The `--oldest` option addresses a long-standing usability gap by making it trivial to expire the oldest full backup when disk space is tight, without modifying retention settings.
This change comes from a feature request in GitHub issue [#2666](https://github.com/pgbackrest/pgbackrest/issues/2666) and should make a previously awkward operation simpler and safer.
