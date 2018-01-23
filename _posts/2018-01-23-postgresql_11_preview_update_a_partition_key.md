---
layout: post
title: PostgreSQL 11 preview - update a partition key
---

In the case of a partitioned table, updating a row might cause it to no
longer satisfy the partition constraint of the containing partition.

There's a recent commit for version 11 modifying PostgreSQL behavior in 
that case.

<!--MORE-->

-----

With PostgreSQL 10, that case threw an error.

For example:

```sql
# CREATE TABLE t1 (c1 int) PARTITION BY RANGE (c1);
# CREATE TABLE t1_part_1 PARTITION OF t1 FOR VALUES FROM (1) to (10);
# CREATE TABLE t1_part_2 PARTITION OF t1 FOR VALUES FROM (10) to (20);
# INSERT INTO t1 VALUES(generate_series(1,9));

# SELECT COUNT(*) FROM t1_part_1;
 count 
-------
     9
(1 row)

# SELECT COUNT(*) FROM t1_part_2;
 count 
-------
     0
(1 row)
```

Let's try this simple `UPDATE`:

```sql
v10=# UPDATE t1 SET c1 = c1 + 10;
ERROR:  new row for relation "t1_part_1" violates partition constraint
DETAIL:  Failing row contains (11).
```

-----

Some days ago, there was this new feature added in version 11 which is currently under development:

```
commit: 2f178441044be430f6b4d626e4dae68a9a6f6cec
author: Robert Haas <rhaas@postgresql.org>	
date:   Fri, 19 Jan 2018 21:33:06 +0100
Allow UPDATE to move rows between partitions.

When an UPDATE causes a row to no longer match the partition
constraint, try to move it to a different partition where it does
match the partition constraint.  In essence, the UPDATE is split into
a DELETE from the old partition and an INSERT into the new one.  This
can lead to surprising behavior in concurrency scenarios because
EvalPlanQual rechecks won't work as they normally did; the known
problems are documented.  (There is a pending patch to improve the
situation further, but it needs more review.)

Amit Khandekar, reviewed and tested by Amit Langote, David Rowley,
Rajkumar Raghuwanshi, Dilip Kumar, Amul Sul, Thomas Munro, Álvaro
Herrera, Amit Kapila, and me.  A few final revisions by me.

Discussion: http://postgr.es/m/CAJ3gD9do9o2ccQ7j7+tSgiE1REY65XRiMb=yJO3u3QhyP8EEPQ@mail.gmail.com
```

Now, if there is some other partition in the partition tree for which the
row satisfies its partition constraint, then the row is moved to that
partition. If there isn't such a partition, an error will occur. The error
will also occur when updating a partition directly. 

Behind the scenes, the row movement is actually a `DELETE` and `INSERT` operation.

Let's see how the simple example above behave:

```sql
v11=# UPDATE t1 SET c1 = c1 + 10;
UPDATE 9

v11=# SELECT COUNT(*) FROM t1_part_1;
 count 
-------
     0
(1 row)

v11=# SELECT COUNT(*) FROM t1_part_2;
 count 
-------
     9
(1 row)
```

The rows are indeed moved from one partition to another. Nice, isn't it?
