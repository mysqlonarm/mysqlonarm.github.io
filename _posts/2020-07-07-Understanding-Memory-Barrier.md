---
layout: post
title: Understanding Memory-Barrier with MySQL EventMutex 
img: ./images/blog9/memory-barrier.png
author: Krunal Bauskar
update: 7th July 2020
---

MySQL has multiple mutex implementations viz. wrapper over pthread, futex based, Spin-Lock based (EventMutex).  All of them have their own pros and cons but since long MySQL defaulted to EventMutex as it has been found to be optimal for MySQL use-cases.

EventMutex was switched to use C++ atomic (with MySQL adding support for C++11). Given that MySQL now also support ARM, ensuring a correct use of memory barrier is key to keep the EventMutex Optimal moving forward too.

In this article we will use an example of EventMutex and understand the memory barrier and also see what is missing, what could be optimized, etc…


## <span style="color:#4885ed">Understanding acquire and release memory order</span>

ARM/PowerPC follows weak memory model that means operations can be re-ordered more freely so ensuring the correct barrier with synchronization logic is important. Easiest alternative is to rely on a default one that uses sequential consistency (as done with x86) but it could affect performance big time on other architectures.

Often a programmer has to deal with 2 barriers (aka order): acquire and release.

* acquire memory order means any operation after this memory-order/barrier can’t be scheduled/re-ordered before it (but operations before it can be scheduled/re-ordered after it)
* release memory order means any operations before this memory-order/barrier can’t be scheduled/re-ordered after it (but operations after it can be scheduled/re-ordered before it).

<img src="/images/blog9/img1.png" width="80%" class="centerimg"/>
 

## <span style="color:#4885ed">Understanding EventMutex structure</span>

EventMutex provides a normal mutex-like interface meant to help synchronize access to the critical sections.

* **Enter (lock mutex)**
   * try_lock (try to get the lock if procured return immediately).
       * Uses an atomic variable (m_lock_word) that is set using compare-and-exchange (CAX) interface.
    * If fail to procure
       * Enter a spin-loop that does multiple attempts to pause followed by check if the lock is again available.
       * If after “N” attempts (controlled by innodb_sync_spin_loops) lock is not available then yield (releasing the cpu control) and enter wait by registering thread in InnoDB home-grown sync array implementation. Also, set a waiter flag after reserving the slot (this ensures we will get a slot in sync-array). Waiter flag is another atomic that is used to coordinate the signal mechanism.
* **Exit (unlock mutex)**
    * Toggling the atomic variable (m_lock_word) to signify leaving the critical section.
    * Check if the waiter flag is set. If yes then signal the waiting thread through the sync-array framework.

Looks pretty straightforward and simple. Isn’t it?<br>
Things get complicated with introduction of memory barriers as ignoring them would mean re-ordering can cause race in your code.

<img src="/images/blog9/img2.png" width="80%" class="centerimg"/>
<hr><br>

* From the above sequence it is pretty clear that while locking m_lock_word (false->true) it could potentially begin the critical section (if CAX succeeds) and so flow shouldn’t execute any statement from the critical section before the lock word is acquired (set to true).
* Going back to our acquire-release barrier section it suggests m_lock_word should take an <span style="color:#1aa260">**acquire barrier** </span> incase of success **<span style="color:#de5246">(instead of default (seq_cst) as it currently does)</span>**.
* But wait, there are 2 potential outcomes. What about failure? Even in case of failure, followup actions like spin, sleep and set-waiter should be done only post CAX evaluation. This again suggests use of an <span style="color:#1aa260">**acquire barrier**</span> for failure case too.
**<span style="color:#de5246">(instead of default (seq_cst) as it currently does)</span>**.

<img src="/images/blog9/img3.png" width="80%" class="centerimg"/>
<hr><br>

* Now let’s look at the release barrier for m_lock_word. Naturally a <span style="color:#4885ed">**release barrier**</span> will be placed once a critical section is done when the m_lock_word is toggled.

<img src="/images/blog9/img4.png" width="80%" class="centerimg"/>
<hr><br>

* There is another atomic variable (waiter flag) that needs to get a proper barrier too.
* Action to set a waiter flag should be done only when flow has ensured it can get a sync array slot. This naturally invites the need for a <span style="color:#4885ed">**release barrier**</span> so the code is not re-ordered beyond set_waiter. Note: This is different atomic though so the co-ordination of m_lock_word acquire and release will not apply here.

<img src="/images/blog9/img5.png" width="80%" class="centerimg"/>
<hr><br>

* Same way signal logic should be done only after the waiter flag is cleared so it should use an  <span style="color:#1aa260">**acquire barrier**</span> that will ensure it is not re-scheduled before the clear-waiter. **<span style="color:#de5246">(instead of release as it currently does)</span>**.

* This will also help us change the waiter-load flag check to use <span style="color:#800000">**relaxed barrier**</span> (vs acquire). (There is a potential catch here; we will discuss it below).

* With all that in place we should able to get rid of explicit memory_fence too.

<img src="/images/blog9/img6.png" width="80%" class="centerimg"/>
<hr><br>


### <span style="color:#de5246">Anomalies:</span>

* release-barrier on ``lock_word`` followed by an acquire-barrier on ``waiter`` this could be reordered.
  * This was even my understanding of potential risk and I presume that’s why MySQL introduced a fence between these 2 operations. As per the said [blog](https://preshing.com/20170612/can-reordering-of-release-acquire-operations-introduce-deadlock/), C++ standard should limit compilers from doing so.
 <br>
* By using a <span style="color:#800000">**relaxed barrier**</span> for the waiter (while checking for its value) there is potential re-ordering that could move load instruction before the m_lock_word release (note: <span style="color:#4885ed">**release barrier**</span> can allow followup instructions to get scheduled before it).
   * If “waiter” is true then a signal loop will be called. 
   * If “waiter” is false then the signal loop will not be called by this thread but some other thread may call it.
  * What if there are only 2 threads and thread-1 evaluates waiter=false by re-ordering it before the release barrier and then immediately posts that waiter is set to true by thread-2 and goes to wait. Now thread-1 will never signal thread-2.

So using a <span style="color:#800000">**relaxed barrier**</span> is not possible so let’s switch it to use an <span style="color:#1aa260">**acquire barrier** </span> that should avoid moving the followup statement beyond the said point and as clarified above release-acquire needs to follow C++ standard.

All this to help save an extra memory fence. memory-fence intention is to help co-ordinate non-atomic synchronization since our flow has inherent atomic (waiter) using proper memory barrier can help achieve the needed effect. 

So with all that taken-care this is how things would look

<img src="/images/blog9/img7.png" width="80%" class="centerimg"/>
<hr><br>

### <span style="color:#1aa260">What we gained from this revamp?</span>

So we achieved 3 things
* Corrected use of memory barrier that helps also clarify the code/flow/developer intention. (This is one of the important thing stressed with use of memory barrier. Correct use will help make the code flow naturally obvious to understand and follow).

* Moved from strict sequential ordering to one-way barrier without loosing on correctness. (acquire and release)
 
* Avoided use of fence memory barrier meant for synchronization of non-atomic.

Revamp is not justified unless it has performance impact and this revamp is no exception. Revamp helps improve performance
<span style="background-color: #FFFF00">on ARM in range of 4-15% and on x86_64 in range of 4-6%.</span>
<hr><br>

## <span style="color:#4885ed">Conclusion</span>

Atomics are good but memory-barrier make them challanging and ensuring proper use of these barriers is key to the optimal performance on all platforms. Adaptation of barrier is catching up but still naive (though present in C+11) as most of the softwares recently started adapting to it. Proper use of barrier help clear the intention too.

<br>
<em>If you have more questions/queries do let me know. Will try to answer them.</em>

<font size="1">Index-Page Link Icons made by <a href="https://www.flaticon.com/authors/freepik" title="Freepik">Freepik</a> from Flaticon</font>
