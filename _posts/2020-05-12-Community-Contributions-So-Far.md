---
layout: post
title: ARM Community Contribution (to MySQL) so far...
img: ./images/blog6/community-contributions.jpg
author: Krunal Bauskar
update: 12th May 2020
---

ARM community that has developers from varied organizations has contributed some really good patches to MySQL. Most of them are awaiting acceptance. Blog is meant to analyze these patches along with their pros and cons. Hopefully this would help ease MySQL/Oracle to accept these long-awaited patches.

## <span style="color:#4885ed">Community Patches</span>

### <span style="color:#1aa260">1. Optimizing checksum</span>

* MySQL uses 2 types of checksum: crc32c and crc32. They both are different since both uses different polynomials.
  * crc32c is used in MySQL by InnoDB to calculate page-checksum.
  * crc32 is used in MySQL for table checksum, binlog-checksum, etc…
  
#### <span style="color:#db3236">crc32c</span>
* Page checksum is calculated during each page read/write so crc32c can quickly show up as one of the top functions in perf report. Ensuring use of optimized versions of it could help improve the overall throughput of the system.
* crc32c has been implemented as a hardware function on both x86 (SSE) and ARM (ACLE). InnoDB currently uses hardware based implementation for x86 but not yet for ARM (ACLE). Patch helps address the said issue.
* Latest patch (bug#85819) also helps further optimize it using crypto (PMULL) processing instruction.

Open Contributions: <br>
[bug#79144](https://bugs.mysql.com/bug.php?id=79144 (suggest use of crc32c)) No hardware CRC32 implementation for AArch64 <br>
[bug#85819](https://bugs.mysql.com/bug.php?id=85819) Optimize AARCH64 CRC32c implementation <br>

```
[example test-case runs update-non-index and gets the crc32 as top mysqld function].

perf analysis (w/o patch)
+   10.43%          8027  mysqld   [kernel.kallsyms]    [k] _raw_spin_unlock_irqrestore                                                                 
+    3.23%          2486  mysqld   mysqld               [.] ut_crc32_sw                                                                                 
+    2.33%          1797  mysqld   [kernel.kallsyms]    [k] finish_task_switch                                                                          
+    1.73%          1330  mysqld   libc-2.27.so         [.] malloc                                                                                    

perf analysis (w/ patch)
  Overhead       Samples  Command  Shared Object        Symbol                                                                                          
+   10.60%          8133  mysqld   [kernel.kallsyms]    [k] _raw_spin_unlock_irqrestore                                                                 
+    2.37%          1816  mysqld   [kernel.kallsyms]    [k] finish_task_switch                                                                          
+    1.78%          1366  mysqld   libc-2.27.so         [.] malloc                                                                                      
....
     0.44%           338  mysqld   mysqld               [.] ut_crc32_aarch64      
```
<br>
**Conclusion:** Clearly a saving of around 3% in overall throughput can be seen. Also, as part of wider testing we see crc32c helps in overall throughput gain for all kind of test-cases.

#### <span style="color:#db3236">crc32</span>
For calculating table checksum MySQL uses zlib-based crc32. As per my knowledge, x86 doesn’t support hardware optimized versions for crc32 calculation but fortunately ARM (ACLE) supports it. The same code/flow path is also used for binlog-checksum.

Open Contributions:<br>
[bug#99118](https://bugs.mysql.com/bug.php?id=99118) ARM CRC32 intrinsic call to accelerate table-checksum (not crc32c but crc32)

```
[example test-case runs checksum on all tables and update-non-index].

perf analysis (w/o patch)

checksum:
+   49.46%         13480  mysqld   mysqld               [.] crc32_z

update-non-index:
     0.40%           311  mysqld   mysqld               [.] crc32_z                                                                                     


perf analysis (w/ patch)

checksum:
+    8.15%           988  mysqld   mysqld               [.] aarch64_crc32_checksum                                                                      

update-non-index:
     0.07%            56  mysqld   mysqld               [.] aarch64_crc32_checksum
```
<br>
**Conclusion:** This patch helps on both front. Super-accelerate table checksum (average improvement of 50%) and also marginally helps in binlog-checksum.

### <span style="color:#1aa260">2. my_convert (in turn copy_and_convert) routine is suboptimal for ARM:</span>
* my_convert is used as part of the send result for converting among charsets.
* Given the amount of the data that is converted this function gets spotted in perf top-list.
* Existing implementation uses 4 bytes copying for x86 but falls back to byte copy for ARM. This could be overall improved by using 8 bytes copying for x86-64 and aarch64 and then falling back for trailing things to existing logic. Patch for this simple operation help save significant cycles and help improve performance.

Open Contributions: <br>
[bug#98737](https://bugs.mysql.com/bug.php?id=98737) my_convert routine is suboptimal in case of non-x86 platforms

```
[example test-case runs oltp-read-write on all tables].
perf analysis (w/o patch)
+    0.79%          1114  mysqld   mysqld               [.] my_convert

perf analysis (w/ patch)
     0.22%           299  mysqld   mysqld               [.] my_convert 
```
**Conclusion:** Patch can help improve overall throuhgput.

### <span style="color:#1aa260">3. Improving memory barrier for atomic variables:</span>

* MySQL/InnoDB has a lot of variables for which it uses gcc inbuilt atomic functions (__sync_add_and_fetch or __atomic_add_fetch).
* While this is all good x86 being a strong memory model most of these counter functions were implemented to use sequential memory ordering (default).
* ARM has a relaxed memory model so using sequential memory ordering (default one) is not recommended.
* Multiple patches were submitted to help revamp the said snippets. Patches help achieve 2 things:
  * Switch to use C++11 atomics. (Now that MySQL supports it).
  * Switch to use relaxed memory order (vs sequential).

Open Contributions:<br>
[bug#97228](https://bugs.mysql.com/bug.php?id=97228) rwlock: refine lock->lock_word with C11 atomics <br>
[bug#97230](https://bugs.mysql.com/bug.php?id=97230)  rwlock: refine lock->waiters with C++11 atomics <br>
[bug#97703](https://bugs.mysql.com/bug.php?id=97703) innobase/dict: refine dict_temp_file_num with c++11 atomics <br>
[bug#97704](https://bugs.mysql.com/bug.php?id=97704) innobase/srv: refine srv0conc with c++11 atomics <br>
[bug#97765](https://bugs.mysql.com/bug.php?id=97765) innobase/os: refine os_total_large_mem_allocated with c++11 atomics <br>
[bug#97766](https://bugs.mysql.com/bug.php?id=97766) innobase/os_once: optimize os_once with c++11 atomics <br>
[bug#97767](https://bugs.mysql.com/bug.php?id=97767) innobase/dict: refine zip_pad_info->pad with c++11 atomics <br> 
[bug#99432](https://bugs.mysql.com/bug.php?id=99432) Improving memory barrier during rseg allocation	<br>

**Conclusion:** Impact is wide spread so difficult to judge using perf. Also, some of the fixes help improve semantics and may not be for performance reason as such.

### <span style="color:#1aa260">4. Restore core affinity scheduler for global counter:</span>

ARM is known for its large number of cores (and numa sockets) and to harvest the max throughput from multi-cores it is important to ensure that global counters are programmed accordingly. Having a distributed counter and incrementing part of the counter closer to the thread core should avoid cross-numa latency.

MySQL use to call sched_getcpu for getting the counter slots but this logic was changed as part of the different bug fix (that surely made sense for the said issue) but it also affected the normal global counters. Patch proposes to correct this and use sched_getcpu (core affinity) based counter for global counters.

<em>On ARM this patch unfortunately is running into overhead resulting from use of sched_getcpu which is optimized on x86 using VDSO.</em>

Open Contributions:<br>
[bug#79455](https://bugs.mysql.com/bug.php?id=79455) Restore get_sched_indexer_t in 5.7

### <span style="color:#1aa260">5. Scalability issue on ARM platform with the current UT_RELAX_CPU () code:</span>
InnoDB uses home-grown spin-wait implementation for rw-locks and mutexes.
Whenever there is a need to sleep (or let me correctly say PAUSE) then on x86 MySQL uses supported PAUSE instruction.
ARM doesn’t have support for PAUSE instruction so the flow uses a compiler barrier but this statement fails to introduce the needed delay.
Patch suggest use of  Compare-And-Exchange that should help introduce comparable delay (like PAUSE).

Open Contributions: <br>
[bug#87933](https://bugs.mysql.com/bug.php?id=87933) Scalibility issue on Arm platform with the current UT_RELAX_CPU () code.

Based on internal evaluation we couldn’t get the patch to help improve on throughput so have not-considered it as part of our community-patch branch for now.

### <span style="color:#1aa260">6. Using wider cacheline padding for ARM:</span>

Most of the ARM processors are scheduled to have a wider cache line. Patch proposes use of a wider cache line padding for ARM based processors to avoid false sharing.

Open Contributions: <br>
[bug#98499](https://bugs.mysql.com/bug.php?id=98499) Improvement about CPU cache line size	

### <span style="color:#1aa260">7. Other open contributions</span>

Besides the 6 main categories listed above there are more contributions in other areas too. But most of them didn’t have a patch associated or the said idea has been folded in MySQL as part of another major revamp (not specific to ARM work) or the idea is less likely to have a performance impact. So for now we were not able to consider these set of patches.

## <span style="color:#4885ed">Performance impact of Community Patches</span>

Based on the inputs collected above we have analyzed performance impact of community patches and below table help shows how the throughput would improve if the said patches are accepted. Limiting results for higher scalability (256 threads) where it shows major effect but we have run test-case across the board and patches helps improve overall throughput (even for single threaded).


|| point select | read only| read write | update index | update non index |
|without-patch| 218447 | 145755 |	 5646 | 22200	| 22601 |
|with-patch| 224355 | 149718 | 5829 |	23070 |	23292 |
|%| 2.7| 2.72 | 3.24	| 3.92 | 3.06 |

<em>Evaluated using mysql-8.0.20. For configuration check [here](https://github.com/mysqlonarm/benchmark-suites/blob/master/sysbench/conf/96tx1.5m_cpubound.cnf). Processor: ARM Kunpeng 920 24vCPU/48GB</em>

## <span style="color:#4885ed">Conclusion</span>

Patches contributed by community surely helps in optimizing MySQL on ARM but the impact is still limited and lot of ground to cover to make MySQL accelerate on ARM. If you have good ideas on how things could be pushed further then let's connect. ARM MySQL community can help brainstorm the idea and aid/help in materializing it to a contribution.


<br>
<em>If you have more questions/queries do let me know. Will try to answer them.</em>
