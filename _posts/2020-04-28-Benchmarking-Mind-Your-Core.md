---
layout: post
title: Benchmarking? Mind your core.
img: ./images/blog5/mind-your-core.png
author: Krunal Bauskar
update: 28th April 2020
---

Often we observe jitter in MySQL throughput while running benchmark. Same could be true even for users but there are so many other things to look for (especially IO bottleneck) that the aspect we plan to discuss today may get overlooked. In this article we will discuss one such reason that could affect the MySQL performance.

## <span style="color:#4885ed">Scheduling threads on NUMA enabled VM/Machine</span>

NUMA is often looked upon from a memory allocation perspective but the article tries to explore how booting thread on different vCPUs can affect performance in a big way. During our experiment we have seen performance swing upto 66%.

MySQL has an option named ``innodb_numa_interleave`` that if enabled will try to uniformly allocate the buffer pool across the NUMA node. This is good but what about the worker threads. Are these worker threads too uniformly allocated across the NUMA node?  Cross NUMA access is costlier and so having a worker thread closer to the data is always prefered but given the generic nature of these worker threads shouldn’t they be uniformly distributed.

Say I am booting 12 worker threads on a 24 vCPU machine with 2 NUMA nodes then uniform distribution would expect 6 worker threads bound to vCPUs from NUMA-node-0 and remaining 6 to vCPUs from NUMA-node-1.

The OS (Linux) scheduler doesn’t work it that way. It would try to exhaust vCPUs from one of  the NUMA-nodes and then proceed to another.

All this could affect performance big-way till your worker threads (scalability) < number-of-cores. Even beyond this point one may see varying performances for the same test-case due to core-switches.

## <span style="color:#4885ed">Understanding the setup:</span>

Let’s now try to see how the throughput can change based on where the worker threads are located.

I am using the same machine to run client (sysbench) and server so few cores are reserved for clients too. We also consider the position of client threads as it is an important aspect while running benchmark (unless you plan to use some dedicated machine for it).

* 24 vCPU/48 GB VM with 2 NUMA nodes.
    * NUMA-1: 0-11 vCPU/24GB
    * NUMA-2: 12-23 vCPU/24GB

* x86 (Intel(R) Xeon(R) Gold 6266C CPU @ 3.00GHz) VM has 2 threads per physical core so 24 vCPU = 12 physical cores. So we would also explore scenarios when  both worker threads are located on different vCPU but have the same physical core.

* Test-Case: oltp-point-select with 2 threads. Purposely limiting it to 2 threads to keep other cores open to allow the OS to do core-switches (this has its own sweet effect). Also, all data is in-memory and using point-select means no IO being done so IO bottlenecks or background threads are mostly idle. All iterations are timed for 60 seconds.

* vCPUs/cores are bound to sysbench and mysqld using numactl (vs taskset) given the flexibility it provides. 

* For server configuration please check [here](https://github.com/mysqlonarm/benchmark-suites/blob/master/sysbench/conf/96tx1.5m_cpubound.cnf).<br>
Data-Size: 34G and BP: 36G. Complete data in memory and equally distributed so 50% of data on numa-0 and remaining 50% of numa-1. Sysbench uses range-type=uniform that should touch most of the varied parts of the table.

| client-threads | server-threads | tps
--- | --- | --- | --- | --- | --- |
Client-Threads bounded to vCPU: (0, 1, 12, 13)|Server Thread bounded to vCPUs: (2-11, 14-23)|**35188, 37426, 35140, 37640<br> 37625, 35574, 35709, 37680**


Naturally the tps is fluctuating. Some closer look revealed that OS continues to do core-switch that causes TPS to fluctuate (range of 7% is too high for small test-case like this). Also, OS continues to switch client threads cores too.

That prompted me to explore more about server core binding. As part of completeness I also explored client thread positioning.


### <span style="color:#1aa260">Position of Client and Server Threads</span>

|  | Server-Threads: (Numa Node: 0, Physical Core: 2-5, vCPU: 4-11) |  Server-Threads: (Numa Node: 0, Physical Core: 2, 3, vCPU: 4, 6) | Server-Threads: (Numa Node: 0, Physical Core: 2, vCPU: 4, 5)
| Client Threads | |
--- | --- | --- | --- 
(Numa Node: 0, Physical Core: 0, vCPU: 0,1)| - Client+Server threads on same NUMA<br>- Server threads may do core-switch (OS-scheduler dependent)<br>- Client threads on the same physical core <br><br> **TPS: 39570, 38656, 39633** |- Client+Server threads on same NUMA<br>- Server threads are less likely to do core-switch.<br>- Client threads on the same physical core.<br><br><br>**TPS: 39395, 39481, 39814**|- Client+Server threads on same NUMA<br>- Server threads less likely to do core-switch (on same physical core)<br>- Client threads on the same physical core.<br><br>**TPS: 39889,  40270, 40457**|
(Numa Node: 0, Physical Core: 0,1, vCPU: 0,2)| - Client+Server threads on same NUMA<br>- Server threads may do core-switch (OS-scheduler dependent)<br>- Client threads on different physical core <br><br> **TPS: 39890, 38698, 40005**|- Client+Server threads on same NUMA<br>- Server threads are less likely to do core-switch.<br>- Client threads on different physical core.<br><br><br>**TPS: 40068, 40309, 39961**|- Client+Server threads on same NUMA<br>- Server threads less likely to do core-switch (on same physical core)<br>- Client threads on different same physical core.<br><br>**TPS: 40680, 40571, 40481**|
(Numa Node: 0, Physical Core: 0, vCPU: 0)| - Client+Server threads on same NUMA<br>- Server threads may do core-switch (OS-scheduler dependent)<br>- Client threads on same physical core and same vCPU <br><br> **TPS: 37642, 39730, 35984**|- Client+Server threads on same NUMA<br>- Server threads are less likely to do core-switch.<br>- Client threads on same physical core and same vCPU<br><br><br>**TPS: 40426, 40063, 40200**|- Client+Server threads on same NUMA<br>- Server threads less likely to do core-switch (on same physical core)<br>- Client threads on same physical core and same vCPU<br><br>**TPS: 40292, 40158, 40125**|
(Numa Node: 1, Physical Core: 6, vCPU: 12,13)| - Client+Server threads on different NUMA<br>- Server threads may do core-switch (OS-scheduler dependent)<br>-Client threads on the same physical core <br><br> **TPS: 34224, 34463, 34295** |- Client+Server threads on different NUMA<br>- Server threads are less likely to do core-switch.<br>- Client threads on the same physical core.<br><br><br>**TPS: 34518, 34418, 34436**|- Client+Server threads on different NUMA<br>- Server threads less likely to do core-switch (on same physical core)<br>- Client threads on the same physical core.<br><br>**TPS: 34282, 34512, 34583**|
(Numa Node: 1, Physical Core: 6,7, vCPU: 12,14)| - Client+Server threads on different NUMA<br>- Server threads may do core-switch (OS-scheduler dependent)<br>-Client threads on different physical core <br><br> **TPS: 34462, 34127, 34620**|- Client+Server threads on different NUMA<br>- Server threads are less likely to do core-switch.<br>- Client threads on different physical core.<br><br><br>**TPS: 34438, 34379, 34419**|- Client+Server threads on different NUMA<br>- Server threads less likely to do core-switch (on same physical core)<br>- Client threads on different same physical core.<br><br>**TPS: 34804,34453,34729**|
(Numa Node: 1, Physical Core: 6, vCPU: 12)| - Client+Server threads on different NUMA<br>- Server threads may do core-switch (OS-scheduler dependent)<br>- Client threads on same physical core and same vCPU <br><br> **TPS: 34989, 35162, 35245**|- Client+Server threads on different NUMA<br>- Server threads are less likely to do core-switch.<br>- Client threads on same physical core and same vCPU<br><br><br>**TPS: 35503, 35455, 35632**|- Client+Server threads on same NUMA<br>- Server threads less likely to do core-switch (on same physical core)<br>- Client threads on different physical core and same vCPU<br><br>**TPS: 35572, 35481, 35692**|

### <span style="color: #de5246">Observations:</span>
* Limiting cores for server threads helps stabilize the performance (reduce jitter). OS-core switches are costly (with varying scalability this may not be feasible but a good parameter to understand).
 
* Moving client thread to different NUMA affects performance in a big way (40K -> 34K). I was not expecting this since the real work is done by server worker threads so moving client threads should not affect server performance to this level (17%). 

So from the experiment we learned that client and server threads if co-located on the same NUMA and technique to reduce core-switch (till it is really needed with increased scalability) helps achieve optimal performance.

<span style="color: #de5246">But wait! Our goal is to have balance distribution of client and server threads across the NUMA node to get optimal performance.</span>

### <span style="color:#1aa260">Balance Client and Server Threads across NUMA</span>

Let’s apply the knowledge gained above to balance numa configuration

| client-threads | server-threads | tps|remark
--- | --- | --- | --- 
Client-Threads bounded to vCPU: 0, 1, 12, 13|Server Thread bounded to vCPUs: (2-11, 14-23)|35188, 37426, 35140, 37640, 37625, 35574, 35709, 37680| <span style="color:#800000">Lot of core switches</span>
Client thread bounded to specific vCPU across NUMA (0,12)|Server Thread bounded to specific vCPU across NUMA (4,16)|30001, 36160, 24403, 24354<br>37708, 24478, 36323, 24579| <span style="color:#800000">Limit core switches</span>

Oops it turned out to be worse than expected. Fluctuation increased. Let's understand what went wrong

* 24K: OS opted for skewed distribution with NUMA-x running both client threads and NUMA-y running both server threads.
* 37K: OS opted for well balance distribution with each NUMA running 1 client and 1 server thread.

<em>(All other numbers are mix of combinations)</em>

Let’s try a possible hint. NUMA balancing. You can read more about it [here](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/virtualization_tuning_and_optimization_guide/sect-virtualization_tuning_optimization_guide-numa-auto_numa_balancing)

``
echo 0 > /proc/sys/kernel/numa_balancing
``


| client-threads | server-threads | tps|remark
--- | --- | --- | --- 
Client thread bounded to specific vCPUs across NUMA (0,12)|Server Thread bounded to specific vCPUs across NUMA (4,16) |33628, 34190, 35380, 37572| <span style="color:#800000">Limit core switches + NUMA balancing disabled. <br>Jitter is still there but surely better than 24K case above.</span>

* What if we bind client thread to specific numa cores and balance server threads

| client-threads | server-threads | tps|remark
--- | --- | --- | --- 
Client thread bounded to specific vCPU NUMA (0)|Server Thread bounded to specific vCPUs across NUMA (4,16) |36742, 36326, 36701, 36570| <span style="color:#800000">Limiting core switches + NUMA balancing disabled.<br>Looks well balanced now.</span>
Client thread bounded to specific vCPU NUMA (12)|Server Thread bounded to specific vCPUs across NUMA (4,16) |35440, 35667, 35748, 35578| <span style="color:#800000">Limit core switches + NUMA balancing disabled. <br>Looks well balanced now.</span>


## <span style="color:#4885ed">Conclusion</span>

Through multiple experiments above we saw how the given test-case can help produce different results ranging from 24K -> 40K based on where and how you run client and server threads.

If your benchmark really cares about the lower scalability then you should watch out for the core allocation.

Usual strategies to reduce noise are average of N runs, median of N runs, best of N runs, etc... But if the variance is that high none of them will work best. I tend to use strategy of averaging N smaller time runs so with probabilty things could stablize. Not sure if this is best approach but seems like it help reduce the noise to quite some level. Lesser sample (smaller value of N) would increase noise so I would recommend N = 9 at-least with each run of (60+10 (tc-warmup)) secs so 630 seconds run of test-case is good enough to reduce the jitter.

If you have better alternative, please help share it with community.

BTW, story is more complex with increasing NUMA nodes and more cores on ARM. Topic of future. If anyone has studied it, would love to understand it.

<br>
<em>If you have more questions/queries do let me know. Will try to answer them.</em>
