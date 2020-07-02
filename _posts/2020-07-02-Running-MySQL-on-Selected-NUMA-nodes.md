---
layout: post
title: Running MySQL on selected NUMA Node(s)
img: ./images/blog8/numa-node-selection.png
author: Krunal Bauskar
update: 2nd July 2020
---

"Running MySQL on selected NUMA node(s)” looks pretty straightforward but unfortunately it isn’t. Recently, I was faced with a situation that demanded running MySQL on 2 (out of 4) NUMA nodes.

Naturally, the first thing I tried was to restrict CPU/Core set using ``numactl --physcpubind`` selecting only the said CPUs/cores from the said NUMA nodes. MySQL was configured to use ``innodb_numa_interleave=1`` so I was expecting it to allocate memory from the said NUMA nodes only (as I restricted usage of CPU/core).

### <span style="color:#de5246">Suprise-1:</span>

MySQL uses ``numa_all_nodes_ptr->maskp`` that means all the nodes are opted even though the CPU task-set is limited to 2 NUMA nodes.

Some lookout pointed me to these 2 issues from Daniel Black<br>
* https://github.com/mysql/mysql-server/pull/104 (5.7)<br>
* https://github.com/mysql/mysql-server/pull/138 (8.0)

Issue proposes to switch to a more logical ``numa_get_mems_allowed()``. As per the documentation it should return a mask of the node that are are allowed to allocate memory for the said process.

``
ref-from-doc: numa_get_mems_allowed() returns the mask of nodes from which the process is allowed to allocate memory in it's current cpuset context.
``

So I decided to apply the patch and proceed.

### <span style="color:#de5246">Suprise-2:</span>

Just applying patch and relying on cpu/core set didn't helped. So I thought of trying with membind option.

### <span style="color:#de5246">Suprise-3:</span>

So now the command looks like:

``numactl --physcpubind=<cpu-set-from-numa-node-0,1> --membind=0,1``

This time I surely expected that memory would be allocated from the said NUMA nodes only but it still didn’t. Memory was allocated from all 4 nodes.

Some more documentation search, suggested that for ``numa_all_nodes_ptr`` looks at ``mems_allowed`` field as mentioned below<br>

``
numa_all_nodes_ptr: The set of nodes to record is derived from /proc/self/status, field "Mems_allowed". The user should not alter this bitmask.
``

and as Alexey Kopytov pointed in PR#138, ``numa_all_nodes_ptr`` and  ``numa_get_mems_allowed`` reads the same mask.

This tends to suggest that ``numa_get_mems_allowed`` is broken or documentation needs to be updated.

<em>Just for completeness, I also tried numctl  --interleave but that too didn't helped</em>

### <span style="color:#4885ed">Fact Validation:</span>

So I decided to try this using a simple program (outside MySQL) to validate the said fact.

```
#include <iostream>
#include <numa.h>
#include <numaif.h>
using namespace std;
int main()
{
cout << *numa_all_nodes_ptr->maskp << endl;
cout << *numa_get_mems_allowed()->maskp << endl;
}

numactl --membind=0-1 ./a.out
15
15
```

It is pretty clear that both seem to return the same mask value when ``numa_get_mems_allowed`` should return only memory allowed nodes.

### <span style="color:#1aa260">Workaround:</span>

I desperately needed a solution so tried using a simple workaround of manually feeding the mask (will continue to follow up about numactl behavior with OS vendor). This approach finally worked and now I can allocate memory from selected NUMA nodes only.

````
+const unsigned long numa_mask = 0x3;
 
 struct set_numa_interleave_t {
   set_numa_interleave_t() {
     if (srv_numa_interleave) {
       ib::info(ER_IB_MSG_47) << "Setting NUMA memory policy to"
                                 " MPOL_INTERLEAVE";
-      if (set_mempolicy(MPOL_INTERLEAVE, numa_all_nodes_ptr->maskp,
+      if (set_mempolicy(MPOL_INTERLEAVE, &numa_mask,
                         numa_all_nodes_ptr->size) != 0) {
         ib::warn(ER_IB_MSG_48) << "Failed to set NUMA memory"
                                   " policy to MPOL_INTERLEAVE: "
@@ -1000,7 +1001,7 @@ static buf_chunk_t *buf_chunk_init(
 #ifdef HAVE_LIBNUMA
   if (srv_numa_interleave) {
     int st = mbind(chunk->mem, chunk->mem_size(), MPOL_INTERLEAVE,
-                   numa_all_nodes_ptr->maskp, numa_all_nodes_ptr->size,
+                   &numa_mask, numa_all_nodes_ptr->size,
                    MPOL_MF_MOVE);
     if (st != 0) {
       ib::warn(ER_IB_MSG_54) << "Failed to set NUMA memory policy of"

```

(Of-course this needs re-build from source code and not an option for binary/package user (well there is .. check following section)).

### <span style="color:#1aa260">But then why didn't you used ... ?</span>

Naturally, most of you may suggest that this could be avoided by toggling ``innodb_numa_interleave`` back to OFF and using membind. Of-course this approach works but this approach is slightly different because then all the memory allocated is bounded by the said restriction vs ``innodb_numa_interleave`` is applicable only during buffer pool allocation. It may serve specific purpose but may not be so called comparable.

This has been on my todo list to check effect of complete interleave vs innodb_numa_interleave.

## <span style="color:#4885ed">Conclusion</span>

Balance distribution on NUMA node has multiple aspects including core-selection, memory allocation, thread allocation (equally on selected numa node), etc.... Lot of exciting and surprising things to explore.

<br>
<em>If you have more questions/queries do let me know. Will try to answer them.</em>
