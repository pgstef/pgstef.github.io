---
layout: post
title: Back from PGDay Belgium
---

For the first PGDay organized in Belgium by the 
[PgBE PostgreSQL Users Group Belgium](https://www.meetup.com/fr-FR/PostgresBE/), 
[this](http://pgconf.be) event regrouped around 40 PostgreSQL fans on 17 May 
2019 in Leuven. It was for me a really nice day and I'd like to share it with 
you.

<!--MORE-->

-----

This day was all about PostgreSQL and a big opportunity to meet with other 
people interested in PostgreSQL in Belgium. The event was suitable for 
everybody, from first-time users, students, to experts and from clerks to 
decision-makers.

With not less than 10 talks, 2 parallel tracks in the afternoon, the list of 
speakers was impressive, with a lot of international speakers too. 

After having learned some good advices with Hans-Jürgen Schönig during his 
**Fixing common performance problems** talk, Ilya Kosmodemiansky told us how 
to screw it with some **PostgreSQL Worst practices**.

In the afternoon, I choose to stay in the main room. So, I could watch Thijs 
Lemmens showing a demo on how to perform 
[**Downtimeless PG upgrades using logical replication**](https://github.com/thijslemmens/pg-logical-replication-presentation). 
Based on docker and docker-compose, he created 2 PostgreSQL clusters and 
installed pgAdmin 4 and HAProxy. If you wish to try the demo by yourself, Thijs 
released everything you need on his GitHub account.

Tomas Vondra explained us next what 
[**Create statistics**](https://github.com/tvondra/create-statistics-talk) is 
and when to use it. He also gave us a quick overview of what will be new in 
PostgreSQL 12 on this topic. It was really interesting to see examples based 
on Belgian cities. 

Then... the pressure was on my shoulders to talk about 
[Streaming Replication, the basics]({{ site.url }}/talks/en/20190517_pgconfBE_Streaming-Replication.reveal.pdf). 
The idea was to summarize for beginners what WALs are, how does the _Streaming 
Replication_ works, give some advices and best practices.

Boriss Mejías re-explained afterwards, with his own way, **Replication, where 
things just work but you don't know how**. Even if we had some overlap, the 
two talks were in fact pretty much complementary.

Unfortunately, I missed:

* **Declarative Table Partitioning - What do I need it for?** - Boriss Mejías
* **Dynamically switch between datasources in code** - Marco Huygen
* **PostgreSQL Buffers** - Vik Fearing
* **Data Vault 2.0 & Pivotal Greenplum** - Joren Oris

Hans-Jürgen Schönig finally showed some of the most common, most funny and 
most interest cases he had over the years. That was a really cool way to end 
the day.

The event was hosted by the University of Leuven. The place was awesome. Cozy, 
roomy and fully geared for the speakers. I really enjoyed being there.

[A warm thank you](https://twitter.com/the_hydrobiont/status/1129416778872958982) 
to the organizers, speakers and attendees for making this event a reality!