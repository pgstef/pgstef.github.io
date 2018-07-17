---
layout: post
title: Back from PGDay Amsterdam
---

[PGDay Amsterdam](https://twitter.com/PGDayAmsterdam) regrouped more than 90 PostgreSQL fans (according to Devrim) on 12 July 2018. It was for me a really nice day and I'd like to share it with you.

<!--MORE-->

-----

User feedbacks, PostgreSQL internals, tools,... the topics covered were pretty wide.

Jan Karremans first started with **Why I picked Postgres over Oracle?**

Jan shared with us some of the lessons he learned during his journey:
* weigh your needs (what do you need? what does your project need?);
* don't be afraid, you are not alone;
* PostgreSQL is the best hidden secret.

Inspired by Dimitri Fontaine and his *Mastering PostgreSQL in Application Development* book, Oleksii Kliukin presented next **Ace it with ACID: Postgres transactions for fun and profit**.

Building a sample application to find his bike in Amsterdam, Oleksii explained what ACID means. MVCC, isolation levels, transactional DDL,... complete and local, great talk!

Daniel Westermann then tried to summarize **What we already know about PostgreSQL 11**. 

A lot of new interesting improvements are coming regarding partitioning, parallelism, covering unique indexes, procedures with transaction control,... and also, my favorite:

```
$ psql
postgres=# help
You are using psql, the command-line interface to PostgreSQL.
Type:  \copyright for distribution terms
       \h for help with SQL commands
       \? for help with psql commands
       \g or terminate with semicolon to execute query
       \q to quit
postgres=# let met get out !
postgres-# exit
Use \q to quit.
postgres-# quit
Use \q to quit.
postgres-# \q
```

After a short coffee break, Stefanie Stoelting explained how to use **PostgreSQL As Data Integration Tool** by using Foreign Data Wrappers.

FDW, what is it? Which one exists? An overview with useful examples.

Then, well...

...

My turn : [**Save your data with pgBackRest**]({{ site.url }}/talks/en/20180712_pgdayAmsterdam_pgBackRest.html.gz).

Short introduction of this **A.W.E.S.O.M.E!** backup and restore system. For a more detailed example, there's actually another post on this blog which talk about it :-)

Jeroen de Graaff told us, after lunch, the **Step-by-step implementation of PostgreSQL at Rijkswaterstaat**. Very interesting user feedback in data-science where PostgreSQL helps to manage ships harbouring in the Netherlands!

Jeroen shared with us some lessons learned:
* work agile, apply what has value ASAP;
* knowledge exchange is very valuable;
* PostgreSQL is a robust and reliable data platform.

We then made some "mind gymnastic" with Hans-Jürgen Schönig and **PostgreSQL: Timeseries analysis**.

How to:
* unleash the power of Timeseries with window functions;
* find trends with string encode;
* find continuous activity;
* use the BRIN index for heavily correlated data.

Hans-Jürgen said something very important: "Dropping data is destruction of value". I can't agree more with him and that's why you also need a good backup and restore system!

After that, Alistair Parry told us his **Adventure(work)s with PostgreSQL in the Cloud** : how he went from his "hand-crafted PostgreSQL setup" to the Amazon cloud. 

Our host, Devrim Gündüz, explained afterwards **WAL: Everything You Want to Know**. Long story short, it's designed to prevent data-loss in most of the situation.

Useful tip: **DO NOT delete WAL files manually**. The pg_xlog directory was renamed to pg_wal because people delete files under "log" directories...

After the last coffee break, Ilya Kosmodemiansky showed the **Latest evolution of Linux IO stack, explained for database people**. Ilya clarified how PostgreSQL interacts with disks and why we can't rely on *fsync* forever. Another implementation would be to use *DirectIO* but it's very OS specific.

Boriss Mejias then talked about **Internet of Things with PostgreSQL - Performance & Security**. Today, more "things" are connected than people.

Getting data from those things can be done with PostgreSQL using JSON. Declarative partitioning can also be used. One level for each device and one level over time. The BRIN index would also be very useful.

Andreas Scherbaum then gave us a **Tour de Data Types: VARCHAR2 or CHAR(255)?**. There's around 82 datatypes, with 41 for general purpose. Little tip: use boolean for partial indexes.

Finally, Bruce Momjian tried to answer to the **Will Postgres Live Forever** question. I'll stick with this quote:

> Ideas don’t die, as long as they are shared.

> Ideas are shared, as long as they are useful.

> Postgres will live, as long as it is useful.

And Voila! A long day full of very interesting talks. 

Once again, thanks to the organizers, speakers and people there for making this event a reality!