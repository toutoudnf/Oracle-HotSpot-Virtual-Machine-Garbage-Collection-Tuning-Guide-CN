11 Other Considerations
============================================================

This section covers other situations that affect garbage collection. 

### Finalization and Weak, Soft, and Phantom References

Some applications interact with garbage collection by using finalization and weak, soft, or phantom references. These features can create performance artifacts at the Java programming language level. An example of this is relying on finalization to close file descriptors, which makes an external resource (descriptors) dependent on garbage collection promptness. Relying on garbage collection to manage resources other than memory is almost always a bad idea. 

The section [Related Documents][1] in the [Preface][2] includes an article that discusses in depth some of the pitfalls of finalization and techniques for avoiding them.

### Explicit Garbage Collection

Another way that applications can interact with garbage collection is by invoking full garbage collections explicitly by calling `System.gc()`. This can force a major collection to be done when it may not be necessary (for example, when a minor collection would suffice), and so in general should be avoided. The performance effect of explicit garbage collections can be measured by disabling them using the flag `-XX:+DisableExplicitGC`, which causes the VM to ignore calls to `System.gc()`.

One of the most commonly encountered uses of explicit garbage collection occurs with the distributed garbage collection (DGC) of Remote Method Invocation (RMI). Applications using RMI refer to objects in other virtual machines. Garbage cannot be collected in these distributed applications without occasionally invoking garbage collection of the local heap, so RMI forces full collections periodically. The frequency of these collections can be controlled with properties, as in the following example:

```
java -Dsun.rmi.dgc.client.gcInterval=3600000
    -Dsun.rmi.dgc.server.gcInterval=3600000 ...
```

This example specifies explicit garbage collection once per hour instead of the default rate of once per minute. However, this may also cause some objects to take much longer to be reclaimed. These properties can be set as high as `Long.MAX_VALUE` to make the time between explicit collections effectively infinite if there is no desire for an upper bound on the timeliness of DGC activity. 

### Soft References

Soft references are kept alive longer in the server virtual machine than in the client. The rate of clearing can be controlled with the command-line option `-XX:SoftRefLRUPolicyMSPerMB=``<N>`, which specifies the number of milliseconds (ms) a soft reference will be kept alive (once it is no longer strongly reachable) for each megabyte of free space in the heap. The default value is 1000 ms per megabyte, which means that a soft reference will survive (after the last strong reference to the object has been collected) for 1 second for each megabyte of free space in the heap. This is an approximate figure because soft references are cleared only during garbage collection, which may occur sporadically. 

### Class Metadata

Java classes have an internal representation within Java Hotspot VM and are referred to as class metadata. In previous releases of Java Hotspot VM, the class metadata was allocated in the so called permanent generation. In JDK 8, the permanent generation was removed and the class metadata is allocated in native memory. The amount of native memory that can be used for class metadata is by default unlimited. Use the option `MaxMetaspaceSize` to put an upper limit on the amount of native memory used for class metadata.

Java Hotspot VM explicitly manages the space used for metadata. Space is requested from the OS and then divided into chunks. A class loader allocates space for metadata from its chunks (a chunk is bound to a specific class loader). When classes are unloaded for a class loader, its chunks are recycled for reuse or returned to the OS. Metadata uses space allocated by `mmap`, not by `malloc`.

If `UseCompressedOops` is turned on and `UseCompressedClassesPointers` is used, then two logically different areas of native memory are used for class metadata. `UseCompressedClassPointers` uses a 32-bit offset to represent the class pointer in a 64-bit process as does `UseCompressedOops` for Java object references. A region is allocated for these compressed class pointers (the 32-bit offsets). The size of the region can be set with `CompressedClassSpaceSize` and is 1 gigabyte (GB) by default. The space for the compressed class pointers is reserved as space allocated by `mmap` at initialization and committed as needed. The `MaxMetaspaceSize` applies to the sum of the committed compressed class space and the space for the other class metadata.

Class metadata is deallocated when the corresponding Java class is unloaded. Java classes are unloaded as a result of garbage collection, and garbage collections may be induced in order to unload classes and deallocate class metadata. When the space committed for class metadata reaches a certain level (a high-water mark), a garbage collection is induced. After the garbage collection, the high-water mark may be raised or lowered depending on the amount of space freed from class metadata. The high-water mark would be raised so as not to induce another garbage collection too soon. The high-water mark is initially set to the value of the command-line option `MetaspaceSize`. It is raised or lowered based on the options `MaxMetaspaceFreeRatio` and `MinMetaspaceFreeRatio`. If the committed space available for class metadata as a percentage of the total committed space for class metadata is greater than `MaxMetaspaceFreeRatio`, then the high-water mark will be lowered. If it is less than `MinMetaspaceFreeRatio`, then the high-water mark will be raised.

Specify a higher value for the option `MetaspaceSize` to avoid early garbage collections induced for class metadata. The amount of class metadata allocated for an application is application-dependent and general guidelines do not exist for the selection of `MetaspaceSize`. The default size of `MetaspaceSize` is platform-dependent and ranges from 12 MB to about 20 MB. 

Information about the space used for metadata is included in a printout of the heap. A typical output is shown in [Example 11-1, "Typical Heap Printout"][3].

 Example 11-1 Typical Heap Printout

```
Heap
  PSYoungGen      total 10752K, used 4419K
    [0xffffffff6ac00000, 0xffffffff6b800000, 0xffffffff6b800000)
    eden space 9216K, 47% used
      [0xffffffff6ac00000,0xffffffff6b050d68,0xffffffff6b500000)
    from space 1536K, 0% used
      [0xffffffff6b680000,0xffffffff6b680000,0xffffffff6b800000)
    to   space 1536K, 0% used
      [0xffffffff6b500000,0xffffffff6b500000,0xffffffff6b680000)
  ParOldGen       total 20480K, used 20011K
      [0xffffffff69800000, 0xffffffff6ac00000, 0xffffffff6ac00000)
    object space 20480K, 97% used 
      [0xffffffff69800000,0xffffffff6ab8add8,0xffffffff6ac00000)
  Metaspace       used 2425K, capacity 4498K, committed 4864K, reserved 1056768K
    class space   used 262K, capacity 386K, committed 512K, reserved 1048576K
```

In the line beginning with `Metaspace`, the `used` value is the amount of space used for loaded classes. The `capacity` value is the space available for metadata in currently allocated chunks. The `committed` value is the amount of space available for chunks. The `reserved` value is the amount of space reserved (but not necessarily committed) for metadata. The line beginning with `class space` line contains the corresponding values for the metadata for compressed class pointers.

[1]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/preface.html#gct_related
[2]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/preface.html#CHDBHEGA
[3]:https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/considerations.html#typical_heap_printout