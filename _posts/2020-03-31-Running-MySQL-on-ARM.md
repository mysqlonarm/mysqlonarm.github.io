---
layout: post
title: Running MySQL on ARM. Does it work?
img: ./images/running-mysql-on-arm.png
author: Krunal Bauskar
update: 31st March 2020
---

I am sure most of you may have this question. In fact, I too had it before I started working on #mysqlonarm initiative. What does it take to run MySQL on ARM? Does it really work? What about dependencies? What kind of performance does it have? What about support? Is there enough community support? This could go on.....

Let’s try to answer these questions in simple question answer format.

#### Q: Is MySQL supported on ARM?
A: Yes, MySQL is officially supported on ARM. There are packages available that you can download from mysql.com site.

#### Q: Which OS are supported?
A: Currently support is enabled for RHEL-7 & 8/Oracle-Linux- 7 & 8. I don’t see direct package support for other OS.

#### Q: Can we build it from source code for other OS (like say Ubuntu)?
A: Yes. It works. I have been using binaries built from source code (using mysql-8.0.19 tag current release tag) on Ubuntu-18.04 (Bionic Beaver). (Also, build it on CentOS if you want to go the source code way). This also means all needed dependencies are taken care off or are already available.

#### Q: Are supporting tools available on ARM?
A: Since packages are available and I was able to build it from source too the default utilities like mysql shell/mysqladmin/mysqlslap/mysqldump/etc... and tons of other things that default ships along with binaries are available. If you care about a specific tools do let me know I will check them out. For now I have tried percona-toolkit some selective tools and they too work.

#### Q: Does MariaDB and Percona too support their respective server flavor on ARM?
A: MariaDB Community Server packages (from MariaDB corporation) are available for ARM (CentOS7/Ubuntu-16.04/18.04). Tools for MariaDB server are not yet officially available on ARM.
Percona doesn’t yet officially support ARM but I was able to build it from source (MyRocks/TokuDB are not available).

#### Q: Non-availability of tools. Can that block my progress of trying MySQL (or its variants) on ARM?
A: No. Since most of these tools talk mysql protocol you can of-course install them on x86 with server running on ARM. (if tool is not yet ported to ARM)

#### Q: Is there enough community support?
A: MySQL on ARM is there for quite some time. There are active contributions from multiple vendors including ARM, Qualcomm, Huawei etc… and the community is growing rapidly. There is a lot of interest from all sections on optimizing MySQL on ARM. Lot of developers wanted to connect with this initiative. There are few challenges, most importantly non-availability of the hardware. If you are interested in contributing please talk to me (shoot me an email).

#### Q: All that looks good. What about performance?
A: This is a wide topic so I will be publishing multiple posts on this topic in the coming days but to put it in short performance is comparable. On other hand ARM instances should provide better price performance.

#### Q: What about Support?
Since packages are available officially from MySQL I presume their service offering should also cover ARM. Same with MariaDB. And of-course beyond official support there are common groups and independent developers.


#### Command to build MySQL on ARM
```
cmake .. -DWITH_NUMA=1 -DDOWNLOAD_BOOST=1 -DWITH_BOOST=<boost-dir> -DCMAKE_INSTALL_PREFIX=<dir-to-install>
make -j <num-of-cores>
```
So no special flag is needed to build MySQL on ARM. (Assumes you have installed standard dependencies). It defaults compiles with "CMAKE_BUILD_TYPE=RelWithDebInfo"


## <span style="color:#4885ed">Conclusion</span>

MySQL on ARM is reality and it is now officially supported with ever growing eco-system/community. So give it a try. It could be your next cost-saving options without comprising performance or functionality.

<em>If you have more questions/queries do let me know. Will try to answer them</em>
