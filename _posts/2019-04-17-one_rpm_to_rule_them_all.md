---
layout: post
title: One RPM to rule them all...
---

As of 15 April 2019, there is only one repository RPM per distro, and it 
includes repository information for all available PostgreSQL releases. 

This change, announced by Devrim on the `pgsql-pkg-yum` mailing list, has some 
impacts.

<!--MORE-->

-----

# [](#announce)Announce

The announce from Devrim may be found [here](https://www.postgresql.org/message-id/flat/6f1e601300d575195d4f0d8a066ef4abf4c90c99.camel%40gunduz.org).

  * Instead of having separate repo RPMs per PostgreSQL major version, we now
have one single repo RPM that supports all supported PostgreSQL releases. 
The new packages obsolete the current ones.

  * The repo RPM version has been bumped to 42. Hopefully that
will be the end of the "The repo RPM is 10-4, how can I find 10-7 repo rpm, so
that I can install PostgreSQL 10.7?" type questions.

  * The "latest" suffix has been added to all repo RPMs.

-----

# [](#installation)Installation

Let's see some impacts of those changes on CentOS 7.

As usual, go to `https://www.postgresql.org/download/linux/redhat/` and chose 
the version (11), the platform (CentOS 7) and the architecture (x86_64) you 
want to install.

Today, you still get the link to the [pgdg-centos11-11-2](https://download.postgresql.org/pub/repos/yum/11/redhat/rhel-7-x86_64/pgdg-centos11-11-2.noarch.rpm) rpm.

Let's install it:

```bash
# yum install https://download.postgresql.org/pub/repos/yum/11/redhat/rhel-7-x86_64/pgdg-centos11-11-2.noarch.rpm
Loaded plugins: fastestmirror
pgdg-centos11-11-2.noarch.rpm
Examining /var/tmp/yum-root-5eSWGp/pgdg-centos11-11-2.noarch.rpm: pgdg-redhat-repo-42.0-4.noarch
Marking /var/tmp/yum-root-5eSWGp/pgdg-centos11-11-2.noarch.rpm to be installed

Resolving Dependencies
--> Running transaction check
---> Package pgdg-redhat-repo.noarch 0:42.0-4 will be installed
--> Finished Dependency Resolution

Dependencies Resolved
========================================================================================================
 Package                   Arch            Version            Repository                           Size
========================================================================================================
Installing:
 pgdg-redhat-repo          noarch          42.0-4             /pgdg-centos11-11-2.noarch          6.8 k

Transaction Summary
========================================================================================================
Install  1 Package

Total size: 6.8 k
Installed size: 6.8 k
```

In fact, the new `pgdg-redhat-repo` rpm will be installed...

The yum `.repo` file will now contains the urls of all supported PostgreSQL 
releases:

```bash
# cat /etc/yum.repos.d/pgdg-redhat-all.repo |grep "\["
[pgdg12]
[pgdg11]
[pgdg10]
[pgdg96]
[pgdg95]
[pgdg94]
...
```

The repositories for version 9.4 to 11 are enabled by default.

The PostgreSQL packages are then easily reachable and you might even install 
two different releases at once:

```bash
# yum install postgresql11-server postgresql10-server
...

Dependencies Resolved
========================================================================================================
 Package                        Arch              Version                       Repository         Size
========================================================================================================
Installing:
 postgresql10-server            x86_64            10.7-2PGDG.rhel7              pgdg10            4.5 M
 postgresql11-server            x86_64            11.2-2PGDG.rhel7              pgdg11            4.7 M
Installing for dependencies:
 libicu                         x86_64            50.1.2-17.el7                 base              6.9 M
 postgresql10                   x86_64            10.7-2PGDG.rhel7              pgdg10            1.6 M
 postgresql10-libs              x86_64            10.7-2PGDG.rhel7              pgdg10            355 k
 postgresql11                   x86_64            11.2-2PGDG.rhel7              pgdg11            1.6 M
 postgresql11-libs              x86_64            11.2-2PGDG.rhel7              pgdg11            360 k

Transaction Summary
========================================================================================================
Install  2 Packages (+5 Dependent packages)

Total download size: 20 M
Installed size: 80 M
```

To simplify blog posts or automated scripts, you might easily use the simlink 
to the actual latest repo RPM:

```
https://download.postgresql.org/pub/repos/yum/11/redhat/rhel-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

-----

# [](#updates)Updates

This change also has an impact on existing installations. 

On, for example, an existing v11 installation:

```bash
# cat /etc/yum.repos.d/pgdg-11-centos.repo |grep "\["
[pgdg11]
...
```

The update operation will replace the old `pgdg-centos11` rpm by the new one 
and create the new `.repo` file:

```bash
# yum update pgdg-centos11
...
Resolving Dependencies
--> Running transaction check
---> Package pgdg-centos11.noarch 0:11-2 will be obsoleted
---> Package pgdg-redhat-repo.noarch 0:42.0-4 will be obsoleting
--> Finished Dependency Resolution

Dependencies Resolved
========================================================================================================
 Package                        Arch                 Version                 Repository            Size
========================================================================================================
Installing:
 pgdg-redhat-repo               noarch               42.0-4                  pgdg11               5.6 k
     replacing  pgdg-centos11.noarch 11-2

Transaction Summary
========================================================================================================
Install  1 Package

Total download size: 5.6 k
```

-----

# [](#eol)EOL'd releases

While the yum repositories for EOL'd releases still exist, the repo rpms 
usually found on `https://yum.postgresql.org/repopackages.php#pg93` aren't 
available anymore: `404 - Not Found`.

In case you need to use the old repositories, you can add it manually:

```bash
# cat /etc/yum.repos.d/pgdg-93-centos.repo 
[pgdg93]
name=PostgreSQL 9.3 $releasever - $basearch
baseurl=https://download.postgresql.org/pub/repos/yum/9.3/redhat/rhel-$releasever-$basearch
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-PGDG-93

[pgdg93-source]
name=PostgreSQL 9.3 $releasever - $basearch - Source
failovermethod=priority
baseurl=https://download.postgresql.org/pub/repos/yum/srpms/9.3/redhat/rhel-$releasever-$basearch
enabled=0
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-PGDG-93
```

Of course, since those releases are not supported anymore, you shouldn't have 
to use it anymore...

-----

# [](#conclusion)Conclusion

While this change is still new, it will make our lives easier in the future...
