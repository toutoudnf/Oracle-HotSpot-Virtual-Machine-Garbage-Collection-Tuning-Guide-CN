4 Sizing the Generations
============================================================

A number of parameters affect generation size. [Figure 4-1, "Heap Parameters"][1] illustrates the difference between committed space and virtual space in the heap. At initialization of the virtual machine, the entire space for the heap is reserved. The size of the space reserved can be specified with the -Xmx option. If the value of the -Xms parameter is smaller than the value of the -Xmx parameter, than not all of the space that is reserved is immediately committed to the virtual machine. The uncommitted space is labeled "virtual" in this figure. The different parts of the heap (tenured generation and young generation) can grow to the limit of the virtual space as needed.

Some of the parameters are ratios of one part of the heap to another. For example the parameter NewRatio denotes the relative size of the tenured generation to the young generation.

 **_Figure 4-1 Heap Parameters_** 

![Description of Figure 4-1 follows](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/img/jsgct_dt_006_prm_gn_sz.png) [Description of "Figure 4-1 Heap Parameters"][9]

### Total Heap

The following discussion regarding growing and shrinking of the heap and default heap sizes does not apply to the parallel collector. (See the section [Parallel Collector Ergonomics][2] in [Sizing the Generations][3] for details on heap resizing and default heap sizes with the parallel collector.) However, the parameters that control the total size of the heap and the sizes of the generations do apply to the parallel collector.

The most important factor affecting garbage collection performance is total available memory. Because collections occur when generations fill up, throughput is inversely proportional to the amount of memory available.

By default, the virtual machine grows or shrinks the heap at each collection to try to keep the proportion of free space to live objects at each collection within a specific range. This target range is set as a percentage by the parameters -XX:MinHeapFreeRatio=<minimum> and -XX:MaxHeapFreeRatio=<maximum>, and the total size is bounded below by -Xms<min> and above by -Xmx<max>. The default parameters for the 64-bit Solaris operating system (SPARC Platform Edition) are shown in [Table 4-1, "Default Parameters for 64-Bit Solaris Operating System"][4]:

 Table 4-1 Default Parameters for 64-Bit Solaris Operating System

| Parameter | Default Value |
| --- | --- |
| `MinHeapFreeRatio` | `40` |
| `MaxHeapFreeRatio` | `70` |
| `-Xms` | `6656k` |
| `-Xmx` | `calculated` |  

With these parameters, if the percent of free space in a generation falls below 40%, then the generation will be expanded to maintain 40% free space, up to the maximum allowed size of the generation. Similarly, if the free space exceeds 70%, then the generation will be contracted so that only 70% of the space is free, subject to the minimum size of the generation. 

As noted in [Table 4-1, "Default Parameters for 64-Bit Solaris Operating System"][5], the default maximum heap size is a value that is calculated by the JVM. The calculation used in Java SE for the parallel collector and the server JVM are now used for all the garbage collectors. Part of the calculation is an upper limit on the maximum heap size that is different for 32-bit platforms and 64-bit platforms. See the section [Default Heap Size][6] in [The Parallel Collector][7]. There is a similar calculation for the client JVM, which results in smaller maximum heap sizes than for the server JVM.

The following are general guidelines regarding heap sizes for server applications:

*   Unless you have problems with pauses, try granting as much memory as possible to the virtual machine. The default size is often too small.

*   Setting -Xms and -Xmx to the same value increases predictability by removing the most important sizing decision from the virtual machine. However, the virtual machine is then unable to compensate if you make a poor choice.

*   In general, increase the memory as you increase the number of processors, since allocation can be parallelized.

### The Young Generation

After total available memory, the second most influential factor affecting garbage collection performance is the proportion of the heap dedicated to the young generation. The bigger the young generation, the less often minor collections occur. However, for a bounded heap size, a larger young generation implies a smaller tenured generation, which will increase the frequency of major collections. The optimal choice depends on the lifetime distribution of the objects allocated by the application.

By default, the young generation size is controlled by the parameter `NewRatio`. For example, setting `-XX:NewRatio=3` means that the ratio between the young and tenured generation is 1:3\. In other words, the combined size of the eden and survivor spaces will be one-fourth of the total heap size.

The parameters `NewSize` and `MaxNewSize` bound the young generation size from below and above. Setting these to the same value fixes the young generation, just as setting `-Xms` and `-Xmx` to the same value fixes the total heap size. This is useful for tuning the young generation at a finer granularity than the integral multiples allowed by `NewRatio`. 

### Survivor Space Sizing

You can use the parameter `SurvivorRatio` can be used to tune the size of the survivor spaces, but this is often not important for performance. For example, `-XX:SurvivorRatio=6` sets the ratio between eden and a survivor space to 1:6\. In other words, each survivor space will be one-sixth the size of eden, and thus one-eighth the size of the young generation (not one-seventh, because there are two survivor spaces).

If survivor spaces are too small, copying collection overflows directly into the tenured generation. If survivor spaces are too large, they will be uselessly empty. At each garbage collection, the virtual machine chooses a threshold number, which is the number times an object can be copied before it is tenured. This threshold is chosen to keep the survivors half full. The command line option `-XX:+PrintTenuringDistribution` (not available on all garbage collectors) can be used to show this threshold and the ages of objects in the new generation. It is also useful for observing the lifetime distribution of an application. 

[Table 4-2, "Default Parameter Values for Survivor Space Sizing"][8] provides the default values for 64-bit Solaris:

 Table 4-2 Default Parameter Values for Survivor Space Sizing

| Parameter | Server JVM Default Value |
| --- | --- |
| `NewRatio` | `2` |
| `NewSize` | `1310M` |
| `MaxNewSize` | not limited |
| `SurvivorRatio` | `8` |  

The maximum size of the young generation will be calculated from the maximum size of the total heap and the value of the `NewRatio` parameter. The "not limited" default value for the `MaxNewSize` parameter means that the calculated value is not limited by `MaxNewSize` unless a value for `MaxNewSize` is specified on the command line.

The following are general guidelines for server applications:

*   First decide the maximum heap size you can afford to give the virtual machine. Then plot your performance metric against young generation sizes to find the best setting.

    *   Note that the maximum heap size should always be smaller than the amount of memory installed on the machine to avoid excessive page faults and thrashing.

*   If the total heap size is fixed, then increasing the young generation size requires reducing the tenured generation size. Keep the tenured generation large enough to hold all the live data used by the application at any given time, plus some amount of slack space (10 to 20% or more).

*   Subject to the previously stated constraint on the tenured generation:

    *   Grant plenty of memory to the young generation.

    *   Increase the young generation size as you increase the number of processors, because allocation can be parallelized.

[1]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/sizing.html#heap_parameters
[2]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/parallel.html#parallel_collector_ergonomics
[3]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/sizing.html#sizing_generations
[4]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/sizing.html#default_param_solaris
[5]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/sizing.html#default_param_solaris
[6]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/parallel.html#default_heap_size
[7]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/parallel.html#CHDCFBIF
[8]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/sizing.html#defaults_survivor_space
[9]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/img_text/jsgct_dt_006_prm_gn_sz.html