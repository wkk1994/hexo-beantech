---
title: JVM之GC日志分析与调优
date: 2021-03-12 23:04:47
subtitle: "JVM"
catalog: true
header-img: "/images/default-head6.jpg"
tags:
- JVM调优
- GC日志
- java基础
- java
- JVM
catagories:
- java基础
---

# GC日志分析与调优

[GC日志分析与调优](./GC日志分析与调优.png)

## 调优前的基础概念

### GC的概念名词

* Minor GC(新生代收集)：只是在新生代进行的垃圾回收，又称为Young GC。
* Major GC(老年代收集)：只是在老年代进行的垃圾回收，又称为Old GC，目前只有CMS收集器会有单独收集老年代的行为。需要注意的是"Major GC"说法有点混淆，有时候也“Full GC”的意识。
* Mixed GC(混合收集)：在整个新生代以及部分老年代进行的垃圾回收，目前只有G1收集器有这种行为。
* Full GC(整堆收集)：在整个堆上进行垃圾回收，包括堆、元数据区，Parallel GC收集器有这种行为。Full GC包括了Minor GC和Major GC。

### 吞吐量和响应时间的选择

* 吞吐量：用户代码时间 /（用户代码执行时间 + 垃圾回收时间）。
* 响应时间：STW时间越短，响应时间就越短。

调优之前需要确定是吞吐量优先还是响应时间优先。

对于科学计算、数据挖掘等，是吞吐量优先，一般选择PS+PO
对于网址类的应用，一般是响应时间优先，选择G1(1.8)或者ParNew + CMS。

### 什么是调优

* 1.根据需求进行JVM规划和预调优；
* 2.优化运行JVM运行环境；
* 3.解决JVM运行过程中产生的各种问题。

### Hotspot参数

* 标准： - 开头，所有的HotSpot都支持，通过`java -version`可以查看所有支持的标准参数。
* 非标准：-X 开头，特定版本HotSpot支持特定命令，通过`java -X`可以查看支持的所有非标准参数。
* 不稳定：-XX 开头，下个版本可能取消，通过`java -XX:+PrintFlagsFinal -version`可以查看传递给JVM的所有默认参数。

常用参数：

* `java -server -XX:+PrintCommandLineFlags -version`：打印出用户自定义参数和JVM设置的参数名称和值。
* `java -XX:+PrintFlagsFinal -version`：打印出JVM支持的所有默认参数和运行时的最终值。
* `java -XX:+PrintFlagsInitial -version`：打印出JVM支持的所有默认参数和初始值值。

### GC日志相关参数

* -XX:+PrintGC：打印GC日志不显示详情信息。等同于jdk9的-Xlog:gc:。
* -XX:+PrintGCDetails：打印GC日志详情。等同于jdk9的-Xlog:gc*。
* -Xloggc:gc.demo.log：指定GC日志输出的文件目录，JDK8开始可以使用%p,%t等占位符表示进程pid和启动时间戳，指定相对目录就在当前项目的根目录下或者指定绝对路径。如`-Xloggc:gc.demo.%p.log`。
* -XX:+UseGCLogFileRotation：启用GC日志循环。
* -XX:NumberOfGCLogFiles=5：设置GC日志循环的日志数量。
* -XX:GCLogFileSize=20M：设置每个GC日志的大小。
* -XX:PrintGCTimeStamps：每行日志前面显示一个时间戳（如0.310）表示JVM启动后经过的时间（单位秒）。
* -XX:+PrintGCDateStamps：每行日志前面显示一个时间戳，表示当前时间。
* -XX:+PrintHeapAtGC：查看GC前后的堆、方法区可用容量变化。等同于jdk9的-Xlog:gc+heap=debug:。
* -XX:+PrintGCApplicationConcurrentTime -XX:+PrintGCApplicationStoppedTime：查看GC过程中用户线程并发时间以及停顿的时间。等同于jdk9的-Xlog:safepoint。
* -XX:+PrintAdaptiveSizePolicy：查看收集器Ergonomics机制（自动设置堆空间各分代区域大小、收集目标等内容，从Parallel收 集器开始支持）自动调节的相关信息。等同于jdk9的-Xlog:gc+ergo*=trace。
* -XX:+PrintTenuringDistribution：查看熬过收集后剩余对象的年龄分布信息。等同于jdk9的-Xlog:gc+age=trace。

一般线上的日志参数命令：`-Xloggc:/opt/xxx/logs/xxx-xxx-gc-%t.log -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=xxxM -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCCause`

### GC常用参数

#### 通用参数

* -Xms4g -Xmx4g：最小堆、最大堆建议设置成一致。
* -Xss128k：栈空间大小，一般不需要设置，默认值是1m。
* -Xmn2g：年轻代大小，一般不需要设置。
* -XX:+UseTLAB：使用TLAB，默认打开。
* -XX:+PrintTLAB 打印TLAB的使用情况，默认关闭。
* -XX:TLABSize：设置TLAB大小，不建议设置
* -XX:+ResizeTLAB：自动调整TLAB的大小，默认开启。
* -XX:+DisableExplictGC：system.gc()不起作用。
* -XX:+PrintGC
* -XX:+PrintGCDetails
* -XX:+PrintHeapAtGC
* -XX:+PrintGCTimeStamps
* -XX:+PrintGCApplicationConcurrentTime：打印应用程序时间。
* -XX:+PrintGCApplicationStoppedTime：打印暂停时长。
* -XX:+PrintReferenceGC：记录回收了多少种不同引用类型的引用。
* -verbose:class：类加载详细过程。
* -XX:+PrintVMOptions
* -XX:+PrintFlagsFinal -XX:+PrintFlagsInitial
* -Xloggc:opt/log/gc.log
* -XX:MaxTenuringThreshold：升代年龄，最大值15。
* -XX:TargetSurvivorRatio=n：动态年龄计算时需要的比率。
* **-XX:+ParallelGCThreads**：并行收集器的线程数，不指定时系统默认计算。
  对于Parallel GC，G1 GC，CMS GC的默认的计算公式：
  如果CPU <= 8，ParallelGCThreads = cpus数；
  如果CPU > 8，ParallelGCThreads = 8 + (cpu - 8 ) * 5/8或ParallelGCThreads = 8 + (cpu - 8 ) * 5/16，这个由平台决定，一般是5/8。

[ParallelGCThreads计算方式](https://blog.csdn.net/BDX_Hadoop_Opt/article/details/38021209)

#### Parallel常用参数

* -XX:SurvivorRatio
* -XX:PretenureSizeThreshold=10000：指定超过这个大小的对象直接在老年代分配，默认值为0表示都先在eden分配。
* -XX:MaxTenuringThreshold
* -XX:TargetSurvivorRatio=n：动态年龄计算时需要的比率。
* -XX:+UseAdaptiveSizePolicy：自动选择各区大小比例，默认开启。

### 环境优化

#### 系统CPU经常100%，如何查找问题？

* 1.使用`top`查找出哪个进程的CPU占用高；
* 2.使用`top -Hp pid`查找进程中哪个线程占用CPU高；
* 3.对于java应用导出进程的堆栈信息，通过`jstack pid`命令，并在导出的堆栈信息中查找指定线程的信息：`jstack pid|grep thread_pid -A 100`，注意这里的thread_pid是上一步线程的16进制。

#### 系统内存经常飙高，如何查找问题？

* 1.使用`top`查找出哪个进程的内存占用高；
* 2.对于java应用导出进程堆内存，`jmap -heap pid`；

## GC日志分析

### Serial GC

启动参数:`java -XX:+UseSerialGC -Xms512m -Xmx512m -Xloggc:gc.demo.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps  com.wkk.demo.jvm.GCLogAnalysis`

* Minor GC

```text
2020-10-25T15:06:53.096-0800: 0.167: [GC (Allocation Failure) 2020-10-25T15:06:53.096-0800: 0.167: [DefNew: 139776K->17472K(157248K), 0.0264035 secs] 139776K->49474K(506816K), 0.0264814 secs] [Times: user=0.02 sys=0.01, real=0.03 secs]
```

`2020-10-25T15:06:53.096-0800`: 表示发生GC的时间戳。
`0.167`: 表示GC事件相对于JVM启动时间的时间间隔，单位秒。
`GC (Allocation Failure)`: `GC`用来区分这是Minor GC还是Major GC。`GC`表示这是Minor GC。`Allocation Failure`表示发生GC的原因，这里是分配对象失败。
`DefNew`: 表示垃圾收集器的名称。这个名称表示：年轻代使用的是单线程、标记-复制、STW垃圾收集器。
`139776K->17472K`: 表示在GC前后年轻代的使用量；`(157248K)`: 表示当前年轻代的总空间大小。
`139776K->49474K(506816K)`: 表示GC前后的堆内存使用量的大小以及可用堆空间大小。
`0.0264814 secs`: 表示GC事件持续的时间，以秒为单位。
`[Times: user=0.01 sys=0.01, real=0.03 secs]`: 表示此次GC事件的持续时间，`user`表示所有GC线程消耗CPU时间；`sys`表示系统调用和系统等待事件消耗的时间；`real`表示应用程序的暂停时间。因为串行垃圾收集器(Serial Garbage Collector)只使用单个线程， 所以这里 real = user + system ，0.03秒也就是30毫秒。

**`user`、`sys`和`real`在计算机中的含义：**

`user`: 进程中执行用户态代码所耗费的处理器时间。
`sys`: 进程中执行核心态代码所耗费的处理器时间。
`real`: 执行动作从开始到结束耗费的时钟时间。

`user`和`sys`是处理器时间，`real`是时钟时间。它们的区别是：处理器时间代表的是线程占用一个处理器一个核心的耗时计数，而时钟时间是现实世界中的时间计数。如果是单核单线程的场景下，这两者是等价的。

* Full GC

```text
2020-10-25T15:06:53.449-0800: 0.520: [GC (Allocation Failure) 2020-10-25T15:06:53.449-0800: 0.520: [DefNew: 157247K->157247K(157248K), 0.0000124 secs]2020-10-25T15:06:53.449-0800: 0.520: [Tenured: 338706K->277655K(349568K), 0.0390082 secs] 495954K->277655K(506816K), [Metaspace: 2740K->2740K(1056768K)], 0.0391023 secs] [Times: user=0.04 sys=0.00, real=0.04 secs]
```

`Tenured`: 用于清理老年代空间的垃圾收集器名称。Tenured 表明使用的是单线程的STW垃圾收集器，使用的算法为`标记‐清除‐整理(mark‐sweep‐compact)`。
`338706K->277655K(349568K)`: 表示GC前后老年的使用量以及老年代的大小。
`495954K->277655K(506816K)`: 表示GC前后的堆内存使用量的大小以及可用堆空间大小。
`[Metaspace: 2740K->2740K(1056768K)]`: Metaspace 空间的变化情况。一般GC前后没有什么变化。

对于每次GC后，我们需要关注的是GC后年轻代或者老年代的占用率，以及停顿时间。

### Parallel GC

启动命令: `java -XX:+UseParallelGC -Xms256m -Xmx256m -Xloggc:gc.demo.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps  com.wkk.demo.jvm.GCLogAnalysis`

其中`-XX:+UseParallelGC`，`-XX:+UseParallelOldGC`，`-XX:+UseParallelGC -XX:+UseParallelOldGC`三个参数等价。

* Minor GC

```text
2020-10-25T15:58:40.887-0800: 0.143: [GC (Allocation Failure) [PSYoungGen: 131584K->21503K(153088K)] 131584K->44170K(502784K), 0.0174490 secs] [Times: user=0.02 sys=0.11, real=0.02 secs]
```

`PSYoungGen`: 表示垃圾收集器的名称。这个名称表示是在年轻代中使用的并行的`标记-复制(mark-copy)`，全线暂停(STW)垃圾收集器。

`[Times: user=0.02 sys=0.11, real=0.02 secs]`: 因为在Parallel GC中，并不是所有的操作过程都是并行的，所以`real约等于user+system/GC线程数`。

* Full GC

```text
2020-10-25T15:58:41.172-0800: 0.428: [Full GC (Ergonomics) [PSYoungGen: 19582K->0K(116736K)] [ParOldGen: 326352K->251598K(349696K)] 345935K->251598K(466432K), [Metaspace: 2740K->2740K(1056768K)], 0.0280731 secs] [Times: user=0.20 sys=0.01, real=0.03 secs]
```

`Full GC`: Full GC标志；`Ergonomics`: 表示这次GC发生的原因是表示JVM内部环境认为此时可以进行一次垃圾收集。
`PSYoungGen: 19582K->0K(116736K)`: 表示垃圾收集器的名称。年轻代使用量从 14341K 变为 0 ，一般`Full GC`中年轻代的结果 都是这样。
`ParOldGen`: 用于清理老年代的垃圾收集器名称。ParOldGen是一个并行STW垃圾收集器，使用`标记-清楚-整理(mark-sweep-compact)`。`326352K->251598K(349696K)`: 在GC前后老年代内存使用情况以及老年代空间大小。
`345935K->251598K(466432K)`: 表示在GC前后堆内存的使用量和内存大小。

### CMS GC

启动命令: ` java -XX:+UseConcMarkSweepGC -Xms512m -Xmx512m -Xloggc:gc.demo.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps  com.wkk.demo.jvm.GCLogAnalysis`

* Minor GC

```text
2020-10-25T16:51:27.346-0800: 0.166: [GC (Allocation Failure) 2020-10-25T16:51:27.346-0800: 0.167: [ParNew: 139776K->17471K(157248K), 0.0199700 secs] 139776K->51922K(506816K), 0.0200949 secs] [Times: user=0.04 sys=0.13, real=0.02 secs]
```

`ParNew`: 表示用于清理老年代的垃圾收集器的名称。ParNew实质上是是Serial收集器的多线程并行版本，也是一个STW、标记复制的收集器。

`139776K->17471K(157248K)`: 回收前后年轻代的内存变化情况。
`139776K->51922K(506816K)`: 回收前后堆内存的变化情况。

* Major GC

```text
# 初始化标记
2020-10-25T19:32:44.824-0800: 0.537: [GC (CMS Initial Mark) [1 CMS-initial-mark: 282369K(349568K)] 300490K(506816K), 0.0001964 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
# 并发标记
2020-10-25T19:32:44.825-0800: 0.538: [CMS-concurrent-mark-start]
2020-10-25T19:32:44.825-0800: 0.538: [CMS-concurrent-mark: 0.001/0.001 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
# 并发预清理
2020-10-25T19:32:44.825-0800: 0.538: [CMS-concurrent-preclean-start]
2020-10-25T19:32:44.826-0800: 0.539: [CMS-concurrent-preclean: 0.000/0.000 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
2020-10-25T19:32:44.826-0800: 0.539: [CMS-concurrent-abortable-preclean-start]
# 这里又发生一次Minor GC
2020-10-25T19:32:44.842-0800: 0.555: [GC (Allocation Failure) 2020-10-25T19:32:44.842-0800: 0.555: [ParNew: 157246K->17471K(157248K), 0.0119928 secs] 439615K->347399K(506816K), 0.0120773 secs] [Times: user=0.12 sys=0.00, real=0.01 secs] 

2020-10-25T19:32:44.855-0800: 0.567: [CMS-concurrent-abortable-preclean: 0.001/0.029 secs] [Times: user=0.13 sys=0.00, real=0.03 secs] 
# 最终标记
2020-10-25T19:32:44.855-0800: 0.568: [GC (CMS Final Remark) [YG occupancy: 23151 K (157248 K)]2020-10-25T19:32:44.855-0800: 0.568: [Rescan (parallel) , 0.0002915 secs]2020-10-25T19:32:44.855-0800: 0.568: [weak refs processing, 0.0000132 secs]2020-10-25T19:32:44.855-0800: 0.568: [class unloading, 0.0001698 secs]2020-10-25T19:32:44.855-0800: 0.568: [scrub symbol table, 0.0002907 secs]2020-10-25T19:32:44.856-0800: 0.568: [scrub string table, 0.0001502 secs][1 CMS-remark: 329928K(349568K)] 353079K(506816K), 0.0009796 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
# 并发清除
2020-10-25T19:32:44.856-0800: 0.569: [CMS-concurrent-sweep-start]
2020-10-25T19:32:44.856-0800: 0.569: [CMS-concurrent-sweep: 0.000/0.000 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
# 并发重置
2020-10-25T19:32:44.856-0800: 0.569: [CMS-concurrent-reset-start]
2020-10-25T19:32:44.856-0800: 0.569: [CMS-concurrent-reset: 0.000/0.000 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 

```

`CMS Initial Mark`(初始化标记): 初始化标记需要STW，但是速度很快，`282369K(349568K)`表示当前老年代的内存使用情况；`300490K(506816K)`表示当前堆的内存使用情况；`0.0001964 secs`初始化标记用时。

`CMS-concurrent-mark`(并发标记): `0.001/0.001 secs`表示当前阶段的持续时间和时钟时间。

`CMS-concurrent-preclean`(并发预清理): 

`CMS Final Remark`(最终标记): `YG occupancy: 23151 K (157248 K)`表示当前年轻代的内存使用情况；`Rescan (parallel) , 0.0002915 secs`整个最终标记扫描对象的用时总计；`weak refs processing, 0.0000132 secs`对弱连接处理用时；`class unloading, 0.0001698 secs`卸载不用的类用时；`scrub symbol table, 0.0002907 secs`表示清理分别包含类级元数据和内部化字符串的符号和字符串表的耗时；`CMS-remark: 329928K(349568K)`表示经历过上面阶段后老年代的内存使用情况；`353079K(506816K), 0.0009796 secs`表示堆内存使用情况以及整个最终标记的用时。

`CMS-concurrent-sweep`(并发清除): `CMS-concurrent-sweep: 0.000/0.000 secs`表示并发清除的持续时间和时钟时间。

`CMS-concurrent-reset`(并发重置): `CMS-concurrent-reset: 0.000/0.000 secs`表示并发重置的持续时间和时钟时间。

* Full GC

```text
2021-03-11T14:39:13.823-0800: 10.599: [GC (Allocation Failure) 2021-03-11T14:39:13.823-0800: 10.599: [ParNew: 157079K->157079K(157248K), 0.0000189 secs]2021-03-11T14:39:13.823-0800: 10.599: [CMS2021-03-11T14:39:13.823-0800: 10.599: [CMS-concurrent-abortable-preclean: 0.002/0.044 secs] [Times: user=0.18 sys=0.00, real=0.05 secs] 
 (concurrent mode failure): 289662K->162991K(349568K), 0.0236826 secs] 446741K->162991K(506816K), [Metaspace: 3729K->3729K(1056768K)], 0.0237884 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
```

这个日志显示的就是并发标记失败（Concurrent Mode Failure），虚拟机临时使用Serial Old收集器来执行老年代的回收。

### G1 GC

启动命令：`java -XX:+UseG1GC -Xms512m -Xmx512m -Xloggc:gc.demo.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps  com.wkk.demo.jvm.GCLogAnalysis`

这里使用的命令是`-XX:+PrintGC`不显示详情，因为详情信息太多太难分析。

* Minor GC

```text
2020-11-07T17:16:57.129-0800: 0.129: [GC pause (G1 Evacuation Pause) (young), 0.0097386 secs]
   [Parallel Time: 8.3 ms, GC Workers: 10]
      [GC Worker Start (ms): Min: 128.9, Avg: 128.9, Max: 129.2, Diff: 0.3]
      [Ext Root Scanning (ms): Min: 0.0, Avg: 0.2, Max: 0.4, Diff: 0.4, Sum: 1.8]
      [Update RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
         [Processed Buffers: Min: 0, Avg: 0.0, Max: 0, Diff: 0, Sum: 0]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.2, Diff: 0.2, Sum: 0.2]
      [Object Copy (ms): Min: 7.2, Avg: 7.4, Max: 7.8, Diff: 0.5, Sum: 73.9]
      [Termination (ms): Min: 0.0, Avg: 0.4, Max: 0.6, Diff: 0.6, Sum: 4.4]
         [Termination Attempts: Min: 1, Avg: 1.0, Max: 1, Diff: 0, Sum: 10]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
      [GC Worker Total (ms): Min: 8.0, Avg: 8.0, Max: 8.2, Diff: 0.2, Sum: 80.5]
      [GC Worker End (ms): Min: 136.9, Avg: 137.0, Max: 137.1, Diff: 0.2]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.1 ms]
   [Other: 1.3 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 1.0 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.2 ms]
      [Humongous Register: 0.0 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.0 ms]
   [Eden: 25.0M(25.0M)->0.0B(21.0M) Survivors: 0.0B->4096.0K Heap: 25.0M(512.0M)->14.9M(512.0M)]
 [Times: user=0.03 sys=0.04, real=0.01 secs] 
```

`25.0M(512.0M)->14.9M(512.0M)`: 表示堆内存进行GC前后的空间使用情况。

* 并发标记（Concurrent Marking）

```text
# 初始标记
2020-11-07T17:16:57.469-0800: 0.468: [GC pause (G1 Evacuation Pause) (young) (initial-mark), 0.0083076 secs]
   [Parallel Time: 7.7 ms, GC Workers: 10]
      [GC Worker Start (ms): Min: 467.9, Avg: 468.0, Max: 468.1, Diff: 0.2]
      [Ext Root Scanning (ms): Min: 0.0, Avg: 0.2, Max: 0.4, Diff: 0.4, Sum: 2.4]
      [Update RS (ms): Min: 0.0, Avg: 0.1, Max: 0.2, Diff: 0.2, Sum: 0.8]
         [Processed Buffers: Min: 0, Avg: 1.1, Max: 2, Diff: 2, Sum: 11]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Object Copy (ms): Min: 7.0, Avg: 7.1, Max: 7.1, Diff: 0.1, Sum: 70.6]
      [Termination (ms): Min: 0.0, Avg: 0.1, Max: 0.1, Diff: 0.1, Sum: 0.9]
         [Termination Attempts: Min: 1, Avg: 1.0, Max: 1, Diff: 0, Sum: 10]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
      [GC Worker Total (ms): Min: 7.4, Avg: 7.5, Max: 7.6, Diff: 0.2, Sum: 75.0]
      [GC Worker End (ms): Min: 475.5, Avg: 475.5, Max: 475.5, Diff: 0.0]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.1 ms]
   [Other: 0.5 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 0.2 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.1 ms]
      [Humongous Register: 0.0 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.0 ms]
   [Eden: 83.0M(83.0M)->0.0B(43.0M) Survivors: 20.0M->13.0M Heap: 396.3M(512.0M)->384.8M(512.0M)]
 [Times: user=0.04 sys=0.01, real=0.01 secs] 
# Root区扫描
2020-11-07T17:16:57.477-0800: 0.476: [GC concurrent-root-region-scan-start]
2020-11-07T17:16:57.477-0800: 0.476: [GC concurrent-root-region-scan-end, 0.0001037 secs]
# 并发标记
2020-11-07T17:16:57.477-0800: 0.476: [GC concurrent-mark-start]
2020-11-07T17:16:57.478-0800: 0.478: [GC concurrent-mark-end, 0.0010957 secs]
# 最终标记
2020-11-07T17:16:57.478-0800: 0.478: [GC remark 2020-11-07T17:16:57.478-0800: 0.478: [Finalize Marking, 0.0002246 secs] 2020-11-07T17:16:57.479-0800: 0.478: [GC ref-proc, 0.0000554 secs] 2020-11-07T17:16:57.479-0800: 0.478: [Unloading, 0.0004953 secs], 0.0011637 secs]
 [Times: user=0.00 sys=0.00, real=0.00 secs] 
2020-11-07T17:16:57.480-0800: 0.479: [GC clearnup 394M->213M(512M), 0.0007590 secs]
 [Times: user=0.00 sys=0.00, real=0.00 secs] 
# 并发清除
2020-11-07T17:16:57.480-0800: 0.480: [GC concurrent-clearnup-start]
2020-11-07T17:16:57.481-0800: 0.480: [GC concurrent-clearnup-end, 0.0001027 secs]
```

精简的日志：

```text
# 初始标记
2020-10-25T20:34:33.285-0800: 0.314: [GC pause (G1 Humongous Allocation) (young) (initial-mark) 348M->222M(512M), 0.0211430 secs]
# Root区扫描
2020-10-25T20:34:33.306-0800: 0.336: [GC concurrent-root-region-scan-start]
2020-10-25T20:34:33.306-0800: 0.336: [GC concurrent-root-region-scan-end, 0.0001163 secs]
# 并发标记
2020-10-25T20:34:33.306-0800: 0.336: [GC concurrent-mark-start]
2020-10-25T20:34:33.308-0800: 0.337: [GC concurrent-mark-end, 0.0013900 secs]
# 最终标记
2020-10-25T20:34:33.308-0800: 0.337: [GC remark, 0.0010625 secs]
# 清除
2020-10-25T20:34:33.309-0800: 0.338: [GC cleanup 234M->234M(512M), 0.0005050 secs]
```

* Mixed GC

```text
2020-11-07T17:16:57.494-0800: 0.493: [GC pause (G1 Evacuation Pause) (mixed), 0.0047898 secs]
   [Parallel Time: 2.8 ms, GC Workers: 10]
      [GC Worker Start (ms): Min: 493.0, Avg: 493.3, Max: 493.8, Diff: 0.9]
      [Ext Root Scanning (ms): Min: 0.0, Avg: 0.1, Max: 0.2, Diff: 0.2, Sum: 0.7]
      [Update RS (ms): Min: 0.0, Avg: 0.0, Max: 0.1, Diff: 0.1, Sum: 0.3]
         [Processed Buffers: Min: 0, Avg: 1.0, Max: 3, Diff: 3, Sum: 10]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Object Copy (ms): Min: 1.2, Avg: 1.6, Max: 2.0, Diff: 0.8, Sum: 16.0]
      [Termination (ms): Min: 0.0, Avg: 0.6, Max: 0.9, Diff: 0.9, Sum: 6.1]
         [Termination Attempts: Min: 1, Avg: 1.0, Max: 1, Diff: 0, Sum: 10]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
      [GC Worker Total (ms): Min: 1.7, Avg: 2.3, Max: 2.8, Diff: 1.1, Sum: 23.3]
      [GC Worker End (ms): Min: 495.5, Avg: 495.6, Max: 495.7, Diff: 0.2]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.2 ms]
   [Other: 1.8 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 1.5 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.1 ms]
      [Humongous Register: 0.0 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.0 ms]
   [Eden: 18.0M(18.0M)->0.0B(21.0M) Survivors: 7168.0K->4096.0K Heap: 264.6M(512.0M)->214.5M(512.0M)]
 [Times: user=0.03 sys=0.01, real=0.00 secs] 
```

* Full GC

```text
2020-11-07T17:16:58.794-0800: 1.793: [Full GC (Allocation Failure)  512M->315M(512M), 0.0487123 secs]
   [Eden: 0.0B(25.0M)->0.0B(72.0M) Survivors: 0.0B->0.0B Heap: 512.0M(512.0M)->315.7M(512.0M)], [Metaspace: 3728K->3728K(1056768K)]
 [Times: user=0.05 sys=0.00, real=0.05 secs]
```

正常情况下应该有Full GC，Full GC会让虚拟机使用Serial Old收集器进行垃圾回收。

## JVM线程堆栈数据分析

思维导图：`JVM线程堆栈数据分析.png`

### 线程的实现

主流的操作系统都提供了线程实现，Java根据不同的操作系统和硬件平台对线程操作做了统一处理，每个已经调用了start()方法并且还没执行结束的java.lang.Thread类的实例就代表一个线程。

实现线程主要有三种方式：使用内核线程实现（1:1 实现），使用用户线程实现（1:N 实现），使用用户线程加轻量级进程混合实现（N:M 实现）。

### Java线程的实现

Java线程的实现不受Java虚拟机规范的约束。在线程主流的HotSpot虚拟机上使用内核线程实现，它的每一个java线程都是直接映射到一个操作系统原生线程来实现，而且中间没有额外的间接结构，所以HotSpot自己是不会去干涉线程调度的（可以设置线程优先级给操作系统提供调度建议），全权交给底下的操作系统去处理，所以何时冻结或唤醒线程、该给线程分配多少处理器执行时间、该把线程安排给哪个处理器核心去执行等，都是由操作系统完成的，也都是由操作系统全权决定的。

其他JVM虚拟机也提供了其他方式的实现。

### JVM线程模型示意图

![VM线程模型示意图](./JVM线程模型示意图.png)

### JVM线程分类

* VM线程：单例的VMThread对象，负责进行VM操作，主线程。
* 定时任务线程：单例的WatcherThread对象，模拟在VM中执行定时操作的计时器中断。
* GC线程：垃圾收集器中，用于支持并发和并行垃圾回收的线程。
* 编译器线程：用于将热点字节码编译为本地机代码。
* 信号分发线程：配合和协调JVM线程执行。

### JVM进行线程转储的方式

* JDK工具，包括jstack工具，jcmd工具，jsonsole，jvisualvm等
* Shell命令或者系统控制台，如Linux的 kill -3，window的Ctrl+Break等。kill -3输出的堆栈信息是在程序的日志中。
* JMX技术，主要是使用ThreadMxBean。

### JVM线程死锁

死锁可以通过jconsole，jstack查看或者通过[fastThread](https://fastthread.io)、[perfma](https://thread.console.perfma.com)网站上查看。

解决死锁的方式：

* 限定线程的执行时间，如果时间到了还没有执行完成视为发生死锁，直接终止。
* 对死锁环中的一个线程强制终止或者取消。

死锁示例：

jstack -l pid

```text
...
Java stack information for the threads listed above:
===================================================
"Thread-1":
        at com.wkk.demo.jvm.thread.DeadlockTest.testB(DeadlockTest.java:45)
        - waiting to lock <0x00000005c00051a0> (a java.lang.String)
        - locked <0x00000005c00051b8> (a java.lang.String)
        at com.wkk.demo.jvm.thread.DeadlockTest.lambda$main$1(DeadlockTest.java:19)
        at com.wkk.demo.jvm.thread.DeadlockTest$$Lambda$2/668386784.run(Unknown Source)
        at java.lang.Thread.run(Thread.java:748)
"Thread-0":
        at com.wkk.demo.jvm.thread.DeadlockTest.testA(DeadlockTest.java:32)
        - waiting to lock <0x00000005c00051b8> (a java.lang.String)
        - locked <0x00000005c00051a0> (a java.lang.String)
        at com.wkk.demo.jvm.thread.DeadlockTest.lambda$main$0(DeadlockTest.java:16)
        at com.wkk.demo.jvm.thread.DeadlockTest$$Lambda$1/764977973.run(Unknown Source)
        at java.lang.Thread.run(Thread.java:748)

Found 1 deadlock.
```

## JVM问题分析

### JVM问题产生的两个原因

* **分配速率(Allocation Rate)**
  分配率表示单位时间内分配的内存量，通常使用Mb/sec为单位。计算方式：上一次垃圾收集之后，与下一次GC开始之前的年轻代使用量，两者的差值除以时间。高分配速率(High Allocation Rate)：分配速率过高就会严重影响程序的性能，在 JVM 中可能会导致巨大的 GC 开销。

  * 正常系统：分配速率较低 ~ 回收速率 -> 健康
  * 内存泄漏：分配速率 持续大于 回收速率 -> OOM
  * 性能劣化：分配速率较高 ~ 回收速率 -> 压健康
  
  增大年轻代的大小只会降低高分配率产生的影响，不会降低分配率，但是会减少GC的频率。

* **提升速率(promotion rate)**
  过早提升(Premature Promotion)：对象的存活时间还不够长的时候就被提升到了老年代。这可能造成老年代进行GC时，需要清理大量无用的对象，会导致GC暂停时间过长，影响系统吞吐量。

  提升速率过高的表现：

  * 短时间内频繁地执行 full GC；
  * 每次 full GC 后老年代的使用率都很低，在10-20%或以下；
  * 提升速率接近于分配速率。

**如何解决上面的问题：**

* 修改代码，减少批处理的数量，降低分配速率。
* 增加年轻代大小，降低提升速率。

### 内存溢出分析

* OutOfMemoryError: Java heap space: 存放新对象时，堆内存的空间不足以存放创建的对象。
  * 超出预期的访问量/数据量：加大堆内存空间或者增加服务节点。
  * 内存泄露(Memory leak)：检查代码解决内存泄漏。

* OutOfMemoryError: PermGen space/OutOfMemoryError: Metaspace: 产生这个的主要原因是加载到内存中的class数量太多或者体积太大，超过了PermGen区的大小。
  * 增大PermGen/Metaspace的大小：-XX:MaxPermSize=512m -XX:MaxMetaspaceSize=512m（jdk8默认元空间内存是不限制的）。

* OutOfMemoryError: Unable to create new native thread: 程序创建的线程数量已经达到上限值。
  * 调整系统参数 ulimit -a，echo 120000 > /proc/sys/kernel/threads-max。
  * 通过减小Xss等参数，增加线程数量的限制。
  * 调整代码，改变线程创建和使用的方式，比如使用线程池。

### GC诊断工具

#### Arthas

Alibaba开源的Java诊断工具。在Docker环境中需要有jdk支持。

使用方式：下载arthas-demo.jar，再用java -jar命令启动：

```text
curl -O https://arthas.aliyun.com/arthas-boot.jar
java -jar arthas-boot.jar
```

常用命令：

* dashboard: 每隔一段时间显示当前进程的信息，包括线程、内存。
* thread: 只显示第一页的线程信息。
* thread --all: 显示全部线程信息。
* thread id: 查看指定线程的栈信息。
* thread -b: 查看当前阻塞其他线程的线程。**目前只支持找出synchronized关键字阻塞住的线程。**
* thread -i 1000: 统计最近1000ms内的线程CPU时间。
* thread -n 3 -i 1000: 列出1000ms内最忙的3个线程栈。
* thread --state WAITING: 查看指定状态的线程。
* jvm: 查看当前JVM信息。
* sysprop: 查看当前系统属性。
* sysenv: 查看当前JVM环境属性。
* vmoption: 查看所有VM诊断相关的参数。
* vmoption PrintGCDetails: 查看指定参数值。
* vmoption PrintGCDetails true: 更新执行参数值。
* classloader:显示classloader的信息。
* sc、sm、getstatic可以查看类新、类的方法信息、类的静态变量信息。
* jad: 反编译，可以用来动态代理生成类的问题定位；查看第三方的类；版本问题，确定最新的代码是否发布。
* redefine: 加载指定的.class文件，可以用来重新加载class文件，做到不停机发布。redefine的class不能修改、添加、删除类的field和method，包括方法参数、方法名称及返回值。现在推荐retransform。
* heapdump /tmp/dump.hprof: 将java heap信息导出，类似jmap命令的heap dump功能。
* heapdump --live /tmp/dump.hprof: 只导出存活对象。
* monitor、watch、trace、stack等支持对方法的执行进行监控。

#### Datadog

一个商业的监控应用。https://www.datadoghq.com/

### GC疑难情况问题分析方法

* 查询业务日志，可以发现这类问题：请求压力大，波峰，遭遇降级，熔断等等， 基础服务、外部 API 依赖 。

* 查看系统资源和监控信息：硬件信息、操作系统平台、系统架构；排查 CPU 负载、内存不足，磁盘使用量、硬件故障、磁盘分区用满、IO 等待、IO 密集、丢数据、并发竞争等情况；排查网络：流量打满，响应超时，无响应，DNS 问题，网络抖动，防火墙问题，物理故障，网络参数调整、超时、连接数。

* 查看性能指标，包括实时监控、历史数据。可以发现 假死，卡顿、响应变慢等现象；
  排查数据库， 并发连接数、慢查询、索引、磁盘空间使用量、内存使用量、网络带宽、死锁、TPS、查询数据量、redo日志、undo、 binlog 日志、代理、工具 BUG。
  可以考虑的优化包括： 集群、主备、只读实例、分片、分区；大数据，中间件，JVM 参数。

* 排查系统日志， 比如重启、崩溃、Kill 。

* APM(全链路监控)，比如发现有些链路请求变慢等等。

* 排查应用系统。
  排查配置文件: 启动参数配置、Spring 配置、JVM 监控参数、数据库参数、Log 参数、APM 配置、内存问题，比如是否存在内存泄漏，内存溢出、批处理导致的内存放大、GC 问题等等；
  GC 问题， 确定 GC 算法、确定 GC 的 KPI，GC 总耗时、GC 最大暂停时间、分析 GC 日志和监控指标： 内存分配速度，分代提升速度，内存使用率等数据。适当时修改内存配置；
  排查线程, 理解线程状态、并发线程数，线程 Dump，锁资源、锁等待，死锁；
  排查代码， 比如安全漏洞、低效代码、算法优化、存储优化、架构调整、重构、解决业务代码 BUG、第三方库、XSS、CORS、正则；
  单元测试： 覆盖率、边界值、Mock 测试、集成测试。

* 排除资源竞争、坏邻居效应。
* 疑难问题排查分析手段
  DUMP 线程\内存；抽样分析\调整代码、异步化、削峰填谷。

## 参考

* [JOL：分析Java对象的内存布局](https://zhuanlan.zhihu.com/p/151856103)
* [深入理解Java虚拟机 12.4Java与线程](深入理解Java虚拟机)
* [深入理解Java虚拟机 3.4.2安全点](深入理解Java虚拟机)
* [JVM 安全点介绍](https://zhuanlan.zhihu.com/p/55303309)

## 问题

* 为什么使用Serial GC，堆内存的最大值和设置的不一样？
* 使用Serial GC，为什么Minor GC中出现老年代的回收日志`2020-10-25T15:06:53.613-0800: 0.684: [GC (Allocation Failure) 2020-10-25T15:06:53.613-0800: 0.684: [DefNew: 157247K->157247K(157248K), 0.0000157 secs]2020-10-25T15:06:53.613-0800: 0.684: [Tenured: 330374K->313555K(349568K), 0.0367768 secs] 487622K->313555K(506816K), [Metaspace: 2740K->2740K(1056768K)], 0.0369009 secs] [Times: user=0.03 sys=0.00, real=0.04 secs]`？根据这个可以看出这是full gc，不是minor gc了。
* int[128][2] 实例占用3600字节。 而 int[256] 实例则只占用1040字节。里面的有效存储空间是一样的，3600 比起1040多了246%的额外开销。 实际是3072和1056。是我自己算错了。
* 生产上一般会设置GC日志的参数吗？
