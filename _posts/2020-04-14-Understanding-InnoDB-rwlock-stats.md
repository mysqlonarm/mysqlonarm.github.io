---
layout: post
title: Understanding InnoDB rw-lock stats
img: ./images/blog4/rwlock-stats.png
author: Krunal Bauskar
update: 14th April 2020
---

InnoDB uses mutexes for exclusive access and rw-locks for the shared access of the resources. rw-locks are used to control access to the common shared resources like buffer pool pages, tablespaces, adaptive search systems, data-dictionary, informaton_schema, etc… In short, rw-locks play a very important role in the InnoDB system and so tracking and monitoring them is important too.

InnoDB provides an easy way to track them using “SHOW ENGINE INNODB STATUS”.

```
RW-shared spins 38667, rounds 54868, OS waits 16539
RW-excl spins 6353, rounds 126218, OS waits 3936
RW-sx spins 1896, rounds 43888, OS waits 966
Spin rounds per wait: 1.42 RW-shared, 19.87 RW-excl, 23.15 RW-sx
```

In this article we will try to understand how these stats are calculated and what is the significance of each of these numbers. We will also try to draw inferences using different use-cases and touch base important stats [bug](https://bugs.mysql.com/bug.php?id=99171) that makes the current state of the stats almost ineffective for tuning.

## <span style="color:#4885ed">rw-lock spin algorithm</span>

There are 3 types of rw-locks:
* Shared: offers shared access to the resource. Multiple shared locks are allowed.
* Exclusive: offers exclusive access to the resource. Shared locks wait for exclusive locks.
* Shared-Exclusive (SX): offer write access to the resource with inconsistent read. (relaxed exclusive).

We will first try to understand the flow and then discuss some tuning steps. <br>
(For sake of discussion, to start with, let’s assume <span style="background-color: #fff2e6; color: red">spins=0, rounds=0, os-waits=0</span>).

### <span style="color:#1aa260">Locking Steps</span>
+ **Step-1:** Try to obtain the needed lock
   +  If <span style="color: green">SUCCESS</span> then return immediately. <span style="background-color: #fff2e6; color: red"> (spins=0, rounds=0, os-waits=0)</span>
   + If <span style="color: red">FAILURE</span> then enter spin-loop.
<p></p> 
+ **Step-2:** Start Spin-loop. Increment spin-count. <em>(Why is spin-loop needed? If we enter wait then the OS will take away CPU from the given thread and then the thread will have to wait for its turns as per OS-scheduling. Better approach is to busy-wait using spin-loop (with condition check) so that the CPU is kept. Since most of these locks are short-duration likely chance of re-obtaining it very high).</em>
<p></p> 
+ **Step-3:** Start spinning for N rounds. (Here N is defined and controlled by [innodb_sync_spin_loops](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_sync_spin_loops)). Default is 30 rounds. 
   * **Step-3a:** Each round will invoke a PAUSE logic (see a separate section below about PAUSE logic) that will cause the CPU to go to PAUSE for X cycles.
   * **Step-3b:** Post each round a soft check is done if the said lock is available (busy-wait).
       * If it is available then spin-cycle exits. (There could still be some rounds pending. We will use this info below).
   * **Step-3c:** Again try to obtain the needed lock.
       * If <span style="color: green">SUCCESS</span> then return.<span style="background-color: #fff2e6; color: red"> (spins=1, rounds=M (M <= N), os-waits=0)</span>
       * If <span style="color: red">FAILURE</span> and there are some pending rounds (max=innodb_sync_spin_loops) then resume spinning. <em>(How come the loop was interrupted and locking failed. Note: the said lock is being looked upon by multiple threads in parallel. While multiple threads got signals about lock availability by the time said thread tried to obtain the lock, some other thread took it. So the said thread is now back re-trying)</em>.
   * **Step-3d:** Say a thread now completes its set rounds of spin-wait and even now it failed to obtain the lock. There is no point in  spinning further and wasting CPU cycles. Better give up pending CPU cycles back to OS and let OS scheduling do the needful. Also, since the said thread is now going to go to sleep it should register itself with some common infrastructure that will help signal it back to active whenever the said lock is available.
   *  **Step-3e:** This infrastructure to signal it back to active is sync-array infrastructure in InnoDB.
       * Said thread registers itself by reserving a slot in the said array.
       * Before starting the wait, give another try to see if the lock is available. (since reserving could be time consuming and lock could be available in meantime).
       * If still lock is not available then wait for sync-array infrastructure to signal back the said thread.
       * This wait is called OS-wait and entering this loop will now cause OS-waits count to increase.
   *  **Step-3f:** Say the said thread is signaled by sync-array infrastructure for the wait-event. It retries to obtain the needed lock.
      * If <span style="color: green">SUCCESS</span> then return. <span style="background-color: #fff2e6; color: red"> (spins=1, rounds=N, os-waits=1)</span>
      *  If <span style="color: red">FAILURE</span> then the whole loop restarts from spinning logic (Back to Step-3 with rounds-count re-intialize to 0). Note: spins count is not re-incremented.

So let’s now assign the meaning to these counts

* **spins:** represent how many times flow failed to get a lock in first go and had to enter spin-loop.
* **rounds:** represent how many rounds of PAUSE logic executed.
* **os-waits:** how many times spin-loop failed to grant a lock that resulted in os-waits.

It is possible that during a given spin loop for acquiring said lock flow may need more than 30 (innodb_sync_spin_loops) rounds of PAUSE logic and have to multiple time enter os-waits. This can cause os-waits > spins-count.

### <span style="color:#1aa260">PAUSE logic</span>

K = {random value from between  (0 - [innodb_spin_wait_delay](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_spin_wait_delay)) * [innodb_spin_wait_pause_multiplier](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_spin_wait_pause_multiplier)}

Invoke low-level PAUSE instruction K times.

Not all architectures provide low-level pause instruction. x86 does provide it but ARM doesn’t. Even with x86 latency of this pause instruction continues to change with different families of processors. It was around 10-15 cycles for old generation processors (pre-skylake). Went upto 140 cycles with Skylake and again came down with CascadeLake (I see 13 cycles with Intel(R) Xeon(R) Gold 6266C CPU @ 3.00GHz which belongs to CascadeLake family). <em>(I have not personally benchmarked it on all platforms (except Cascadelake) but this is based on the references.</em> This means the delay (in terms of cycle) introduced by PAUSE continues to change and so tuning PAUSE logic for each generation/type of processor is important. 2 configurable variables viz. innodb_spin_wait_delay, innodb_spin_wait_pause_multiplier exactly help achieve this.

## <span style="color:#4885ed">Interpreting stats</span>

Now that we understand the stats let’s look at the number and try to draw some inferences.

But before we get into further details let me point out a [bug](https://bugs.mysql.com/bug.php?id=99171) that makes these stats inconsistent and incorrect.

To get a fair idea we will use the version of mysql with the patch applied (as pointed in bug, stats w/o patch doesn’t help yield correct picture and so all kinds of interpretation and tuning is bound to fail).

### <span style="color:#1aa260">Use Case 1</span>

```
RW-shared spins 338969, rounds 20447615, OS waits 592941
RW-excl spins 50582, rounds 1502625, OS waits 56124
RW-sx spins 12583, rounds 360973, OS waits 10484
Spin rounds per wait: 60.32 RW-shared, 29.71 RW-excl, 28.69 RW-sx
```
Let’s analyze the shared spins case:<br>
```RW-shared spins 338969, rounds 20447615, OS waits 592941```


* 338K times flow couldn’t find the lock in first go, forcing thread to enter spin-lock.
* During each spin-cycle there were 60 rounds of PAUSE cycle executed (so the said spin-cycles were done 2 times).
* OS-waits/spins = 592/338 = 1.75 suggest that majority of the flow entered OS-wait (delay from PAUSE was not sufficient).
* It also suggests that for the majority of spin-cycles, single OS -wait was not enough so it was repeated.

**Conclusion:** Use-case represents heavy contention. Also, it sounds like PAUSE loop is unable to introduce the needed delay that is causing so many ROUNDS of PAUSE loop per spin-cycle.

<em>256 thread oltp-read-write workload on 24 vCPU ARM machine</em>

### <span style="color:#1aa260">Use Case 2</span>

```
RW-shared spins 35943, rounds 777178, OS waits 19051
RW-excl spins 4269, rounds 121796, OS waits 4164
RW-sx spins 13407, rounds 321954, OS waits 7346
Spin rounds per wait: 21.62 RW-shared, 28.53 RW-excl, 24.01 RW-sx
```

Let’s analyze the shared spins case:<br>
```RW-shared spins 35943, rounds 777178, OS waits 19051```

* Flow invokes spin-loop 35K times.
* Only 19K (that is approximately half of the spin-loop) caused OS-waits.
* Per spin-cycle average too is limited to 21.62 that suggests that for each spin-cycle on average 22 rounds of PAUSE loop was 
invoked.

**Conclusion:** Use-case represents medium contention.

<em>16 thread oltp-read-write workload on 24 vCPU ARM machine</em>

### <span style="color:#1aa260">Use Case 3</span>

Just for reference, let me put an example of very less contention. This is with 16 threads oltp-read-write workload on x86_64 based VM with 16 CPU.

```
RW-shared spins 39578, rounds 424553, OS waits 7329
RW-excl spins 5828, rounds 78225, OS waits 1341
RW-sx spins 11666, rounds 67297, OS waits 449
Spin rounds per wait: 10.73 RW-shared, 13.42 RW-excl, 5.77 RW-sx
```

* Flow invokes spin-loop 39K times.
* Only 7K (that is approximately 20% of the spin-loop) caused OS-waits.
* Per spin-cycle average too is limited to 10.

**Conclusion:** Use-case represents low contention.

<em>16 thread oltp-read-write workload on 24 vCPU x86_64 machine</em>

### <span style="color:#1aa260">Note on tuning</span>

Remember the high contention case we saw above. By tuning some things + code changes I could reduce the contention for shared-spins significantly.

```
RW-shared spins 318800, rounds 13856732, OS waits 374634
RW-excl spins 35759, rounds 656955, OS waits 22310
RW-sx spins 10750, rounds 226315, OS waits 5598
Spin rounds per wait: 43.47 RW-shared, 18.37 RW-excl, 21.05 RW-sx
```

Rounds per spins-cycles: average of 60 -> 43 <br>
OS-wait per spin-cycle: 1.75 -> 1.17

Is this good.? Not really. There are multiple factors to consider.

* Do you see improvement in TPS?
* Sometime it may be suggested to simply increase the PAUSE loop. But increasing PAUSE loop beyond some point can cause extended spin-cycle wasting precious CPU cycles especially if that causes it to land back in OS-wait. (Does this help more in HT cases vs multi-core case).
* Also, as pointed above, processor generation and type affect the PAUSE loop latency.

There are multiple factors to consider. Even I am exploring this to see how we can tune this for all kinds of CPUs. I will blog more about it once I get some good solid algorithms on this front (or maybe we can develop some automated, self-adjustable or adaptive algorithms) so users don’t need to worry about it.


## <span style="color:#4885ed">Conclusion</span>
As we saw above rw-locks stats can help us get good insight on understanding the contention of the system. Of-course it is not the only in-sight about InnoDB contention as mutexes are not covered as part of these stats. Tuning could be challenging but over-tuning can affect performance in the wrong way too.

<br>
<em>If you have more questions/queries do let me know. Will try to answer them.</em>
