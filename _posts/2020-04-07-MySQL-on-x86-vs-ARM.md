---
layout: post
title: MySQL on x86 vs ARM
img: ./images/blog3/mysql_x86_vs_ARM.png
author: Krunal Bauskar
update: 7th April 2020
---

By and large this would be a topic of interest for most of us including me when I started to explore this space. Before we dwell into the numbers let’s first understand some basic differences between 2 architectures. Beyond being CISC and RISC let’s look at the important differences from MySQL perspective.

* Strong vs Weak memory model (weak memory model needs proper memory barrier while writing lock-free code).
* Underlying hardware specific specialized instructions. For example: both now support crc32c hardware instructions but being low-level they are different ways to invoke them. For more differences checkout for x86-SSE/ARM-ACLE.
* Cache Line differences. Most of the ARM processors tend to use bigger cache lines (128 bytes for all caches or a mix of 64/128 bytes).
* Other sys-call level differences like: absence of PAUSE instructions with ARM and substitute instruction with very low latency failing to induce needed delay, sched_getcpu is costlier on ARM introducing challenges with use of lock-free construct, memory operations seems to show higher latency, etc...

Community has contributed multiple patches around this space (Topic for another blog). Since MySQL just started supporting MySQL on ARM there are  few optimizations but most of the work is yet to be done.

## <span style="color:#4885ed">Performance</span>

Now let’s look at the most important aspect: Performance

We tested the performance of MySQL (current release 8.0.19) on x86 and ARM. Details of the test and machine are given below.

### <span style="color:#1aa260">Test Setup</span>

* 24 vCPU/48 GB Intel(R) Xeon(R) Gold 6266C CPU @ 3.00GHz for running MySQL on x86.
* 24 vCPU/48 GB ARM @ 2.60GHz for running MySQL on ARM
* sysbench is running on a dedicated machine located in the same data-center.
* sysbench steps:
   * Load Tables. (Same seed db is reused for multiple runs so warmup is needed).
   * Checksum based warmup. Run checksum on all tables. For checksum, flow needs to fetch the rows in the buffer pool there-by causing it to warm up.
   * Query based warm up. Can skip but helpful if you are using adaptive hash indexes.
   * Execute TC (oltp-read-write/oltp-update-index/oltp-update-non-index/oltp-read-only/oltp-point-select)
   * Each TC is executed for N different scalability. Given 24 vCPU tried it for 1/2/4/8/16/32/128/256.
   * Before switching TC, an intermediate sleep is introduced to help flush changes from previous TC. This can’t ensure all changes are flushed but sleep of X secs ensures least impact on followup TC.
   * MySQL-Server Configuration:
       * BP is large enough to accomodate complete data in-memory
       * For more details please check the [following configuration details](https://github.com/mysqlonarm/benchmark-suites/blob/master/sysbench/conf/96tx1.5m_cpubound.cnf)

<p></p><br>
 Details of running the scripts and automated test-script to invoke sysbench are also [available here](https://github.com/mysqlonarm/benchmark-suites)


### <span style="color: #1aa260">Run specific details:</span>

- Table: 96-tables * 1.5 million (data-size= 34GB)
- Buffer Pool: 36GB
- Redo-Log: 4GB*2
- TC-run-time: 300 secs
- TC-warmup: 60 (sysbench --warmup-time)
- workload-query-based warmup: 600
- change-over-sleep: 180
- checksum-based-warmup: enabled
- data-storage: 300GB (support for 16500 IOPS (nullify effect of Burst IOPS)).

<font size="3"><em>Note: Frequency Scaling (FS). Given ARM is running @ 2.6 GHz vs x86 is running @ 3.0 GHz. Comparing them directly is not fair. In order to compensate for the frequency difference, graphs also add frequency-scaled tps/qps for ARM (ARM-fscaled simply extrapolate original ARM tps/qps number by (3/2.6) factor). In real life, the factor could be a bit on the higher side given increasing CPU frequency can affect contention graphs and wait cycles.</em></font>

<br>
<hr>
### <span style="color: #de5246"><ins>1. Point Select:</ins></span>

<img src="/images/blog3/ARM-vs-x86-ps.png" width="100%"/>

| threads | ARM (qps) | x86 (qps) | ARM (qps - fscaled (FS)) | % ARM-vs-x86 | % ARM (FS)-vs-x86
--- | --- | --- | --- | --- | --- |
1|6696|6439|7726|4|20   
2|12482|11774|14402|6|22
4|23881|21308|27555|12|29
8|45993|42110|53069|9|26
16|88517|81239|102135|9|26
32|142974|136724|164970|5|21
64|198839|212484|229430|-6|8
128|217778|241555|251282|-10|4  
256|209797|224009|242073|-6|8

#### Analysis:
* ARM performs better than x86 for lower scalability but fails to scale at same rate with increasing scalability.
* With frequency scaling applied ARM continues to beat x86 despite of the scalability issues.

<br>
<hr>
### <span style="color: #de5246"><ins>2. Read Only:</ins></span>

<img src="/images/blog3/ARM-vs-x86-ro.png" width="100%"/>

| threads | ARM (qps) | x86 (qps) | ARM (qps - fscaled (FS)) | % ARM-vs-x86 | % ARM (FS)-vs-x86
--- | --- | --- | --- | --- | --- |
1|5222|5259|6025|-1|15  
2|10333|10200|11923|1|17
4|19176|19349|22126|-1|14
8|36881|37035|42555|0|15
16|70337|67065|81158|5|21
32|109207|113210|126008|-4|11
64|139294|164148|160724|-15|-2  
128|151382|175872|174672|-14|-1 
256|149136|164382|172080|-9|5

#### Analysis:
* ARM is almost on par with x86 for lower scalability but again fails to scale for higher scalability.
* With frequency scaling applied ARM continues to beat x86 (in most cases).

<br>
<hr>
### <span style="color: #de5246"><ins>3. Read Write:</ins></span>

<img src="/images/blog3/ARM-vs-x86-rw.png" width="100%"/>

| threads | ARM (tps) | x86 (tps) | ARM (tps - fscaled (FS)) | % ARM-vs-x86 | % ARM (FS)-vs-x86
--- | --- | --- | --- | --- | --- |
1|137|149|158|-8|6
2|251|273|290|-8|6
4|462|502|533|-8|6
8|852|920|983|-7|7
16|1539|1678|1776|-8|6  
32|2556|2906|2949|-12|1 
64|3770|5158|4350|-27|-16
128|5015|8131|5787|-38|-29
256|5676|8562|6549|-34|-24


#### Analysis:
* Pattern is different with read-write workload. ARM starts lagging. Frequency scaling helps ease this lag for lower scalability but increasing scalability continues to increase the gap.

<br>
<hr>
### <span style="color: #de5246"><ins>4. Update Index:</ins></span>

<img src="/images/blog3/ARM-vs-x86-ui.png" width="100%"/>

| threads | ARM (tps) | x86 (tps) | ARM (tps - fscaled (FS)) | % ARM-vs-x86 | % ARM (FS)-vs-x86
--- | --- | --- | --- | --- | --- |
1|328|373|378|-12|1     
2|623|768|719|-19|-6    
4|1060|1148|1223|-8|7   
8|1905|2028|2198|-6|8   
16|3284|3590|3789|-9|6  
32|5543|6275|6396|-12|2 
64|9138|10381|10544|-12|2
128|13879|16868|16014|-18|-5
256|19954|25459|23024|-22|-10 

#### Analysis:
* Frequency scaled ARM continues to perform on par/better with x86 (except for heavy contention use-cases).

<br>
<hr>
### <span style="color: #de5246"><ins>5. Update Non-Index:</ins></span>

<img src="/images/blog3/ARM-vs-x86-uni.png" width="100%"/>

| threads | ARM (tps) | x86 (tps) | ARM (tps - fscaled (FS)) | % ARM-vs-x86 | % ARM (FS)-vs-x86
--- | --- | --- | --- | --- | --- |
1|328|373|378|-12|1     
2|588|686|678|-14|-1    
4|1075|1118|1240|-4|11  
8|1941|2043|2240|-5|10  
16|3367|3662|3885|-8|6  
32|5681|6438|6555|-12|2 
64|9328|10631|10763|-12|1
128|14158|17245|16336|-18|-5
256|20377|26367|23512|-23|-11

#### Analysis:
* Frequency scaled ARM continues to perform on par/better with x86 (except for heavy contention use-cases).

<br>
<hr>
## <span style="color:#4885ed">Conclusion</span>

There are some important observations we can make:

* For read only workload MySQL on ARM continues to perform on-par with MySQL on x86. 
* For write involving workload MySQL on ARM starts lagging a bit but if we consider frequency scaling things start getting  better.
* Frequency scaling is not a real life parameter so we should consider the price-per-performance ratio. This could be a topic in itself but just a quick fact: ARM instance is 66% cheaper than x86 (24U48G same one we used).
* There is a pattern that we can observe. ARM workloads are very well scalable till it hits the CPU limits. With increasing scalability, contention increases and ARM starts lagging. This is expected since mutexes/contention hot-spots were all tuned for x86 (for example: spin-lock). But now that MySQL officially supports ARM and the growing ARM community and interest, it would be tuned for ARM too.

To summarize, MySQL on ARM is a worth exploring option from a cost and performance perspective.

<br>
<em>If you have more questions/queries do let me know. Will try to answer them.</em>
