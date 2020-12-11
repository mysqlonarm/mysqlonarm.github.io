---
layout: post
title: Why run MySQL on ARM - Part 3 (Compute Power Comparison)
img: ./images/blog16/whyrunmysqlonarm.jpg
author: Krunal Bauskar
update: 11th Dec 2020
---

In previous [post](https://mysqlonarm.github.io/Why-run-mysql-on-arm-part2/) we analyzed MySQL performance on x86 and ARM using the Cost-Performance Model ([#cpm](https://mysqlonarm.github.io/CPM/)) where-in Cost and other resources (except compute power) was kept constant allowing compute power to differ. ARM being cheaper got more compute power for the same cost. That was a user perspective story but developers were also interested to understand how MySQL scale if we provide the same computing power.

We decided to go ahead with this and benchmark MySQL on x86 and ARM except this time with the same compute power (keeping all other resources memory, storage, etc... same) and allowing cost to differ with ARM variant cheaper than x86 variant.  It would also help understand if ARM is really powerful for High-Performance Computing like databases.

## <span style="color:#4885ed">Benchmarking Setup</span>

Benchmarking is done using sysbench.

* **Server Configuration:**
   * sysbench 100 tables * 3 millions (roughly 69 GB of data)
   * We tried 2 combinations:
     * CPU Bound: buffer pool = 80 GB so complete data in memory (lesser IO).
     * IO Bound: buffer pool = 35 GB so only 50% of the data (heavy IO).
  * REDO-Log: 20 GB
  * Server version: MySQL-8.0.22 (latest GA)
* **Test-Case scenarios:**
   * sysbench: point-select, read-only, read-write, update-index, update-non-index so all aspects are covered well.
   * Access Pattern (please check sysbench site for exact details).
      * Uniform: all data is touched with equal probability there-by generating more IO but lesser contention.
      * Zipfian: part of the data is touched there-by generating lesser IO but more contention.
* **Machine Configuration:**
  * x86_64: Intel(R) Xeon(R) Gold 6151 CPU @ 3.00GHz (HT enabled) [64 vCPU (Server and Client)], 192GB mem
  * ARM: Kunpeng 920 (2.6 Ghz) [64 vCPU (Server and Client)], 192GB mem
* **Storage:**
   * 1.6TB NVMe SSD (random read/write 180K/70K)
   
You can complete configuration [here](https://github.com/mysqlonarm/benchmark-suites/blob/master/mysql-sbench/conf/mysql.cnf).
<br>

By Compute Power we meant same number of vCPU. <br>
Real compute power of x86 is still more than ARM but is not matter of concern for now. <br>
x86: 64 * 3 = 192 <br>
ARM: 64 * 2.6 = 166

Note: All configurations are same. Cost is allowed to differ. ARM is cheaper than x86 by 2-4x magnitude in other word for cost of 1 x86 machine user can get 2-4 ARM machines.

## <span style="color:#4885ed">Benchmarking Number</span>

* <span style="color:#DB4437">**Uniform - CPU Bound**</span>

<img src="/images/blog16/uniform_cpubound.png" height="400" class="centerimg"/>

* <span style="color:#DB4437">**Uniform - IO Bound**</span>

<img src="/images/blog16/uniform_iobound.png" height="400" class="centerimg"/>

* <span style="color:#DB4437">**Zipfian - CPU Bound**</span>

<img src="/images/blog16/zipfian_cpubound.png" height="400" class="centerimg"/>

* <span style="color:#DB4437">**Zipfian - IO Bound**</span>

<img src="/images/blog16/zipfian_iobound.png" height="400" class="centerimg"/>

## <span style="color:#4885ed">Analysis</span>

Result may be surprising as ARM is able to consistently beat x86 despite of having same number of vCPU (infact with lesser compute power).

ARM processors traditionally got tagged as as less powerful given their primary usage in cell phone and network equipments only but a true server class ARM processors is as fast as any other server class processors and from benchmark it seems like it is more benefical for running databases.

<span style="color:#1aa260">**BETTER PERFORMANCE. LESSER COST. GO GREEN**</span>

Let's analyze graph further to understand the result better.

* MySQL on ARM continue to out-perform MySQL on x86 with higher scalability for the majority of the scenarios.
* Infact, for update and non-update index workload, ARM continues to beat x86 even for lower scalability.
* For read-only workload ARM scales inline with x86 (with marginal lag for lower scalability) and then surpass it for higher scalability.
* For read-write workload, ARM continue to beat x86 for some scenarios but a lag is observed for some scenarios (even for higher scalability). Gap is around 12%. Perf analysis showcased multiple reasons but patches for most of these reasons are already submitted by community to MySQL. We decided to apply a couple of them (one of it is already accepted by MySQL (tuning spin-lock should be part of future release) and the other one we applied was crc32c). Gap reduced from 12% to 6%. There are more than 30+ patches pending for ARM optimization. Most of them are performance patches for addressing issues that get showed up in perf and could help further tune mysql on arm. So the gap is set to reduce and infact we are optimistic that arm for read-write too will surpass x86 for all scenarios.

From the graph and analysis it is pretty clear that MySQL on ARM scale way better than MySQL on x86 even with lesser compute power (ARM: 2.6 Ghz vs x86: 3.0 Ghz) with the same number of vCPUs. Of-course not to forget that better tps for lesser cost (that could be in magnitude of 2-4x times cheaper).

Pattern is not limited to MySQL but also observed on other databases too (topic of separate post).

## <span style="color:#4885ed">Conclusion</span>

MySQL continue to scale better with ARM and help provide much needed cost saving. Also, since MySQL is not fully optimized for ARM the results are further set to improve and more wider and signficant impact is expected.

Happy #mysqlingonarm

<br>
<em>If you have more questions/queries do let me know. Will try to answer them.</em>
