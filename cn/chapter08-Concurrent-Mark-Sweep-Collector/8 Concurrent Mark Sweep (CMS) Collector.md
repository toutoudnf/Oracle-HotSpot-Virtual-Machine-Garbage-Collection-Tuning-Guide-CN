8 Concurrent Mark Sweep (CMS) Collector
============================================================

The Concurrent Mark Sweep (CMS) collector is designed for applications that prefer shorter garbage collection pauses and that can afford to share processor resources with the garbage collector while the application is running. Typically applications that have a relatively large set of long-lived data (a large tenured generation) and run on machines with two or more processors tend to benefit from the use of this collector. However, this collector should be considered for any application with a low pause time requirement. The CMS collector is enabled with the command-line option `-XX:+UseConcMarkSweepGC`.

Concurrent Mark Sweep (CMS) 这种垃圾收集器很神奇，他的设计初衷其实有两点：

- 尽可能缩短因为垃圾回收导致的暂停（STW)
- 利用多处理器特点，在垃圾回收的同时，还能并行的运行应用程序

CMS 适用于什么场景呢？比如那些有大量的在 tuned generation 存活时间挺长的对象的场景，同时最好还是多处理器的。这是因为多处理器能很好利用 CMS 并发的特点。判断使用 CMS 的场景其主要原则就是是否追求更小的 GC 暂停时间。通过 `-XX:+UseConcMarkSweepGC` 参数来开启 CMS 垃圾收集器。

CMS 也是分代的，对应的有 minor GC & major GC。为啥 CMS 能减少 GC 时间呢？其实是空间换时间的方法。

> 译者注：GC STW 是为了在 heap 中找到那些可达的对象，但是如果 application thread 在工作，这个事儿就不靠谱。所以当 heap 比较大的时候，STW 的时间会比较长，从而能标记处全部的不可达对象然后清理。CMS 的思路是，我允许你应用和我标记的过程并发，只要我把好门就行了。这个门其实就是 thread 执行某些可能引起对象可达不可达状态发生变化的操作。CMS 表示看好你这些小动作就 ok 了。具体 CMS 的过程如下。

CMS 过程的大体介绍如下：

- 开始的时候，CMS 执行 STW，做第一次标记
- 中间是 CMS GC thread & application thread 共通执行的过程
- 然后 CMS 第二次触发 STW（通常比第一次要长），在此期间，并发收集垃圾。
- 最后 CMS 会采用多个线程并发的进行垃圾清理（此时应用也在运行哦）。minor gc 可能在这个过程中被触发。

Similar to the other available collectors, the CMS collector is generational; thus both minor and major collections occur. The CMS collector attempts to reduce pause times due to major collections by using separate garbage collector threads to trace the reachable objects concurrently with the execution of the application threads. During each major collection cycle, the CMS collector pauses all the application threads for a brief period at the beginning of the collection and again toward the middle of the collection. The second pause tends to be the longer of the two pauses. Multiple threads are used to do the collection work during both pauses. The remainder of the collection (including most of the tracing of live objects and sweeping of unreachable objects is done with one or more garbage collector threads that run concurrently with the application. Minor collections can interleave with an ongoing major cycle, and are done in a manner similar to the parallel collector (in particular, the application threads are stopped during minor collections).

### Concurrent Mode Failure

### Concurrent Mode Failure

> 这个标题不准备翻译，解释下意思吧：Concurrent 模式下的一种失败场景。

The CMS collector uses one or more garbage collector threads that run simultaneously with the application threads with the goal of completing the collection of the tenured generation before it becomes full. As described previously, in normal operation, the CMS collector does most of its tracing and sweeping work with the application threads still running, so only brief pauses are seen by the application threads. However, if the CMS collector is unable to finish reclaiming the unreachable objects before the tenured generation fills up, or if an allocation cannot be satisfied with the available free space blocks in the tenured generation, then the application is paused and the collection is completed with all the application threads stopped. The inability to complete a collection concurrently is referred to as  _concurrent mode failure_  and indicates the need to adjust the CMS collector parameters. If a concurrent collection is interrupted by an explicit garbage collection (`System.gc()`) or for a garbage collection needed to provide information for diagnostic tools, then a concurrent mode interruption is reported.

前面解释过，CMS 为了减少 application 可感知的 GC 导致的暂停，某些阶段是允许 GC 过程与应用同时运行的。但是问题就来了，GC 标记全部的垃圾 & 回收是需要时间的，并且你也保不齐坑爹应用突然快速的大量的给你整一堆对象。这样在 CMS major GC 的过程当中，就可能出现 tuned generation 不够使的情况。application 就不干了啊给 JVM 叫冤，说你看那老些垃圾占地方都没我的地儿了。然后 JVM 没辙，停掉正在进行的 CMS major GC，改用全程 STW 的一次 fullgc 来完成垃圾的回收。这个情况在 CMS major GC 的过程当中就叫做 _concurrent mode failure_ 。这个时候你就需要调整 CMS 相关参数来解决这个问题了。

除了上述情况，显示的 gc 调用（`System.gc()`）或者一些性能诊断工具触发的 gc，也会触发这个 mode。

### Excessive GC Time and OutOfMemoryError

### GC 时间过长导致的内存溢出错误（OutOfMemoryError）

The CMS collector throws an `OutOfMemoryError` if too much time is being spent in garbage collection: if more than 98% of the total time is spent in garbage collection and less than 2% of the heap is recovered, then an `OutOfMemoryError` is thrown. This feature is designed to prevent applications from running for an extended period of time while making little or no progress because the heap is too small. If necessary, this feature can be disabled by adding the option `-XX:-UseGCOverheadLimit` to the command line.

针对 CMS GC 的过程，还有一种情况也是需要避免的：tuned generation 比较小的情况下，CMS major gc 频繁的被触发，但是并没有什么新的垃圾产生。这种情况导致的结果就是大量时间+资源花费在了 GC 上，影响了 application 运行的效率。针对这种情况，CMS 做了优化：假设 CMS collector 发现花费了 98% 的时间在 GC 上（包括并发的时间），但是每次 GC 效果连总 heap 的 2% 有效空间都回收不到，这时候就会抛出 `OutOfMemoryError`。这个特性是可以通过 JVM 参数 `-XX:-UseGCOverheadLimit` 关闭的。

The policy is the same as that in the parallel collector, except that time spent performing concurrent collections is not counted toward the 98% time limit. In other words, only collections performed while the application is stopped count toward excessive GC time. Such collections are typically due to a concurrent mode failure or an explicit collection request (for example, a call to `System.gc`).

待补充

### Floating Garbage

### 浮动垃圾

The CMS collector, like all the other collectors in Java HotSpot VM, is a tracing collector that identifies at least all the reachable objects in the heap. In the parlance of Richard Jones and Rafael D. Lins in their publication  _Garbage Collection: Algorithms for Automated Dynamic Memory_ , it is an incremental update collector. Because application threads and the garbage collector thread run concurrently during a major collection, objects that are traced by the garbage collector thread may subsequently become unreachable by the time collection process ends. Such unreachable objects that have not yet been reclaimed are referred to as floating garbage. The amount of  _floating garbage_  depends on the duration of the concurrent collection cycle and on the frequency of reference updates, also known as  _mutations_ , by the application. Furthermore, because the young generation and the tenured generation are collected independently, each acts a source of roots to the other. As a rough guideline, try increasing the size of the tenured generation by 20% to account for the floating garbage. Floating garbage in the heap at the end of one concurrent collection cycle is collected during the next collection cycle.

这段会讲浮动垃圾（floating garbage）是怎么来的：floating garbage 是指在当前的 CMS 垃圾收集阶段，某些一开始不是垃圾的对象，在 CMS 与 application 并发的阶段，变成了垃圾。然后这些对象在 heap 中又没有被收集，故称之为浮动垃圾。在 [Garbage Collection: Algorithms for Automated Dynamic Memory](https://book.douban.com/subject/2135376/) 这本书当中（译者注：神书，深入研究需细看），作者把 CMS 收集器称之为增量收集器（incremental update collector）。另外根据实际经验，针对 floating garbage，你需要预留 20% 的 tenured generation 来保证不会发生内存溢出啊啥的。

### Pauses

### GC 暂停

The CMS collector pauses an application twice during a concurrent collection cycle. The first pause is to mark as live the objects directly reachable from the roots (for example, object references from application thread stacks and registers, static objects and so on) and from elsewhere in the heap (for example, the young generation). This first pause is referred to as the  _initial mark pause_ . The second pause comes at the end of the concurrent tracing phase and finds objects that were missed by the concurrent tracing due to updates by the application threads of references in an object after the CMS collector had finished tracing that object. This second pause is referred to as the  _remark pause_ .

对于一个完成的 CMS 回收周期来讲，共有两次暂停，分别是：

- initial mark pause：初始标记暂停
- remark pause：重新标记暂停

两次标记都是找 live object，其中初始标记的出发点（root）包括：线程栈啊（application thread stacks）、registers、静态对象（static objects）等等，另外还有来自 heap 中非 tenured gen 部分的引用（如 young generation）；而重新标记是为了找到那些在并发过程中遗漏的 lived object。

### Concurrent Phases

### 并发阶段

The concurrent tracing of the reachable object graph occurs between the initial mark pause and the remark pause. During this concurrent tracing phase one or more concurrent garbage collector threads may be using processor resources that would otherwise have been available to the application. As a result, compute-bound applications may see a commensurate fall in application throughput during this and other concurrent phases even though the application threads are not paused. After the remark pause, a concurrent sweeping phase collects the objects identified as unreachable. Once a collection cycle completes, the CMS collector waits, consuming almost no computational resources, until the start of the next major collection cycle.

并发阶段也有两个：初始标记和重新标记之间的阶段，以及重新标记后，垃圾的并发清理阶段。并发阶段会采用多个线程处理 GC 相关的事务，所以会占用处理器资源。这时候计算密集型的应用可能受影响会比较明显。当 CMS 垃圾收集周期结束之后，CMS gc thread 就会处于 waiting 的状态，基本不消耗 cpu 资源。

### Starting a Concurrent Collection Cycle

### 什么时候开始一个收集周期

With the serial collector a major collection occurs whenever the tenured generation becomes full and all application threads are stopped while the collection is done. In contrast, the start of a concurrent collection must be timed such that the collection can finish before the tenured generation becomes full; otherwise, the application would observe longer pauses due to concurrent mode failure. There are several ways to start a concurrent collection.

对于非并发的那种垃圾收集器，开始的时间点很好选择：tenured generation 满了就开始收集。但是对于 CMS 这种，收集过程中存在并发阶段的收集器，需要选择一个合理的开始时间，否则收集着收集着 tenured generation 满了，就会切换到 full gc 模式，gc 的时间反而要比要来久了（就是前面提到的 Concurrent Mode Failure）。

Based on recent history, the CMS collector maintains estimates of the time remaining before the tenured generation will be exhausted and of the time needed for a concurrent collection cycle. Using these dynamic estimates, a concurrent collection cycle is started with the aim of completing the collection cycle before the tenured generation is exhausted. These estimates are padded for safety, because concurrent mode failure can be very costly.

CMS 收集器会根据历史数据分析 tenured generation 部分什么时候会填满，然后尽量保证收集周期之内，不会出现 tenured generation 满了的情况，从而避免 full gc（concurrent mode failure）。

A concurrent collection also starts if the occupancy of the tenured generation exceeds an initiating occupancy (a percentage of the tenured generation). The default value for this initiating occupancy threshold is approximately 92%, but the value is subject to change from release to release. This value can be manually adjusted using the command-line option `-XX:CMSInitiatingOccupancyFraction=<N>`, where `<N>` is an integral percentage (0 to 100) of the tenured generation size.

如果当 tenured generation 使用率到达了一个百分比（initiating occupancy），CMS 收集周期也会开始。这个值初始大约为 92%，每个版本可能不同。这个值可以通过 `-XX:CMSInitiatingOccupancyFraction=<N>` 来指定，其中 N 是个 0 - 100 的整数，表示的是 tenured generation 的比率。

### Scheduling Pauses

### 暂停时间的优化

The pauses for the young generation collection and the tenured generation collection occur independently. They do not overlap, but may occur in quick succession such that the pause from one collection, immediately followed by one from the other collection, can appear to be a single, longer pause. To avoid this, the CMS collector attempts to schedule the remark pause roughly midway between the previous and next young generation pauses. This scheduling is currently not done for the initial mark pause, which is usually much shorter than the remark pause.

对于 CMS 垃圾收集来说，young gen & tenured gen 的垃圾收集是分开且互相独立的的。这样可能会出现一种情况是，young gen 的垃圾收集（STW）跟 CMS 的垃圾收集的暂停（initial or remark）隔得太近，导致应用感知到的整体暂停时间过长。所以 CMS 会针对自己的 tenured gen 垃圾收集做优化，尽量保证 remark 阶段处于两次 young gen gc 中间（至于init mark 因为通长时间比较短，所以 CMS 不考虑对他的优化）。

### Incremental Mode

### 增量模式（Incremental Mode）

Note that the incremental mode is being deprecated in Java SE 8 and may be removed in a future major release.

要注意这个模式在 Java SE 8 已经不建议使用了，并且可能会在新的版本中移除。

The CMS collector can be used in a mode in which the concurrent phases are done incrementally. Recall that during a concurrent phase the garbage collector thread is using one or more processors. The incremental mode is meant to lessen the effect of long concurrent phases by periodically stopping the concurrent phase to yield back the processor to the application. This mode, referred to here as  _i-cms_ , divides the work done concurrently by the collector into small chunks of time that are scheduled between young generation collections. This feature is useful when applications that need the low pause times provided by the CMS collector are run on machines with small numbers of processors (for example, 1 or 2).

CMS 并发收集阶段会抢占 cpu 资源，对 cpu 比较少的机器是不友好的（cpu 在 1 - 2 这样的）；并且可能因为并发收集周期较长，导致一个 CMS gc 周期中发生多个 young gc。为了解决这类问题，提供了一个 _i-cms_ 模式，即增量收集模式。增量的原理其实就是将并发标记阶段切分为多个子阶段进行，在每个子阶段结束后会将 CPU 资源交还给 application 去使用。每个子阶段尽量都能在两次 ygc 之间完成。

The concurrent collection cycle typically includes the following steps:

*   Stop all application threads, identify the set of objects reachable from roots, and then resume all application threads.

*   Concurrently trace the reachable object graph, using one or more processors, while the application threads are executing.

*   Concurrently retrace sections of the object graph that were modified since the tracing in the previous step, using one processor.

*   Stop all application threads and retrace sections of the roots and object graph that may have been modified since they were last examined, and then resume all application threads.

*   Concurrently sweep up the unreachable objects to the free lists used for allocation, using one processor.

*   Concurrently resize the heap and prepare the support data structures for the next collection cycle, using one processor.

Normally, the CMS collector uses one or more processors during the entire concurrent tracing phase, without voluntarily relinquishing them. Similarly, one processor is used for the entire concurrent sweep phase, again without relinquishing it. This overhead can be too much of a disruption for applications with response time constraints that might otherwise have used the processing cores, particularly when run on systems with just one or two processors. Incremental mode solves this problem by breaking up the concurrent phases into short bursts of activity, which are scheduled to occur midway between minor pauses.

The i-cms mode uses a duty cycle to control the amount of work the CMS collector is allowed to do before voluntarily giving up the processor. The  _duty cycle_  is the percentage of time between young generation collections that the CMS collector is allowed to run. The i-cms mode can automatically compute the duty cycle based on the behavior of the application (the recommended method, known as  _automatic pacing_ ), or the duty cycle can be set to a fixed value on the command line.

### Command-Line Options

[Table 8-1, "Command-Line Options for i-cms"][1] list command-line options that control the i-cms mode. The section [Recommended Options][2] suggests an initial set of options.

Table 8-1 Command-Line Options for i-cms

| Option | Description | Default Value, Java SE 5 and Earlier | Default Value, Java SE 6 and Later |
| --- | --- | --- | --- |
| `-XX:+CMSIncrementalMode`| Enables incremental mode. Note that the CMS collector must also be enabled (with `-XX:+UseConcMarkSweepGC`) for this option to work. | disabled | disabled |
| `-XX:+CMSIncrementalPacing` | Enables automatic pacing. The incremental mode duty cycle is automatically adjusted based on statistics collected while the JVM is running. | disabled | disabled |
| `-XX:CMSIncrementalDutyCycle=<N>` | The percentage (0 to 100) of time between minor collections that the CMS collector is allowed to run. If `CMSIncrementalPacing` is enabled, then this is just the initial value. | 50 | 10 |
| `-XX:CMSIncrementalDutyCycleMin=<N>` | The percentage (0 to 100) that is the lower bound on the duty cycle when `CMSIncrementalPacing` is enabled. | 10 | 0|
| `-XX:CMSIncrementalSafetyFactor=<N>` | The percentage (0 to 100) used to add conservatism when computing the duty cycle | 10 | 10|
| `-XX:CMSIncrementalOffset=<N>` | The percentage (0 to 100) by which the incremental mode duty cycle is shifted to the right within the period between minor collections. | 0 | 0|
| `-XX:CMSExpAvgFactor=<N>` | The percentage (0 to 100) used to weight the current sample when computing exponential averages for the CMS collection statistics. | 25 | 25|

### Recommended Options

To use i-cms in Java SE 8, use the following command-line options:

Java SE 8 中，使用 i-cms 的建议参数配置如下：

```
-XX:+UseConcMarkSweepGC -XX:+CMSIncrementalMode \
-XX:+PrintGCDetails -XX:+PrintGCTimeStamps
```

The first two options enable the CMS collector and i-cms, respectively. The last two options are not required; they simply cause diagnostic information about garbage collection to be written to standard output, so that garbage collection behavior can be seen and later analyzed.

前两个参数分别是为了开启 CMS 垃圾收集器 & i-cms 模式。后两个不是必需的，是垃圾收集的细节日志，方便一些问题的分析和定位。

For Java SE 5 and earlier releases, Oracle recommends using the following as an initial set of command-line options for i-cms:

对于 Java SE 5 以及更早的版本，Oracle 推荐使用下面的参数来开启 i-cms：

```
-XX:+UseConcMarkSweepGC -XX:+CMSIncrementalMode \
-XX:+PrintGCDetails -XX:+PrintGCTimeStamps \
-XX:+CMSIncrementalPacing -XX:CMSIncrementalDutyCycleMin=0
-XX:CMSIncrementalDutyCycle=10
```

The same values are recommended for JavaSE8 although the values for the three options that control i-cms automatic pacing became the default in JavaSE6.

上述参数同样适用于 JavaSE8，只不过最后三个参数从 JavaSE6 开始就是默认的了。

### Basic Troubleshooting

### 基本调试

The i-cms automatic pacing feature uses statistics gathered while the program is running to compute a duty cycle so that concurrent collections complete before the heap becomes full. However, past behavior is not a perfect predictor of future behavior and the estimates may not always be accurate enough to prevent the heap from becoming full. If too many full collections occur, then try the steps in [Table 8-2, "Troubleshooting the i-cms Automatic Pacing Feature"][3], one at a time.

Table 8-2 Troubleshooting the i-cms Automatic Pacing Feature

| Step | Options |
| --- | --- |
| 1\. Increase the safety factor. | `-XX:CMSIncrementalSafetyFactor=<N>` |
| 2\. Increase the minimum duty cycle. | `-XX:CMSIncrementalDutyCycleMin=<N>` |
| 3\. Disable automatic pacing and use a fixed duty cycle. | `-XX:-CMSIncrementalPacing -XX:CMSIncrementalDutyCycle=<N>` |
### Measurements

[Example 8-1, "Output from the CMS Collector"][4] is the output from the CMS collector with the options -verbose:gc and -XX:+PrintGCDetails, with a few minor details removed. Note that the output for the CMS collector is interspersed with the output from the minor collections; typically many minor collections occur during a concurrent collection cycle. CMS-initial-mark indicates the start of the concurrent collection cycle, CMS-concurrent-mark indicates the end of the concurrent marking phase, and CMS-concurrent-sweep marks the end of the concurrent sweeping phase. Not discussed previously is the precleaning phase indicated by CMS-concurrent-preclean. Precleaning represents work that can be done concurrently in preparation for the remark phase CMS-remark. The final phase is indicated by CMS-concurrent-reset and is in preparation for the next concurrent collection.

Example 8-1 Output from the CMS Collector

```
[GC [1 CMS-initial-mark: 13991K(20288K)] 14103K(22400K), 0.0023781 secs]
[GC [DefNew: 2112K->64K(2112K), 0.0837052 secs] 16103K->15476K(22400K), 0.0838519 secs]
...
[GC [DefNew: 2077K->63K(2112K), 0.0126205 secs] 17552K->15855K(22400K), 0.0127482 secs]
[CMS-concurrent-mark: 0.267/0.374 secs]
[GC [DefNew: 2111K->64K(2112K), 0.0190851 secs] 17903K->16154K(22400K), 0.0191903 secs]
[CMS-concurrent-preclean: 0.044/0.064 secs]
[GC [1 CMS-remark: 16090K(20288K)] 17242K(22400K), 0.0210460 secs]
[GC [DefNew: 2112K->63K(2112K), 0.0716116 secs] 18177K->17382K(22400K), 0.0718204 secs]
[GC [DefNew: 2111K->63K(2112K), 0.0830392 secs] 19363K->18757K(22400K), 0.0832943 secs]
...
[GC [DefNew: 2111K->0K(2112K), 0.0035190 secs] 17527K->15479K(22400K), 0.0036052 secs]
[CMS-concurrent-sweep: 0.291/0.662 secs]
[GC [DefNew: 2048K->0K(2112K), 0.0013347 secs] 17527K->15479K(27912K), 0.0014231 secs]
[CMS-concurrent-reset: 0.016/0.016 secs]
[GC [DefNew: 2048K->1K(2112K), 0.0013936 secs] 17527K->15479K(27912K), 0.0014814 secs
]
```

The initial mark pause is typically short relative to the minor collection pause time. The concurrent phases (concurrent mark, concurrent preclean and concurrent sweep) normally last significantly longer than a minor collection pause, as indicated by [Example 8-1, "Output from the CMS Collector"][5]. Note, however, that the application is not paused during these concurrent phases. The remark pause is often comparable in length to a minor collection. The remark pause is affected by certain application characteristics (for example, a high rate of object modification can increase this pause) and the time since the last minor collection (for example, more objects in the young generation may increase this pause).

[1]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/cms.html#i_cms_command_line_options
[2]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/cms.html#i_cms_recommended_options
[3]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/cms.html#troubleshooting_i_cms
[4]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/cms.html#output_from_cms_collector
[5]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/cms.html#output_from_cms_collector
[6]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/toc.html