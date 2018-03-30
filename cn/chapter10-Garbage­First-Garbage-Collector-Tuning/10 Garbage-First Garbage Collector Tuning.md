10 Garbage-First Garbage Collector Tuning
============================================================

This section describes how to adapt and tune the Garbage-First garbage collector (G1 GC) for evaluation, analysis and performance. 

As described in the section [Garbage-First Garbage Collector][14], the G1 GC is a regionalized and generational garbage collector, which means that the Java object heap (heap) is divided into a number of equally sized regions. Upon startup, the Java Virtual Machine (JVM) sets the region size. The region sizes can vary from 1 MB to 32 MB depending on the heap size. The goal is to have no more than 2048 regions. The eden, survivor, and old generations are logical sets of these regions and are not contiguous.

The G1 GC has a pause time target that it tries to meet (soft real time). During young collections, the G1 GC adjusts its young generation (eden and survivor sizes) to meet the soft real-time target. See the sections [Pauses][15] and [Pause Time Goal][16] in [Garbage-First Garbage Collector][17] for information about why the G1 GC takes pauses and how to set pause time targets.

During mixed collections, the G1 GC adjusts the number of old regions that are collected based on a target number of mixed garbage collections, the percentage of live objects in each region of the heap, and the overall acceptable heap waste percentage.

The G1 GC reduces heap fragmentation by incremental parallel copying of live objects from one or more sets of regions (called Collection Sets (CSet)s) into one or more different new regions to achieve compaction. The goal is to reclaim as much heap space as possible, starting with those regions that contain the most reclaimable space, while attempting to not exceed the pause time goal (garbage first).

The G1 GC uses independent Remembered Sets (RSets) to track references into regions. Independent RSets enable parallel and independent collection of regions because only a region's RSet must be scanned for references into that region, instead of the whole heap. The G1 GC uses a post-write barrier to record changes to the heap and update the RSets.

### Garbage Collection Phases

Apart from evacuation pauses (see the section [Allocation (Evacuation) Failure][18] in [Garbage-First Garbage Collector][19]) that compose the stop-the-world (STW) young and mixed garbage collections, the G1 GC also has parallel, concurrent, and multiphase marking cycles. G1 GC uses the snapshot-at-the-beginning (SATB) algorithm, which logically takes a snapshot of the set of live objects in the heap at the start of a marking cycle. The set of live objects also includes objects allocated since the start of the marking cycle. The G1 GC marking algorithm uses a pre-write barrier to record and mark objects that are part of the logical snapshot.

### Young Garbage Collections

The G1 GC satisfies most allocation requests from regions added to the eden set of regions. During a young garbage collection, the G1 GC collects both the eden regions and the survivor regions from the previous garbage collection. The live objects from the eden and survivor regions are copied, or evacuated, to a new set of regions. The destination region for a particular object depends upon the object's age; an object that has aged sufficiently evacuates to an old generation region (that is, it is promoted); otherwise, the object evacuates to a survivor region and will be included in the CSet of the next young or mixed garbage collection. 

### Mixed Garbage Collections

Upon successful completion of a concurrent marking cycle, the G1 GC switches from performing young garbage collections to performing mixed garbage collections. In a mixed garbage collection, the G1 GC optionally adds some old regions to the set of eden and survivor regions that will be collected. The exact number of old regions added is controlled by a number of flags (see "Taming Mixed Garbage Collectors" in the section [Recommendations][20]). After the G1 GC collects a sufficient number of old regions (over multiple mixed garbage collections), G1 reverts to performing young garbage collections until the next marking cycle completes.

### Phases of the Marking Cycle

The marking cycle has the following phases:

*   **Initial marking phase**: The G1 GC marks the roots during this phase. This phase is piggybacked on a normal (STW) young garbage collection.

*   **Root region scanning phase**: The G1 GC scans survivor regions marked during the initial marking phase for references to the old generation and marks the referenced objects. This phase runs concurrently with the application (not STW) and must complete before the next STW young garbage collection can start.

*   **Concurrent marking phase**: The G1 GC finds reachable (live) objects across the entire heap. This phase happens concurrently with the application, and can be interrupted by STW young garbage collections.

*   **Remark phase**: This phase is STW collection and helps the completion of the marking cycle. G1 GC drains SATB buffers, traces unvisited live objects, and performs reference processing.

*   **Cleanup phase**: In this final phase, the G1 GC performs the STW operations of accounting and RSet scrubbing. During accounting, the G1 GC identifies completely free regions and mixed garbage collection candidates. The cleanup phase is partly concurrent when it resets and returns the empty regions to the free list. 

### Important Defaults

The G1 GC is an adaptive garbage collector with defaults that enable it to work efficiently without modification. [Table 10-1, "Default Values of Important Options for G1 Garbage Collector"][21] lists of important options and their default values in Java HotSpot VM, build 24\. You can adapt and tune the G1 GC to your application performance needs by entering the options in [Table 10-1, "Default Values of Important Options for G1 Garbage Collector"][22] with changed settings on the JVM command line.

 Table 10-1 Default Values of Important Options for G1 Garbage Collector

| Option and Default Value | Option |
| --- | --- |
| `-XX:G1HeapRegionSize=``n` | Sets the size of a G1 region. The value will be a power of two and can range from 1 MB to 32 MB. The goal is to have around 2048 regions based on the minimum Java heap size. |
| `-XX:MaxGCPauseMillis=200` | Sets a target value for desired maximum pause time. The default value is 200 milliseconds. The specified value does not adapt to your heap size. |
| `-XX:G1NewSizePercent=5` | Sets the percentage of the heap to use as the minimum for the young generation size. The default value is 5 percent of your Java heap.[Foot1][1]

This is an experimental flag. See [How to Unlock Experimental VM Flags][2] for an example. This setting replaces the `-XX:DefaultMinNewGenPercent` setting. |
| `-XX:G1MaxNewSizePercent=60` | Sets the percentage of the heap size to use as the maximum for young generation size. The default value is 60 percent of your Java heap.[Footref1][3]

This is an experimental flag. See [How to Unlock Experimental VM Flags][4] for an example. This setting replaces the `-XX:DefaultMaxNewGenPercent` setting. |
| `-XX:ParallelGCThreads=``n` | Sets the value of the STW worker threads. Sets the value of `n` to the number of logical processors. The value of `n` is the same as the number of logical processors up to a value of 8.

If there are more than eight logical processors, sets the value of  _n_  to approximately 5/8 of the logical processors. This works in most cases except for larger SPARC systems where the value of  _n_  can be approximately 5/16 of the logical processors. |
| `-XX:ConcGCThreads=``n` | Sets the number of parallel marking threads. Sets `n` to approximately 1/4 of the number of parallel garbage collection threads (ParallelGCThreads). |
| `-XX:InitiatingHeapOccupancyPercent=45` | Sets the Java heap occupancy threshold that triggers a marking cycle. The default occupancy is 45 percent of the entire Java heap. |
| `-XX:G1MixedGCLiveThresholdPercent=85` | Sets the occupancy threshold for an old region to be included in a mixed garbage collection cycle. The default occupancy is 85 percent.[Footref1][5]

This is an experimental flag. See [How to Unlock Experimental VM Flags][6] for an example. This setting replaces the `-XX:G1OldCSetRegionLiveThresholdPercent` setting. |
| `-XX:G1HeapWastePercent=5` | Sets the percentage of heap that you are willing to waste. The Java HotSpot VM does not initiate the mixed garbage collection cycle when the reclaimable percentage is less than the heap waste percentage. The default is 5 percent.[Footref1][7] |
| `-XX:G1MixedGCCountTarget=8` | Sets the target number of mixed garbage collections after a marking cycle to collect old regions with at most `G1MixedGCLIveThresholdPercent` live data. The default is 8 mixed garbage collections. The goal for mixed collections is to be within this target number.[Footref1][8] |
| `-XX:G1OldCSetRegionThresholdPercent=10` | Sets an upper limit on the number of old regions to be collected during a mixed garbage collection cycle. The default is 10 percent of the Java heap.[Footref1][9] |
| `-XX:G1ReservePercent=10` | Sets the percentage of reserve memory to keep free so as to reduce the risk of to-space overflows. The default is 10 percent. When you increase or decrease the percentage, make sure to adjust the total Java heap by the same amount.[Footref1][10]|  
 
 Footnote1This setting is not available in Java HotSpot VM build 23 or earlier. 

### How to Unlock Experimental VM Flags

To change the value of experimental flags, you must unlock them first. You can do this by setting `-XX:+UnlockExperimentalVMOptions` explicitly on the command line before any experimental flags. For example:

`java -XX:+UnlockExperimentalVMOptions -XX:G1NewSizePercent=10 -XX:G1MaxNewSizePercent=75 G1test.jar` 

### Recommendations

When you evaluate and fine-tune G1 GC, keep the following recommendations in mind: 

*   Young Generation Size: Avoid explicitly setting young generation size with the -Xmn option or any or other related option such as -XX:NewRatio. Fixing the size of the young generation overrides the target pause-time goal.

*   Pause Time Goals: When you evaluate or tune any garbage collection, there is always a latency versus throughput trade-off. The G1 GC is an incremental garbage collector with uniform pauses, but also more overhead on the application threads. The throughput goal for the G1 GC is 90 percent application time and 10 percent garbage collection time. Compare this to the Java HotSpot VM parallel collector. The throughput goal of the parallel collector is 99 percent application time and 1 percent garbage collection time. Therefore, when you evaluate the G1 GC for throughput, relax your pause time target. Setting too aggressive a goal indicates that you are willing to bear an increase in garbage collection overhead, which has a direct effect on throughput. When you evaluate the G1 GC for latency, you set your desired (soft) real-time goal, and the G1 GC will try to meet it. As a side effect, throughput may suffer. See the section [Pause Time Goal][11] in [Garbage-First Garbage Collector][12] for additional information.

*   Taming Mixed Garbage Collections: Experiment with the following options when you tune mixed garbage collections. See the section [Important Defaults][13] for information about these options:-XX:InitiatingHeapOccupancyPercent: Use to change the marking threshold.-XX:G1MixedGCLiveThresholdPercent and -XX:G1HeapWastePercent: Use to change the mixed garbage collection decisions.-XX:G1MixedGCCountTarget and -XX:G1OldCSetRegionThresholdPercent: Use to adjust the CSet for old regions.

### Overflow and Exhausted Log Messages

When you see to-space overflow or to-space exhausted messages in your logs, the G1 GC does not have enough memory for either survivor or promoted objects, or for both. The Java heap cannot because it is already at its maximum. Example messages:

*   `924.897: [GC pause (G1 Evacuation Pause) (mixed) (to-space exhausted), 0.1957310 secs]`

*   `924.897: [GC pause (G1 Evacuation Pause) (mixed) (to-space overflow), 0.1957310 secs]`

To alleviate the problem, try the following adjustments:

*   Increase the value of the `-XX:G1ReservePercent` option (and the total heap accordingly) to increase the amount of reserve memory for "to-space".

*   Start the marking cycle earlier by reducing the value of `-XX:InitiatingHeapOccupancyPercent.`

*   Increase the value of the `-XX:ConcGCThreads` option to increase the number of parallel marking threads. 

See the section [Important Defaults][23] for a description of these options.

### Humongous Objects and Humongous Allocations

For G1 GC, any object that is more than half a region size is considered a  _humongous object_ . Such an object is allocated directly in the old generation into  _humongous regions_ . These humongous regions are a contiguous set of regions. `StartsHumongous` marks the start of the contiguous set and `ContinuesHumongous` marks the continuation of the set.

Before allocating any humongous region, the marking threshold is checked, initiating a concurrent cycle, if necessary.

Dead humongous objects are freed at the end of the marking cycle during the cleanup phase and also during a full garbage collection cycle.

To reduce copying overhead, the humongous objects are not included in any evacuation pause. A full garbage collection cycle compacts humongous objects in place.

Because each individual set of `StartsHumongous` and `ContinuesHumongous` regions contains just one humongous object, the space between the end of the humongous object and the end of the last region spanned by the object is unused. For objects that are just slightly larger than a multiple of the heap region size, this unused space can cause the heap to become fragmented.

If you see back-to-back concurrent cycles initiated due to humongous allocations and if such allocations are fragmenting your old generation, then increase the value of `-XX:G1HeapRegionSize` such that previous humongous objects are no longer humongous and will follow the regular allocation path.

[1]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc_tuning.html#not_available_hotspot_vm_build_23
[2]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc_tuning.html#how_to_unlock_experimental_vm_flags
[3]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc_tuning.html#sthref55
[4]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc_tuning.html#how_to_unlock_experimental_vm_flags
[5]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc_tuning.html#sthref56
[6]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc_tuning.html#how_to_unlock_experimental_vm_flags
[7]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc_tuning.html#sthref57
[8]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc_tuning.html#sthref58
[9]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc_tuning.html#sthref59
[10]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc_tuning.html#sthref60
[11]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc.html#pause_time_goal
[12]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc.html#garbage_first_garbage_collection
[13]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc_tuning.html#important_defaults
[14]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc.html#garbage_first_garbage_collection
[15]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc.html#pauses
[16]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc.html#pause_time_goal
[17]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc.html#garbage_first_garbage_collection
[18]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc.html#allocation_evacuation_failure
[19]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc.html#garbage_first_garbage_collection
[20]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc_tuning.html#recommendations
[21]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc_tuning.html#default_g1_gc
[22]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc_tuning.html#default_g1_gc
[23]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc_tuning.html#important_defaults