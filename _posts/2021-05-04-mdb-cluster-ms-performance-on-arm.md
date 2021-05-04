---
layout: post
title: MariaDB Cluster (Master-Slave) Peformance on ARM
img: ./images/blog20/mdb-ms-perf.png
author: Krunal Bauskar
update: 4th May 2021
---

Majority of users use databases in cluster form (either master-slave or multi-master). I often get a question that if I have tried benchmarking MariaDB server in Master-Slave Setup on ARM and if yes then what is slave lag like. So this time I decided to study the same using the basic experiment focusing on slave lag.

## <span style="color:#4885ed">Setup</span>

Let's first understand setup

  * Typical master-slave setup with a master and 2 replicating slaves.
     * Master handles only write-workload (sysbench update index).
     * Slaves handle read-workload (sysbench read only).
  * There are 2 independent experiments
     * Experiment-1: write-workload on master and read-workload on slave with slave actively replicating.
     * Experiment-2: only write workload on master and slave replication (no read-workload)
  * Server:
     * MariaDB-10.6 (10.6 post alpha - #9db14e9).
  * Master/Slave ([Cost-Performance Model](https://mysqlonarm.github.io/CPM/)).
     * 16 vCPU ARM (Kunpeng 920 2.6 Ghz)
     * 10 vCPU x86 (Intel(R) Xeon(R) Gold 6151 CPU @ 3.00GHz)
  * Storage/OS:
     * NVME SSD with the same IOPS, CentOS-7.
  * Configuration details:
     * [detailed configuration](https://github.com/mysqlonarm/benchmark-suites/tree/master/mysql-cluster-bench/cluster-conf/mdb-cluster-conf)
     * Same repo also has a script that I have used for running the said workload.

<img src="/images/blog20/img1.png" height="400" class="centerimg"/>
<br>

### <span style="color:#0F9D58">Benchmark and Analysis</span>

#### <span style="color:#DB4437">read-write workload</span>

<ins>**benchmarking pattern**</ins>
  * write-workload is executed on master and read-only workload on slaves (all workloads are executed in parallel so graph has overlap on timeline front).
  * increasing scalability from 1-128 threads. workload runs for 60 secs for each scalability and then 20 seconds of sleep.
  * script starts to monitor “seconds_behind_master (sbm)” immediately at the start of the workload and continue to monitor it till it drops to 0 and continue to remain so for sometime (to avoid false postive).
  
<img src="/images/blog20/img2.png" height="400" style="border:1px solid black"/>
<br>

<ins>**observations**</ins>
  * arm continues to report higher tps and qps with growing scalability.
  * slaves from both variants start lagging with increased scalability but despite the higher throughput in arm case, lag is kept in check and both cases ends almost at same time.
  * this clearly suggests that the path for processing master-slave workload is equally efficient with arm and can easily handle increasing workloads keeping the critical parameters (like slave lag) under check.

#### <span style="color:#DB4437">write-only workload</span>

<ins>**benchmarking pattern**</ins>
  * same as read-write just that there is no read-only workload only write workload active on master.
  
<img src="/images/blog20/img3.png" height="400" style="border:1px solid black"/>
<br>

<ins>**observations**</ins>
  * arm continues to report higher tps with growing scalability.
  * second_behind_master continue to grow with the growing tps and scale linearly.
  * so even with a write-only workload, arm continues to keep the critical parameter in check.

## <span style="color:#4885ed">Conclusion</span>

Based on the experiment we can easily conclude that MariaDB Server master-slave setup continues to scale on arm (like standalone) without affecting critical parameters like scecond_behind_master. Since most of the users have master-slave kind of setup these findings should be very much comforting if you are considering moving to ARM.

<br>
<em>If you have more questions/queries do let me know. Will try to answer them.</em>
