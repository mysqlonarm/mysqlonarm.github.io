---
layout: post
title: Why run MariaDB on ARM?
img: ./images/blog23/mariadbonarm.jpg
author: Krunal Bauskar
update: 1st June 2021
---

MariaDB has been releasing packages for the MariaDB Server on ARM for quite some time now. Infact, it was first in mysql space to get ported and optimize the server for ARM. It continues to evaluate its new performance features/releases/regression by testing them on ARM through community support.

Let’s explore state of MariaDB server on ARM using 4 main aspects:
 * Feature Set
  * Performance
  * Ecosystem
  * Community Support

<hr>
## <span style="color:#4885ed">Feature Set</span>

MariaDB Server is an ecosystem in itself. It has a lot of plugins and a support for different storage engines. Most of the mainline plugins and InnoDB Storage Engine (default) are supported on ARM. Some of the other optional storage engines viz. S3, Connect (and of-course old MySQL default like MyISAM, CSV, etc…) too works on ARM. Besides this, functionalities like replication, clustering too are supported on ARM.

In short, MariaDB Server on ARM is feature complete from production perspective with support for most of the mainline components.

Official packages for MariaDB Server on ARM are released directly by MariaDB for wide range of distros (centos/ubuntu/rhel/etc...).
<hr>

## <span style="color:#4885ed">Performance</span>

MariaDB has been actively working on improving performance of MariaDB Server on ARM. In addition to that efforts, majority of the community contributed patches are already accepted. Also, the generic new features/improvements (not limited to specific platform) are regularly evaluated on ARM and tuned for ARM before the final acceptance. All this helps scale MariaDB on ARM better than its other counterparts. More on this below when we talk about the performance number.

<hr>

## <span style="color:#4885ed">Ecosystem</span>

Besides the mainline server there are a lot of other supporting components needed for the complete db-system to work. This includes tools from different categories. Let’s look at the categories and open source tools available on ARM that are compatible with MariaDB.
 * Backup: mariabackup (inherent part of the server)
 * High Availability: binlog-based replication, multi-master cluster (through galera replication) (inherent to the server)
 * Monitoring: Percona Management and Monitoring (works on ARM. community evaluated)
 * Load-Balancing: ProxySQL (official packages on ARM)
 * Connectors: ODBC (build on arm), JDBC, Python (respective libraries/interpreter as supported on ARM).
 * Tools: Percona Toolkit (works on ARM), other inherent tools and 3rd party tools.
 
 <img src="/images/blog23/img1.png" height="400" class="centerimg"/>
<br>

Most of the major components (importantly all opensource) are available on ARM that help mark the ecosystem complete.
<hr>

## <span style="color:#4885ed">Community Support</span>

Database on ARM in general is gaining a lot of traction and the db community on arm is growing. The MariaDB community on ARM also enjoy a good support from different community members contributing multiple patches (including performance), help in maintaining the releases, arm bug fixes, testing features, identifying issues, etc… All this also helps create a good knowledge base through blogs/articles, etc… In addition there is a dedicated topic/channel on mariadb community zulip chat about [#mariadbonarm](https://mariadb.zulipchat.com/#narrow/stream/118759-general/topic/mariadbonarm).
<hr>

## <span style="color:#4885ed">Benchmarking</span>

Let’s now look at the main motivation behind considering ARM that is cost-saving. ARM instances are cheaper and in turn are known to provide better tps per dollar spent (in other words cost saving) or even on the same compute power front ARM scales better.

We have evaluated MariaDB Server 10.6.1 using CPM/CCM. You can read more about cpm [here]((https://mysqlonarm.github.io/CPM/)).
  * Workload Configuration:
     * sysbench 100 tables * 3 millions (roughly 69 GB of data) (Buffer Pool: 80GB so CPU bound)
   * REDO-log: 20 GB
   * Server version: MDB-10.6.1 (tagged beta)
   * Test-Case scenarios:
       * sysbench: point-select, read-only, read-write, update-index, update-non-index so all aspects are covered well.
   * Access Pattern (please check sysbench site for exact details).
       * Uniform: all data is touched with equal probability there-by generating more IO but lesser contention.
       * Zipfian: part of the data is touched there-by generating lesser IO but more contention.
   * Machine Configuration ([#cpm](https://mysqlonarm.github.io/CPM/))
      * x86_64: Intel(R) Xeon(R) Gold 6151 CPU @ 3.00GHz (HT enabled) [28 ht-cores: 22 server + 6 client], 192GB mem
      * ARM: Kunpeng 920 (2.6 Ghz) [64 cores: 56 server + 8 client], 192GB mem
   * Storage/OS:
       * 1.6TB NVMe SSD (random read/write 180K/70K), CentOS-7

[Detailed configuration](https://github.com/mysqlonarm/benchmark-suites/blob/master/mysql-sbench/conf/mdb.cnf/100tx3m_106_cpubound.cnf) <br>
<em>**For the same compute power (compute constant model #ccm) we have allocated 64 vCPU to both x86 and ARM.**</em>

### <span style="color:#DB4437"><ins>Uniform - CPM (Same Cost, Different Compute Power)</ins></span>

Given the different architectures and each one has a unique way to scale (more cores, faster cores, turbo-boost, etc...), keeping cost constant and allowing compute power to differ (keeping all other resources same) looks to be best way to evaluate and compare.

 <img src="/images/blog23/img2.png" height="400" class="centerimg"/>
<br>

  * MariaDB on ARM continue to scale better than its counterpart.
  * **<span style="color:#0F9D58">TAKE-AWAY: Same Cost - Better Throughput</span>**

### <span style="color:#DB4437"><ins>Uniform - CCM (Different Cost, Same Compute Power)</ins></span>

Needless to mention, ARM is 50+% cheaper than its counter part.

 <img src="/images/blog23/img3.png" height="400" class="centerimg"/>
<br>

  * MariaDB on ARM, despite lower cost is able to scale on-par with its counterpart with marginally better in some cases. (depsite lesser speed 2.6 Ghz vs 3 Ghz)
  * **<span style="color:#0F9D58">TAKE-AWAY: Lesser Cost - Same/Better Throughput</span>**

### <span style="color:#DB4437"><ins>Zipfian - CPM (Same Cost, Different Compute Power)</ins></span>

 <img src="/images/blog23/img4.png" height="400" class="centerimg"/>
<br>

  * Even with increased contention MariaDB on ARM is able to scale better with increasing scalability.
  * **<span style="color:#0F9D58">TAKE-AWAY: Gain observed for all kind of workloads</span>**

<em> and more such workloads could be tried .... </em>

## <span style="color:#4885ed">Conclusion</span>

So it is quite evident from the above analyzed facts that MariaDB on ARM is the optimal choice and it is ready for adoption in the production environment. We have also tried experimenting with master-slave and multi-master configuration on ARM and all of that too has worked flawlessly providing better cost/performance. Do let us know your experience once you give it a try or if you want us to explore about any specific server feature (trying it ARM) that you regularly use in production.

<br>
<em>If you have more questions/queries do let me know. Will try to answer them.</em>
