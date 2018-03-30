2 Ergonomics
============================================================

 _Ergonomics_  is the process by which the Java Virtual Machine (JVM) and garbage collection tuning, such as behavior-based tuning, improve application performance. The JVM provides platform-dependent default selections for the garbage collector, heap size, and runtime compiler. These selections match the needs of different types of applications while requiring less command-line tuning. In addition, behavior-based tuning dynamically tunes the sizes of the heap to meet a specified behavior of the application.

This section describes these default selections and behavior-based tuning. Use these defaults first before using the more detailed controls described in subsequent sections.

### Garbage Collector, Heap, and Runtime Compiler Default Selections

A class of machine referred to as a server-class machine has been defined as a machine with the following:

*   2 or more physical processors

*   2 or more GB of physical memory

On server-class machines, the following are selected by default:

*   Throughput garbage collector

*   Initial heap size of 1/64 of physical memory up to 1 GB

*   Maximum heap size of 1/4 of physical memory up to 1 GB

*   Server runtime compiler

For initial heap and maximum heap sizes for 64-bit systems, see the section [Default Heap Size][7] in [The Parallel Collector][8].

The definition of a server-class machine applies to all platforms with the exception of 32-bit platforms running a version of the Windows operating system. [Table 2-1, "Default Runtime Compiler"][9], shows the choices made for the runtime compiler for different platforms.

Table 2-1 Default Runtime Compiler

| Platform | Operating System | Default[Foot1][5] | Default if Server-Class[Footref1][6] |
| --- | --- | --- | --- |
| i586 | Linux | Client | Server |
| i586 | Windows | Client | Client[Foot2][1] |
| SPARC (64-bit) | Solaris | Server | Server[Foot3][2] |
| AMD (64-bit) | Linux | Server | Server[Footref3][3] |
| AMD (64-bit) | Windows | Server | Server[Footref3][4] |

Footnote1Client means the client runtime compiler is used. Server means the server runtime compiler is used.

Footnote2The policy was chosen to use the client runtime compiler even on a server class machine. This choice was made because historically client applications (for example, interactive applications) were run more often on this combination of platform and operating system.

Footnote3Only the server runtime compiler is supported.

### Behavior-Based Tuning

For the parallel collector, Java SE provides two garbage collection tuning parameters that are based on achieving a specified behavior of the application: maximum pause time goal and application throughput goal; see the section [The Parallel Collector][10]. (These two options are not available in the other collectors.) Note that these behaviors cannot always be met. The application requires a heap large enough to at least hold all of the live data. In addition, a minimum heap size may preclude reaching these desired goals.

### Maximum Pause Time Goal

The pause time is the duration during which the garbage collector stops the application and recovers space that is no longer in use. The intent of the maximum pause time goal is to limit the longest of these pauses. An average time for pauses and a variance on that average is maintained by the garbage collector. The average is taken from the start of the execution but is weighted so that more recent pauses count more heavily. If the average plus the variance of the pause times is greater than the maximum pause time goal, then the garbage collector considers that the goal is not being met.

The maximum pause time goal is specified with the command-line option `-XX:MaxGCPauseMillis=``<nnn>`. This is interpreted as a hint to the garbage collector that pause times of `<nnn>` milliseconds or less are desired. The garbage collector will adjust the Java heap size and other parameters related to garbage collection in an attempt to keep garbage collection pauses shorter than `<nnn>` milliseconds. By default there is no maximum pause time goal. These adjustments may cause garbage collector to occur more frequently, reducing the overall throughput of the application. The garbage collector tries to meet any pause time goal before the throughput goal. In some cases, though, the desired pause time goal cannot be met.

### Throughput Goal

The throughput goal is measured in terms of the time spent collecting garbage and the time spent outside of garbage collection (referred to as  _application time_ ). The goal is specified by the command-line option `-XX:GCTimeRatio=``<nnn>`. The ratio of garbage collection time to application time is 1 / (1 + `<nnn>`). For example, `-XX:GCTimeRatio=19` sets a goal of 1/20th or 5% of the total time for garbage collection.

The time spent in garbage collection is the total time for both the young generation and old generation collections combined. If the throughput goal is not being met, then the sizes of the generations are increased in an effort to increase the time that the application can run between collections.

### Footprint Goal

If the throughput and maximum pause time goals have been met, then the garbage collector reduces the size of the heap until one of the goals (invariably the throughput goal) cannot be met. The goal that is not being met is then addressed.

### Tuning Strategy

Do not choose a maximum value for the heap unless you know that you need a heap greater than the default maximum heap size. Choose a throughput goal that is sufficient for your application.

The heap will grow or shrink to a size that will support the chosen throughput goal. A change in the application's behavior can cause the heap to grow or shrink. For example, if the application starts allocating at a higher rate, the heap will grow to maintain the same throughput.

If the heap grows to its maximum size and the throughput goal is not being met, the maximum heap size is too small for the throughput goal. Set the maximum heap size to a value that is close to the total physical memory on the platform but which does not cause swapping of the application. Execute the application again. If the throughput goal is still not met, then the goal for the application time is too high for the available memory on the platform.

If the throughput goal can be met, but there are pauses that are too long, then select a maximum pause time goal. Choosing a maximum pause time goal may mean that your throughput goal will not be met, so choose values that are an acceptable compromise for the application.

It is typical that the size of the heap will oscillate as the garbage collector tries to satisfy competing goals. This is true even if the application has reached a steady state. The pressure to achieve a throughput goal (which may require a larger heap) competes with the goals for a maximum pause time and a minimum footprint (which both may require a small heap).

[1]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/ergonomics.html#sthref8
[2]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/ergonomics.html#BABIFJCI
[3]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/ergonomics.html#sthref9
[4]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/ergonomics.html#sthref10
[5]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/ergonomics.html#BABFAAII
[6]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/ergonomics.html#sthref7
[7]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/parallel.html#default_heap_size
[8]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/parallel.html#CHDCFBIF
[9]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/ergonomics.html#BABIIFHA
[10]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/parallel.html#CHDCFBIF