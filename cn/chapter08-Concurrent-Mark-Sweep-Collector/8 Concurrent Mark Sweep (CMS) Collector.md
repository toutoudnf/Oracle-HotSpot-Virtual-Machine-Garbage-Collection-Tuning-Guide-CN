8 Concurrent Mark Sweep (CMS) Collector
============================================================

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

### Concurrent Mode Failure

> 这个标题不准备翻译，解释下意思吧：Concurrent 模式下的一种失败场景。

前面解释过，CMS 为了减少 application 可感知的 GC 导致的暂停，某些阶段是允许 GC 过程与应用同时运行的。但是问题就来了，GC 标记全部的垃圾 & 回收是需要时间的，并且你也保不齐坑爹应用突然快速的大量的给你整一堆对象。这样在 CMS major GC 的过程当中，就可能出现 tuned generation 不够使的情况。application 就不干了啊给 JVM 叫冤，说你看那老些垃圾占地方都没我的地儿了。然后 JVM 没辙，停掉正在进行的 CMS major GC，改用全程 STW 的一次 fullgc 来完成垃圾的回收。这个情况在 CMS major GC 的过程当中就叫做 _concurrent mode failure_ 。这个时候你就需要调整 CMS 相关参数来解决这个问题了。

除了上述情况，显示的 gc 调用（`System.gc()`）或者一些性能诊断工具触发的 gc，也会触发这个 mode。

### GC 时间过长导致的内存溢出错误（OutOfMemoryError）

针对 CMS GC 的过程，还有一种情况也是需要避免的：tuned generation 比较小的情况下，CMS major gc 频繁的被触发，但是并没有什么新的垃圾产生。这种情况导致的结果就是大量时间+资源花费在了 GC 上，影响了 application 运行的效率。针对这种情况，CMS 做了优化：假设 CMS collector 发现花费了 98% 的时间在 GC 上（包括并发的时间），但是每次 GC 效果连总 heap 的 2% 有效空间都回收不到，这时候就会抛出 `OutOfMemoryError`。这个特性是可以通过 JVM 参数 `-XX:-UseGCOverheadLimit` 关闭的。

The policy is the same as that in the parallel collector, except that time spent performing concurrent collections is not counted toward the 98% time limit. In other words, only collections performed while the application is stopped count toward excessive GC time. Such collections are typically due to a concurrent mode failure or an explicit collection request (for example, a call to `System.gc`).

待补充

### 浮动垃圾

这段会讲浮动垃圾（floating garbage）是怎么来的：floating garbage 是指在当前的 CMS 垃圾收集阶段，某些一开始不是垃圾的对象，在 CMS 与 application 并发的阶段，变成了垃圾。然后这些对象在 heap 中又没有被收集，故称之为浮动垃圾。在 [Garbage Collection: Algorithms for Automated Dynamic Memory](https://book.douban.com/subject/2135376/) 这本书当中（译者注：神书，深入研究需细看），作者把 CMS 收集器称之为增量收集器（incremental update collector）。另外根据实际经验，针对 floating garbage，你需要预留 20% 的 tenured generation 来保证不会发生内存溢出啊啥的。

### GC 暂停

对于一个完成的 CMS 回收周期来讲，共有两次暂停，分别是：

- initial mark pause：初始标记暂停
- remark pause：重新标记暂停

两次标记都是找 live object，其中初始标记的出发点（root）包括：线程栈啊（application thread stacks）、registers、静态对象（static objects）等等，另外还有来自 heap 中非 tenured gen 部分的引用（如 young generation）；而重新标记是为了找到那些在并发过程中遗漏的 lived object。

### 并发阶段

并发阶段也有两个：初始标记和重新标记之间的阶段，以及重新标记后，垃圾的并发清理阶段。并发阶段会采用多个线程处理 GC 相关的事务，所以会占用处理器资源。这时候计算密集型的应用可能受影响会比较明显。当 CMS 垃圾收集周期结束之后，CMS gc thread 就会处于 waiting 的状态，基本不消耗 cpu 资源。

### 什么时候开始一个收集周期

对于非并发的那种垃圾收集器，开始的时间点很好选择：tenured generation 满了就开始收集。但是对于 CMS 这种，收集过程中存在并发阶段的收集器，需要选择一个合理的开始时间，否则收集着收集着 tenured generation 满了，就会切换到 full gc 模式，gc 的时间反而要比要来久了（就是前面提到的 Concurrent Mode Failure）。

CMS 收集器会根据历史数据分析 tenured generation 部分什么时候会填满，然后尽量保证收集周期之内，不会出现 tenured generation 满了的情况，从而避免 full gc（concurrent mode failure）。

如果当 tenured generation 使用率到达了一个百分比（initiating occupancy），CMS 收集周期也会开始。这个值初始大约为 92%，每个版本可能不同。这个值可以通过 `-XX:CMSInitiatingOccupancyFraction=<N>` 来指定，其中 N 是个 0 - 100 的整数，表示的是 tenured generation 的比率。

### 暂停时间的优化

对于 CMS 垃圾收集来说，young gen & tenured gen 的垃圾收集是分开且互相独立的的。这样可能会出现一种情况是，young gen 的垃圾收集（STW）跟 CMS 的垃圾收集的暂停（initial or remark）隔得太近，导致应用感知到的整体暂停时间过长。所以 CMS 会针对自己的 tenured gen 垃圾收集做优化，尽量保证 remark 阶段处于两次 young gen gc 中间（至于init mark 因为通长时间比较短，所以 CMS 不考虑对他的优化）。

### 增量模式（Incremental Mode）

要注意这个模式在 Java SE 8 已经不建议使用了，并且可能会在新的版本中移除。

CMS 并发收集阶段会抢占 cpu 资源，对 cpu 比较少的机器是不友好的（cpu 在 1 - 2 这样的）；并且可能因为并发收集周期较长，导致一个 CMS gc 周期中发生多个 young gc。为了解决这类问题，提供了一个 _i-cms_ 模式，即增量收集模式。增量的原理其实就是将并发标记阶段切分为多个子阶段进行，在每个子阶段结束后会将 CPU 资源交还给 application 去使用。每个子阶段尽量都能在两次 ygc 之间完成。

典型的并发垃圾收集如下：

*   停止所用应用线程，确认 root 部分的可达对象都有哪些（啥是 root 呢？就是线程栈啊类全局静态变量啊什么的），然后恢复应用程序执行

*   根据上面的 root 对象，并发的找到哪些关联对象时可达的。采用1个或多个处理器（concurrently 指的是与应用并发，采用的 processors 则表示 CMS 当前步骤是否是并发的）

*   并发阶段，根据被修改过关联关系的对象集合，找到可达对象，使用一个处理器

*   再次停止全部的应用线程，重新标记下可达的 root 对象，然后恢复全部应用线程

*   并发的清理不可达对象（清理是指放到 freelist 中），使用一个处理器

*   动态调整堆大小并且为下次垃圾回收准备相应的数据

正常情况下，CMS 垃圾回收过程中，使用一个或多个 cpu 执行的时候，是不会主动放弃 cpu 的使用权的。对于某些要求响应时间比较严格的 application 来说，是有影响的。尤其当所在的机器 cpu 比较少的情况（1个或者两个2啊）。增量收集（Incremental mode）就是解决这个问题的。他将并发回收阶段拆分为多个子过程，每个子过程在尽量在两次 minor gc 之间完成。

i-cms 模式下，定义了一个叫 duty cycle 的玩意儿，当每个 duty cycle 执行完成， CMS 垃圾收集器就尝试主动放弃 cpu 资源。duty cycle 的实际含义就是，在每两次 minor gc 之间的时间间隔当中，CMS 并发收集能使用百分之多少的时间。这个时间 CMS 可以自动调整（也是推荐的方式，被称之为 automatic pacing），或者可以通过参数，指定一个固定的百分比。

### JVM 命令行参数说明

表 8-1 i-cms 参数说明

| 参数名称 | 描述 | Java SE5 或之前的默认值 |Java SE6 或者之后的默认值 |
| --- | --- | --- | --- |
| `-XX:+CMSIncrementalMode`| 开启 i-cms 模式，前提是 CMS 必须同时开启 (使用 `-XX:+UseConcMarkSweepGC`) | disabled | disabled |
| `-XX:+CMSIncrementalPacing` | 开启 i-cms 步长的动态调整（CMS根据历史数据，动态调整 minor gc 时间间隔占比） | disabled | disabled |
| `-XX:CMSIncrementalDutyCycle=<N>` | CMS 并发阶段，在两次 minor gc 间，能使用百分之多少的时间. 如果 `CMSIncrementalPacing ` 开启了，那这个值就是个初始值. | 50 | 10 |
| `-XX:CMSIncrementalDutyCycleMin=<N>` | `CMSIncrementalPacing`开启后，自动调整的底线 | 10 | 0|
| `-XX:CMSIncrementalSafetyFactor=<N>` | 计算 duty cycle 的时候，为了安全起见，一个会加到计算结果上的兜底值 | 10 | 10|
| `-XX:CMSIncrementalOffset=<N>` | The percentage (0 to 100) by which the incremental mode duty cycle is shifted to the right within the period between minor collections. | 0 | 0|
| `-XX:CMSExpAvgFactor=<N>` | The percentage (0 to 100) used to weight the current sample when computing exponential averages for the CMS collection statistics. | 25 | 25|

### 建议参数配置

Java SE 8 中，使用 i-cms 的建议参数配置如下：

```
-XX:+UseConcMarkSweepGC -XX:+CMSIncrementalMode \
-XX:+PrintGCDetails -XX:+PrintGCTimeStamps
```

前两个参数分别是为了开启 CMS 垃圾收集器 & i-cms 模式。后两个不是必需的，是垃圾收集的细节日志，方便一些问题的分析和定位。

对于 Java SE 5 以及更早的版本，Oracle 推荐使用下面的参数来开启 i-cms：

```
-XX:+UseConcMarkSweepGC -XX:+CMSIncrementalMode \
-XX:+PrintGCDetails -XX:+PrintGCTimeStamps \
-XX:+CMSIncrementalPacing -XX:CMSIncrementalDutyCycleMin=0
-XX:CMSIncrementalDutyCycle=10
```

上述参数同样适用于 JavaSE8，只不过最后三个参数从 JavaSE6 开始就是默认的了。

### 基本调试

i-cms 是根据程序运行时所收集的静态信息来调整 gc，保证不会出现 heap full 的情况。但是程序过去的行为参考价值有限，不一定能保证不会出现变化。如果出现 full gc 的次数比较多，可以试着按照 [表 8-2 调试 i-cms 模式][3] 如下步骤来一步一步调试。一次尝试一步就好了

表 8-2 调试 i-cms 模式

| 操作 | 参数 |
| --- | --- |
| 1\. 指定你认为安全的 gc 开始执行的临界点. | `-XX:CMSIncrementalSafetyFactor=<N>` |
| 2\. 增加回收拆分的次数. | `-XX:CMSIncrementalDutyCycleMin=<N>` |
| 3\. 禁用自动的回收次数，指定一个固定值. | `-XX:-CMSIncrementalPacing -XX:CMSIncrementalDutyCycle=<N>` |

### 如何度量 CMS gc 收集性能

[例 8-1，CMS gc 日志输出][4] 是 CMS 垃圾收集的日志，并且是指定了参数 `-verbose:gc` 和 `-XX:+PrintGCDetails` 的。日志中一些 minor gc 的细节被忽略了。在日志中你可以看到，CMS 的并发期间，是包含 minor gc 的；CMS-initial-mark 表示 CMS 收集周期的开始，CMS-concurrent-mark 表示并发标记阶段的结束，CMS-concurrent-sweep 表示并发清理阶段的结束，CMS-concurrent-preclean 在前面没提到，这也是个并发执行的阶段，目的是为了后面的 CMS-remark 阶段做准备。CMS-concurrent-reset 则表示 CMS 垃圾收集结束，并重置一些参数之类的，为下一次垃圾并发收集做准备。

例 8-1，CMS gc 日志输出

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

初始标记的暂停通常很短，跟 minor gc 的时间差不离。并发阶段（concurrent mark, concurrent preclean and concurrent sweep）可能比 minor gc 的时间要长一些，但是因为是并发，所以应用程序仍然可以执行。重新标记阶段比较费时间，具体跟应用的特点有关系，比如应用的对象增长率很高啊，或者离上一次 minor gc 的时间很长啊（两者导致的结果都是 young generation 里面的对象比较多）。

[1]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/cms.html#i_cms_command_line_options
[2]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/cms.html#i_cms_recommended_options
[3]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/cms.html#troubleshooting_i_cms
[4]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/cms.html#output_from_cms_collector
[6]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/toc.html