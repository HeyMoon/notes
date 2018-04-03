每一个区域的执行顺序是这样的：

1、大部分新创建的对象都处于 Eden 区
2、在 Eden 区域执行第一次 GC 以后，存活下来的对象被移动到其中一个存活区。
3、在 Eden 区域再次执行 GC 以后，存活下来的对象继续堆积已经有对象的那个存活区。
4、一旦一个存活区被存满，存活对象就会被移动到另一个存活区。然后被存满的那一个存活区数据就会被清掉（修改为无数据状态）。
5、如此反复一定次数之后，还处于存活状态的对象被移动到老年区。
如果你仔细检查这些步骤，存活区域总是有一个是空的。如果两个存活区域同时都有数据，或者同时都为空，这意味着你的系统存在问题。

针对HotSpot VM的实现，它里面的GC其实准确分类是由两大类：

1. Partial GC:并不收集整个GC堆的模式
   + Young GC:只收集Young gen的GC
   + Old GC:只收集old gen的GC。只有CMS的Concurrent Collection是这个模式
   + Mixed GC:收集整个Young GC以及部分的Old gen的GC。只有G1有这个模式

2. Full GC:收集整个堆，包括young gen,old gen,perm gen等所有部分的模式

Major GC 和Full GC是等价的，收集整个GC 堆。但因为HotSpot VM发展了这么多年，外界对各种名词的解读已经完全混乱了，当有人说“major GC”的时候一定要问清楚他想要指的是上面的full GC还是old GC。

+ Serial GC算法:Serial（Young） + Serial Old（Full GC）

+ CMS GC算法：ParNew（Young） + CMS（Old）+ （Concurrent mode Failure 时，启用Serial Old for CMS,Full GC)

+ Parallel GC算法：Parallel Scavenge（young） + Serial Old/Parallel Old（Full GC）

+ G1 GC算法：Young GC + Mixed GC + Full GC for G1（待验证）

## 各类GC算法的触发

1. Young GC的触发原因都是young generation 中的eden区满了

2. Serial Old GC／PS MarkSweep GC／Parallel Old GC的触发则是当准备要触发一次young GC时，如果发现统计数据说之前young GC的平均晋升大小比目前old gen剩余的空间大，则不会触发young GC而是转为触发full GC（因为HotSpot VM的GC里，除了CMS的concurrent collection之外，其它能收集old gen的GC都会同时收集整个GC堆，包括young gen，所以不需要事先触发一次单独的young GC）；或者，如果有perm gen的话，要在perm gen分配空间但已经没有足够空间时，也要触发一次full GC；或者System.gc()、heap dump带GC，默认也是触发full GC。

3. CMS GC,主要是定时去检查old gen的使用量，当使用量超过了触发比例就会启动一次CMS GC，对old gen做并发收集。

4. G1 GC的触发条件是Heap使用比率超过某值(待验证)

5. Full GC for CMS算法,主要是当CMS发生Concurrent Mode Faulure时，启用Serial Old来回收。

6. PS MarkSweep GC／Parallel Old GC（Full GC）之前会跑一次Parallel Young GC；

## System.gc
System.gc只是建议JVM执行一次GC。具体取决于JVM。Sun JDK的实现上，System.gc()的默认效果是引发一次stop-the-world的full GC，对整个GC堆做收集。如果设置了-XX:+DisableExplicitGC，那么不会触发GC。而且，System.gc不一定触发Full gc，还是取决于JVM。


## NIO的direct memory 和 -XX:+DisableExplicitGC
很多人都见过JVM调优建议里使用这个参数，对吧？但是为什么要用它，什么时候应该用而什么时候用了会掉坑里呢？

首先要了解的是这个参数的作用。在Oracle/Sun JDK这个具体实现上，System.gc()的默认效果是引发一次stop-the-world的full GC，对整个GC堆做收集。关键点是，用了-XX:+DisableExplicitGC参数后，System.gc()的调用就会变成一个空调用，完全不会触发任何GC（但是“函数调用”本身的开销还是存在的哦～）。

其中一种情况是下述三个条件同时满足时会发生的：
1. 应用本身在GC堆内的对象行为良好，正常情况下很久都不发生full GC；

2. 应用大量使用了NIO的direct memory，经常、反复的申请DirectByteBuffer

3. 使用了-XX:+DisableExplicitGC

Oracle/Sun JDK的实现里，DirectByteBuffer有几处值得注意的地方。

1. DirectByteBuffer没有finalizer，它的native memory的清理工作是通过sun.misc.Cleaner自动完成的。

2. sun.misc.Cleaner是一种基于PhantomReference的清理工具，比普通的finalizer轻量些。

Oracle/Sun JDK 6中的HotSpot VM只会在old gen GC（full GC/major GC或者concurrent GC都算）的时候才会对old gen中的对象做reference processing，而在young GC/minor GC时只会对young gen里的对象做reference processing。

也就是说，做full GC的话会对old gen做reference processing，进而能触发Cleaner对已死的DirectByteBuffer对象做清理工作。而如果很长一段时间里没做过GC或者只做了young GC的话则不会在old gen触发Cleaner的工作，那么就可能让本来已经死了的、但已经晋升到old gen的DirectByteBuffer关联的native memory得不到及时释放。

3. 为DirectByteBuffer分配空间过程中会显式调用System.gc()，以期通过full GC来强迫已经无用的DirectByteBuffer对象释放掉它们关联的native memory

这几个实现特征使得Oracle/Sun JDK 6依赖于System.gc()触发GC来保证DirectByteMemory的清理工作能及时完成。如果打开了-XX:+DisableExplicitGC，清理工作就可能得不到及时完成，于是就有机会见到direct memory的OOM。
