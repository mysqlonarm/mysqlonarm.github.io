---
layout: post
title: Mariabackup on ARM
img: ./images/blog18/mbackuponarm.png
author: Krunal Bauskar
update: 27th Jan 2021
---

One of the most important things an user does on a regular basis is backup of the database. MariaDB offers Mariabackup that helps users to take full and incremental backup. Mariabackup is already supported on ARM so let’s explore its performance on ARM.

## <span style="color:#4885ed">MariaBackup on ARM</span>

Taking backup using mariabackup involves 3 main stages:
   * Backup: Live backup from an active mariadb-server instance.
   * Prepare: This is like redo-log-based recovery that restores the backup to the sane state.
   * Move-Back/Copy-Back: Restoring the backup as data-directory for the new instance.

Backup is like risk protection and so backup step is executed regularly but prepare and move/copy-back steps are used only if the said backup needs to be restored which is when a server instance crashes or something gets accidentally deleted or for booting a new replicas.

## <span style="color:#4885ed">Backup Step</span>

Let’s consider backup under 3 different scenarios:
   * No workload active
   * Read-Only workload active
   * Read-Write workload active
   
Both ARM and x86 have the same resources
   * Compute: 64 vCPU (2 numa nodes) [different compute power 2.6 * 64 < 3 * 64]
      * ARM: 2.6 GHz Kupeng 920
      * x86: Intel(R) Xeon(R) Gold 6151 CPU @ 3.00GHz
   * Storage: NVMe SSD (1.6 TB)
   * Memory: 192GB.
   
<i>Cost of the machine is of-course different, with ARM being 50+% cheaper.</i>

Base invocation is without any special options<br>
```mariabackup --backup --target-dir=/data/mdb-data/backup --user=backup --password=backup --socket="socket"```

Workload in form of sysbench client (oltp-read-write use-case) with 512 threads is kept running while backup is active.

### <span style="color:#0F9D58">No workload active</span>

   * No workload is active, only backup is running.
   * With a seed database (clean loaded database so neatly packed), time taken for backup is 2-26% less for ARM (vs x86).
   * With a production used database (split, empty/half-filled pages), time taken for backup is 18% less for ARM (vs x86).
 
<img src="/images/blog18/backup-no-workload.png" height="400" class="centerimg"/>

### <span style="color:#0F9D58">Read-Only workload active</span>

   * “read-only” workload is active when the backup is taken.
   * Again, time taken for backup in the ARM case is less than that of x86.
   * Another interesting observation is about the tps drop.
      * For single thread backup there is no or marginal tps drop during backup (backup starts @ 300 secs for each use-case)
      * For multi-threaded backup there is 2-3% drop in tps for both ARM and x86 cases. Since x86 takes longer, drop continues for longer period.
 
<img src="/images/blog18/backup-ro-workload-parallel1.png" height="400" class="centerimg"/>
<br>
<img src="/images/blog18/backup-ro-workload-parallel64.png" height="400" class="centerimg"/>

### <span style="color:#0F9D58">Read-Write workload active (Full Backup)</span>

   * "read-write" workload is active when the backup is taken.
   * MariaDB on ARM continues to take lesser time. (17-27%)
   * Overall tps is comparable with both architecture scoring in some cases.
   * x86 throughput seems to have lot of jitter during the backup-phase and so throughput is 20% lesser during backup (compared to ARM).
  
<img src="/images/blog18/backup-rw-workload-parallel1.png" height="400" class="centerimg"/>
<br>
<img src="/images/blog18/backup-rw-workload-parallel64.png" height="400" class="centerimg"/>

### <span style="color:#0F9D58">Read-Write workload active (Incremental Backup)</span>

   * Incremental backup is done while read-write workload is active.
   * During backup there is marginal drop in tps (arm=1.3%, x86= 8.5%)
   * Overall time for backup is 28% less with ARM.
   * Jitter with throughput continue with x86 signficantly affected.
 
<img src="/images/blog18/incr-backup-rw-workload-parallel1.png" height="400" class="centerimg"/>
<br>
<img src="/images/blog18/incr-backup-rw-workload-parallel64.png" height="400" class="centerimg"/>

## <span style="color:#4885ed">Prepare</span>

Prepare stage runs redo-recovery algorithm so prepare stage makes sense only when backup is taken with active read-write workload. Let’s find out the performance of the prepare stage.

### <span style="color:#0F9D58">Preparing backup taken with Read-Write workload active</span>
   * Backup was taken while read-write was active so redo-log recovery kicked in on prepare.
   * There is no major difference in the prepare-time for full-backup. For incremental x86 is slighly faster (8%).

```mariabackup --prepare --use-memory=80G --target-dir=/data/mdb-data/backup```

<img src="/images/blog18/prepare.png" height="400" class="centerimg"/>

## <span style="color:#4885ed">Conclusion</span>

Overall, Mariabackup on ARM is performing on par/better than Mariabackup on x86. Importantly, we see a less jitter in throughput with ARM and considerable reduction in time for backup.

<br>
<em>If you have more questions/queries do let me know. Will try to answer them.</em>
