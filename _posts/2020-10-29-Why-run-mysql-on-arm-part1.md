---
layout: post
title: Why run MySQL on ARM - Part 1
img: ./images/blog14/whyrunmysqlonarm.jpg
author: Krunal Bauskar
update: 29th Oct 2020
---

MySQL on ARM is gaining consistent momentum and community is excited about it. Beyond performance, users also tend to explore other aspects like feature-set, ecosystem, support, etc… Let’s explore what users would gain/lose by moving to mysql on arm.

## <span style="color:#4885ed">Evaluation aspects</span>

There are 4 main aspect user tend to consider while migrating database/database environment/database-setup
  * Feature Set
  * Performance
  * Ecosystem
  * Community Support

Let’s analyze MySQL on ARM from these perspectives

<img src="/images/blog14/img1.png" height="400" class="centerimg40"/>

### <span style="color:#0F9D58">Feature Set</span>

MySQL on ARM supports all the features that MySQL has to offer. We are not aware of any feature that doesn't work or has been marked as beta on ARM. This means you don’t lose on the feature front if you decide to run mysql on arm. Beyond the mainline feature binlog-replication, group-replication, in-build plugins, authentication/security plugins all works fine. MySQL also has been actively fixing bugs found on ARM with the same priority like other supported platform.

### <span style="color:#0F9D58">Performance</span>

MySQL's recent efforts to fix performance in general is tuned to consider the increasing number of cores there-by we see increasing usage of distributed counters/locks/etc… All these improvements are supporting ARM as ARM is all about more cores. Lately, we also saw MySQL folded few performance patches from the community that were ARM specific. There are still 30+ patches that are pending in the queue. Hopefully all of them will get attention now.

Even without these patches, MySQL continues to perform/scale consistently on  ARM so the patches are just going to make it better and better.

Of-course we will see some real performance numbers along with some analysis in part-2 of the blog.

### <span style="color:#0F9D58">Ecosystem</span>

When users think of migrating to a new db-environment they care about surrounding ecosystem components too. Fortunately, a lot of ecosystem components for MySQL are already present on ARM. Let’s look at them.

* Backup: Percona Xtrabackup ([community evaluated](https://mysqlonarm.github.io/Backup-MySQLonARM-using-PXB/))
* High Availability: Binlog replication, InnoDB Cluster (inherent to server)
* Load Balancer: MySQL Router, ProxySQL (work-in-progress)
* Tools: MySQL Shell, Percona toolkit (community evaluated)
* Monitoring: Percona Management and Monitoring (PMM) ([community evaluated](https://mysqlonarm.github.io/Using-PMM-to-track-MySQL-on-ARM-statistics))
* Connector: available from MySQL.

This helps mark the stack complete as there is at-least one tool from each category available on ARM and more tools will be added in near future.

### <span style="color:#0F9D58">Community Support</span>

One of the reason, I believe, MySQL is so successful is due to community support it enjoys. #dbonarm as a wider initiative is gaining a lot of traction and mysql is no exception. #mysqlonarm community is expanding. Lot of users/developers are getting interested in evaluating/contributing to help improve mysql on arm. There is dedicated mysql-community-slack (look for #mysqlonarm) channel. Developers from varied organizations are contributing patches, providing feedback.

## <span style="color:#4885ed">Conclusion</span>

#mysqlonarm is going to be the next big thing in mysql space. During this pandemic time all organizations have become cost conscious and if you get a solution that helps save you on cost/increase performance then that is the solution to go with.

Stay tuned for part-2 of the blog series where we would see the performance number and analyze results.

<br>
<em>If you have more questions/queries do let me know. Will try to answer them.</em>
