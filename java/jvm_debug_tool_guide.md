### -XX:+PrintCommandLineFlags
这个参数的作用是显示出VM初始化完毕后所有跟最初的默认值不同的参数及它们的值。

### -XX:+PrintFlagsFinal
前一个参数只显示跟默认值不同的，而这个参数则可以显示所有可设置的参数及它们的值。不过这个参数本身只从JDK 6 update 21开始才可以用，之前的Oracle/Sun JDK则用不了。
可以设置的参数默认是不包括diagnostic或experimental系的。要在-XX:+PrintFlagsFinal的输出里看到这两种参数的信息，分别需要显式指定-XX:+UnlockDiagnosticVMOptions / -XX:+UnlockExperimentalVMOptions 。

### -XX:+PrintFlagsInitial
这个参数显示在处理参数之前所有可设置的参数及它们的值，然后直接退出程序。“参数处理”包括许多步骤，例如说检查参数之间是否有冲突，通过ergonomics调整某些参数的值，之类的。

结合-XX:+PrintFlagsInitial与-XX:+PrintFlagsFinal，对比两者的差异，就可以知道ergonomics对哪些参数做了怎样的调整。

### 除了在VM启动时传些特殊的参数让它打印出自己的各参数外，jinfo -flag 可以用来查看某个参数的值，也可以用来设定manageable系参数的值。

### 1. jps:虚拟机进程状况工具
这个命令的功能和Unix 下的`ps`命令类似：它可以列出正在运行的虚拟机进程，并显示虚拟机主类(`main()`函数所在的类)名称，以及这些进程的本地虚拟机唯一Id(Local Virtual Machine Identifier,LVMID)。对于本地虚拟机进程来说，LVMID与操作系统的进程ID是一致的。

以下的命令的Java Version 为：`1.8.0_66`

输入`jps -help`可以显示jps命令的用法：

````
dengyunhuideMacBook-Pro:~ dengyunhui$ jps -help
usage: jps [-help]
       jps [-q] [-mlvV] [<hostid>]

Definitions:
    <hostid>:      <hostname>[:<port>]
````

如上，我们可以看到，jps 命令的格式为： `jps [options] [hostid]`

jps 可以通过RMI协议查询开启了RMI服务的远程虚拟机进程状态，`hostid`为RMI注册表中注册的主机名。

jps 的 `options`为`-q -mlvV`，所表示的含义如下所示:

| option | 作用     |
| :------------- | :------------- |
| q     |    只输出LVMID,省略主类的名称    |
| m     |    输出虚拟机进程启动时传递给主类main()方法的参数    |
| l     |    输出主类的全名，如果进程执行的是jar包，输出jar包路径    |
| v     |    输出虚拟机进程启动时的JVM参数    |
| V     |   输出LVMID和主类的名称     |

````
dengyunhuideMacBook-Pro:~ dengyunhui$ jps -q
4497
3747
18100
3688
3705
3725
dengyunhuideMacBook-Pro:~ dengyunhui$ jps -m
4497
3747 ZooKeeperMain -server 127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183
18101 Jps -m
3688 QuorumPeerMain ./z1.cfg
3705 QuorumPeerMain ./z2.cfg
3725 QuorumPeerMain ./z3.cfg
dengyunhuideMacBook-Pro:~ dengyunhui$ jps -l
4497
3747 org.apache.zookeeper.ZooKeeperMain
18104 sun.tools.jps.Jps
3688 org.apache.zookeeper.server.quorum.QuorumPeerMain
3705 org.apache.zookeeper.server.quorum.QuorumPeerMain
3725 org.apache.zookeeper.server.quorum.QuorumPeerMain
dengyunhuideMacBook-Pro:~ dengyunhui$ jps -v
4497  -Dfile.encoding=UTF-8 -XX:+UseConcMarkSweepGC -XX:SoftRefLRUPolicyMSPerMB=50 -ea -Dsun.io.useCanonCaches=false -Djava.net.preferIPv4Stack=true -XX:+HeapDumpOnOutOfMemoryError -XX:-OmitStackTraceInFastThrow -Xverify:none -Xbootclasspath/a:../lib/boot.jar -Xms128m -Xmx750m -XX:MaxPermSize=350m -XX:ReservedCodeCacheSize=240m -XX:+UseCompressedOops -Djb.vmOptionsFile=/Applications/IntelliJ IDEA 15.app/Contents/bin/idea.vmoptions -Didea.java.redist=custom-jdk-bundled -Didea.home.path=/Applications/IntelliJ IDEA 15.app/Contents -Didea.executable=idea -Didea.paths.selector=IntelliJIdea15
3747 ZooKeeperMain -Dzookeeper.log.dir=. -Dzookeeper.root.logger=INFO,CONSOLE
3688 QuorumPeerMain -Dzookeeper.log.dir=. -Dzookeeper.root.logger=INFO,CONSOLE -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.local.only=false
18105 Jps -Dapplication.home=/Library/Java/JavaVirtualMachines/jdk1.8.0_66.jdk/Contents/Home -Xms8m
3705 QuorumPeerMain -Dzookeeper.log.dir=. -Dzookeeper.root.logger=INFO,CONSOLE -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.local.only=false
3725 QuorumPeerMain -Dzookeeper.log.dir=. -Dzookeeper.root.logger=INFO,CONSOLE -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.local.only=false
dengyunhuideMacBook-Pro:~ dengyunhui$ jps -V
4497
3747 ZooKeeperMain
3688 QuorumPeerMain
3705 QuorumPeerMain
18106 Jps
3725 QuorumPeerMain
````

参考:

[](http://docs.oracle.com/javase/7/docs/technotes/tools/share/jps.html)

### 2. jstat:虚拟机统计信息监视工具
jstat(JVM Statistics Monitoring Tool)是用于监视虚拟机各种运行状态信息的命令行工具。它可以显示本地或远程虚拟机进程中的类装载、内存、垃圾收集、JIT编译等运行数据。

jstat用法：

````
Usage: jstat -help|-options
       jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]]

Definitions:
  <option>      An option reported by the -options option
  <vmid>        Virtual Machine Identifier. A vmid takes the following form:
                     <lvmid>[@<hostname>[:<port>]]
                Where <lvmid> is the local vm identifier for the target
                Java virtual machine, typically a process id; <hostname> is
                the name of the host running the target Java virtual machine;
                and <port> is the port number for the rmiregistry on the
                target host. See the jvmstat documentation for a more complete
                description of the Virtual Machine Identifier.
  <lines>       Number of samples between header lines.
  <interval>    Sampling interval. The following forms are allowed:
                    <n>["ms"|"s"]
                Where <n> is an integer and the suffix specifies the units as
                milliseconds("ms") or seconds("s"). The default units are "ms".
  <count>       Number of samples to take before terminating.
  -J<flag>      Pass <flag> directly to the runtime system.
````

对于命令行中的vmid与lvmid需要特别说明一下：如果本地虚拟机进程，vmid和lvmid一致，如果是远程虚拟机进程，那么vmid的格式应该是:

`<lvmid>[@<hostname>[:<port>]]`

参数`interval`和`count` 代表取样的间隔和次数，如果省略这两个参数，说明只查询一次。

格式化参数有两个：`-t` 和 `-h<lines>`。

+ `-t`参数控制在输出第一列添加时间戳信息;

+ `-h<lines>`参数指定每隔n行重新显示一次列名。

`-J<flag>`参数用于传递javaOption到java程序启动参数。比如，-J-Xms48m 设置java启动最小内存为48M。

`option`表示用户希望查询的虚拟机信息，主要分为3类：类装载，垃圾收集，运行期编译情况。如下所示：

````
dengyunhuideMacBook-Pro:~ dengyunhui$ jstat -options
-class
-compiler
-gc
-gccapacity
-gccause
-gcmetacapacity
-gcnew
-gcnewcapacity
-gcold
-gcoldcapacity
-gcutil
-printcompilation
````

| option | 作用     |
| :------------- | :------------- |
|    -class    |  监视类装载、卸载数量、总空间以及类装载所耗费的时间     |
|    -gc    |   监视Java堆状况，包括Eden区、两个survivor区、老年代、永久代等的容量、已用空间、GC时间合计等信息    |
|    -compiler   |     输出JIT编译器编译过的方法、耗时等信息  |
|      -gccapacity  |    监视内容与-gc基本相同，但输出主要关注Java堆各个区域使用到的最大、最小空间   |
|     -gccause   |   与-gcutil功能一样，但是会额外输出导致上一次gc的原因    |
|      -gcutil  |   监视内容与-gc基本相同，但输出主要关注已使用空间占总空间的百分比    |
|   -gcnew     |   监视新生代GC状况   |
|    -gcnewcapacity    |    监视内容和-gcnew 基本相同，但输出主要关注使用到的最大、最小空间   |
|     -gcold   |   监视老年代GC状况    |
|     -gcoldcapacity   |    监视内容和-gcold基本相同，输出主要关注使用到的最大、最小空间   |
|     -printcompilation   |   输出已经被JIT编译的方法    |
|     -gcmetacapacity   |    监视meta space（元数据区）GC状况   |

> Java 1.8里移除了永久区(PermGen space),诞生了一个新的内存区域:Metaspace

比如我们想使用`-gcutil`参数查看进程 3747 的垃圾回收情况，每隔2秒打印一次，总共打印4次:

`jstat -gcutil 3747 2000 4`

输出如下：

````
dengyunhuideMacBook-Pro:~ dengyunhui$ jstat -gcutil 3747 2000 4
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
  0.00   0.00  24.03   0.00  17.22  19.75      0    0.000     0    0.000    0.000
  0.00   0.00  24.03   0.00  17.22  19.75      0    0.000     0    0.000    0.000
  0.00   0.00  24.03   0.00  17.22  19.75      0    0.000     0    0.000    0.000
  0.00   0.00  24.03   0.00  17.22  19.75      0    0.000     0    0.000    0.000
````

可以看到有很多列。那么这些列是如何定义的呢？含义是什么呢？

其实我们可以在`tools.jar`找到一个文件`jstat_options`。这个文件定义了每个option的输出列以及每个列的含义。

我们可以看看：

````
option gcutil {
  column {
    header "^S0^"	/* Survivor 0 Space - Percent Used */
    data (1-((sun.gc.generation.0.space.1.capacity - sun.gc.generation.0.space.1.used)/sun.gc.generation.0.space.1.capacity)) * 100
    scale raw
    align right
    width 6
    format "0.00"
  }
  column {
    header "^S1^"	/* Survivor 1 Space - Percent Used */
    data (1-((sun.gc.generation.0.space.2.capacity - sun.gc.generation.0.space.2.used)/sun.gc.generation.0.space.2.capacity)) * 100
    scale raw
    align right
    width 6
    format "0.00"
  }
  column {
    header "^E^"	/* Eden Space - Percent Used */
    data (1-((sun.gc.generation.0.space.0.capacity - sun.gc.generation.0.space.0.used)/sun.gc.generation.0.space.0.capacity)) * 100
    align right
    scale raw
    width 6
    format "0.00"
  }
  column {
    header "^O^"	/* Old Space - Percent Used */
    data (1-((sun.gc.generation.1.space.0.capacity - sun.gc.generation.1.space.0.used)/sun.gc.generation.1.space.0.capacity)) * 100
    align right
    scale raw
    width 6
    format "0.00"
  }
  column {
    header "^M^"	/* Metaspace Space - Percent Used */
    data (1-((sun.gc.metaspace.capacity - sun.gc.metaspace.used)/sun.gc.metaspace.capacity)) * 100
    align right
    width 6
    scale raw
    format "0.00"
  }
  column {
    header "^CCS^"	/* Compressed Class Space Space - Percent Used */
    data (1-((sun.gc.compressedclassspace.capacity - sun.gc.compressedclassspace.used)/sun.gc.compressedclassspace.capacity)) * 100
    align right
    width 6
    scale raw
    format "0.00"
  }
  column {
    header "^YGC^"	/* Young Generation Collections */
    data sun.gc.collector.0.invocations
    align right
    width 6
    format "0"
  }
  column {
    header "^YGCT^"	/* Young Generation Collection Time */
    data sun.gc.collector.0.time/sun.os.hrt.frequency
    align right
    scale sec
    width 8
    format "0.000"
  }
  column {
    header "^FGC^"	/* Full Collections */
    data sun.gc.collector.1.invocations
    align right
    width 5
    scale raw
    format "0"
  }
  column {
    header "^FGCT^"	/* Full Collection Time */
    data sun.gc.collector.1.time/sun.os.hrt.frequency
    align right
    scale sec
    width 8
    format "0.000"
  }
  column {
    header "^GCT^"	/* Total Garbage Collection Time */
    data (sun.gc.collector.0.time + sun.gc.collector.1.time)/sun.os.hrt.frequency
    align right
    width 8
    scale sec
    format "0.000"
  }
}
....
限于篇幅，不一一列举了
....
````

现在我们知道每一列是什么含义了，我们主要看几个选项。

1. `-gcutil`
使用option `-gcutil`查看进程21891 的内存回收情况，每250毫秒打印一次,打印7次。

````
jstat -gcutil 21891 250 7

  S0     S1     E      O      P     YGC    YGCT    FGC    FGCT     GCT

 12.44   0.00  27.20   9.49  96.70    78    0.176     5    0.495    0.672

 12.44   0.00  62.16   9.49  96.70    78    0.176     5    0.495    0.672

 12.44   0.00  83.97   9.49  96.70    78    0.176     5    0.495    0.672

  0.00   7.74   0.00   9.51  96.70    79    0.177     5    0.495    0.673

  0.00   7.74  23.37   9.51  96.70    79    0.177     5    0.495    0.673

  0.00   7.74  43.82   9.51  96.70    79    0.177     5    0.495    0.673

  0.00   7.74  58.11   9.51  96.71    79    0.177     5    0.495    0.673
````

上图中，`S0`表示`Survivor 0 Space`的使用率；`S1`表示`Survivor 1 Space`的使用率；`E`表示 Eden 区的使用率；`O`表示 Old 区的使用率；`M` 表示 `Meta Space`使用率；`CCS`表示 Compressed Class Space Space的使用率; `YGC`表示Young GC 的次数；`YGCT`表示Young GC 消耗的时间；`FGC`表示Full GC 的次数;`FGCT`表示Full GC消耗的时间；`GCT`表示GC消耗的总时间。

2. `-gc`
使用 option `-gc`查看更加详细的垃圾回收与堆内存状况。

````
dengyunhuideMacBook-Pro:~ dengyunhui$ jstat -gc 3747
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT
10752.0 10752.0  0.0    0.0   65536.0  15750.7   175104.0     0.0     4480.0 771.4  384.0   75.8       0    0.000   0      0.000    0.000
````

`S0C`表示 Survivor 0 区的当前容量；`S0U`表示 Survivor 1 区的已经使用的空间。和`-gcutil`的区别是，`-gc`显示的是真实的容量，而不是百分比。

3. `-gcnew`
使用 option `-gcnew` 查看新生代的垃圾回收和内存情况。

````
dengyunhuideMacBook-Pro:~ dengyunhui$ jstat -gcnew 3747
   S0C    S1C    S0U    S1U   TT MTT  DSS      EC       EU     YGC     YGCT
10752.0 10752.0    0.0    0.0 15  15    0.0  65536.0  15750.7      0    0.000
````

这里主要要注意几个参数：`TT` 表示 Tenuring Threshold。`MTT`表示 Maximum Tenuring Threshold。`DSS`表示 Desired Survivor Size。

>因为采用了分代收集的思想来管理内存，那么内存回收就必须能识别哪些对象应该放在新生代，哪些对象应该放在老年代。为了做到这一点，虚拟机给每个对象定义了一个对象年龄计数器。如果对象在Eden区出生并经过第一次Young GC 后仍然存活，并且能被Survivor容纳的话，将被移动到另一个Survivor空间中，并且对象年龄设为1，当他的年龄增加到一定的程度（默认15岁），就会被晋升到老年代。对象晋升到老年代的阈值可以通过参数`-XX:MaxTenuringThreshold`设置。

 TT 参数表示对象在gc时被放入老年代的年龄期限阈值，MTT 参数表示最大阈值. 这两个参数之间 MTT 设定了 TT 可取的最大值，TT 实际控制着对象进入老年代的年龄限制，会随着垃圾回收过程而发生变化。

 >为了能更好的适应不同程序的内存状况，虚拟机并不是永远地要求对象年龄必须达到了MaxTenuringThreshold才能晋升到老年代。如果在Survivor空间中相同年龄所有对象大小的总和大于Survivor空间的一半，年龄大于或等于该年龄的对象就可以直接进入老年代，无须等到MaxTenuringThreshold中要求的年龄。

 当年龄从1开始的对象大小累计超过了Survivor区域的1/2(TargetSurvivorRatio所定义)时，会计算一个Thenuring Threshold，超过这个年龄的新生代对象会进入到老年代，即使这时候新生代还有很多的空间。注意MaxTenuringThreshold只是设置了最大的Thenuring Threshold，不是说只有大于Max Tenuring Threshold才会进入到老年代，而是只要超过了计算出来的Tenuring Threshold就会进入老年代，MaxTenuringThreshold规定了Tenuring Threshold的最大值而已。Tenuring Threshold这个值在每一轮GC后都会动态计算，它与TargetSurvivorRatio以及Survivor区的大小有关系，TargetSurivivor默认是50即Survivor的1/2, 会计算出一个Desired Survivor Size，当年龄从1开始的对象大小累计超过了这个Desired Survivor Size，那么这个age就是Tenuring Threshold的值。

 4. `-gcnewcapacity`
 使用 option `-gcnewcapacity`查看新生代的内存占用情况。

````
 dengyunhuideMacBook-Pro:~ dengyunhui$ jstat -gcnewcapacity 3747
 NGCMN      NGCMX       NGC      S0CMX     S0C     S1CMX     S1C       ECMX        EC      YGC   FGC
87040.0  1397760.0    87040.0 465920.0  10752.0 465920.0  10752.0  1396736.0    65536.0     0     0
````   

+ NGCMN:New Generation Capacity - Minimum

+ NGCMX:New Generation Capacity - Maximum

+ NGC:New Generation Capacity - Current

+ S0CMX:Survivor 0 Space Capacity - Maximum

+ S0C:Survivor 0 Space Capacity - Current

+ ECMX:Eden Space Capacity - Maximum

+ EC:Eden Space Capacity - Current

+ YGC:Young Generation Collections

+ FGC:Full Collections

NGC : new generation capacity, 新生代内存大小。NGCMN 表示新生代分配内存的最小值，NGCMX 新生代分配内存的最大值，NGCMX = S0CMX + S1CMX + ECMX.

参考：

[](http://www.codeceo.com/article/jvm-debug-tools-guide.html)

[](http://docs.oracle.com/javase/7/docs/technotes/tools/share/jstat.html)

### 3. jinfo：Java配置信息工具
jinfo的作用是实时查看和调整虚拟机各项参数。

````
Usage:
    jinfo [option] <pid>
        (to connect to running process)
    jinfo [option] <executable <core>
        (to connect to a core file)
    jinfo [option] [server_id@]<remote server IP or hostname>
        (to connect to remote debug server)

where <option> is one of:
    -flag <name>         to print the value of the named VM flag
    -flag [+|-]<name>    to enable or disable the named VM flag
    -flag <name>=<value> to set the named VM flag to the given value
    -flags               to print VM flags
    -sysprops            to print Java system properties
    <no option>          to print both of the above
    -h | -help           to print this help message
````

从上面的Usage可以看到，jinfo有三种用法，jinfo可以输出java进程、core文件(jar 包)或远程debug服务器的配置信息。这些配置信息包括JAVA系统参数及命令行参数,如果进程运行在64位虚拟机上，需要指明-J-d64参数，如：`jinfo -J-d64 -sysprops pid`

另外，在[JDK 7的文档](http://docs.oracle.com/javase/7/docs/technotes/tools/share/jinfo.html)中说这个命令，今后JDK可能不支持了。

前面我们已经知道，可以通过`java -XX:+PrintFlagsFinal`查看参数默认值。

jinfo 还可以通过使用`-sysprops`选项把虚拟机进程的`System.getProperties()`的内容打印出来。

同时，注意上面的`-flag <name>=<value> to set the named VM flag to the given value`,使用`-flag <name>=<value> `可以修改一部分运行期可写（也就是`-XX:+PrintFlagsFinal`里显示为`manageable`的参数）的虚拟机参数值。

示例：

`jinfo 3747`

`jinfo -flags 3747`

在运行时改变JVM manageable的参数：

首先,我们通过`java -XX:+PrintFlagsFinal`查看所有的可配置的参数以及其默认值。选取其中的一个标志为`manageable`的，比如`PrintClassHistogramAfterFullGC`

````
jinfo -flag PrintClassHistogramAfterFullGC 83105
-XX:-PrintClassHistogramAfterFullGC
````

然后，改变这个参数的值。

````
jinfo -flag +PrintClassHistogramAfterFullGC 83105
````

最后，我们看一下改变后的值。

````
jinfo -flag PrintClassHistogramAfterFullGC 83105
-XX:+PrintClassHistogramAfterFullGC
````

可以看到，进程83105的`PrintClassHistogramAfterFullGC`参数值变为true了

**要特别注意的是，在运行期，JVM只能改变标志为manageable的参数**

有时，我们可能想让JVM在一些特定事件发生的时候自动做heap dump，比如发生out of Memory,那么可以通过jinfo指定。

除了HeapDumpOnOutOfMemoryError之外，还有HeapDumpBeforeFullGC与HeapDumpAfterFullGC参数，分别用于指定在full GC之前与之后生成heap dump。


参考：

[](http://docs.oracle.com/javase/7/docs/technotes/tools/share/jinfo.html)

### 4. jmap：java内存映像工具
jmap(Memory Map for Java)用于生成堆转储快照（dump）。如果不使用jmap命令，要想获得Java堆转储快照，还有一些比较"暴力的"手段：譬如可以使用参数`-XX:+HeapDumpOnOutOfMemoryError`参数，可以让虚拟机在出现OOM异常之后自动生成dump文件，通过`-XX:HeapDumpOnCtrlBreak`参数则可以使用`Ctrl + Break`键让虚拟机生成dump文件，又或者在Linux系统下通过`Kill -3`命令发送进程退出信号“吓唬”一下虚拟机，也能拿到dump文件。

jmap的作用不仅仅是为了获得dump文件，它还可以查询finalize执行队列、Java 堆和永久代的详细信息，如空间使用率，当前使用的是哪种收集器等。

Usage：

````
Usage:
    jmap [option] <pid>
        (to connect to running process)
    jmap [option] <executable <core>
        (to connect to a core file)
    jmap [option] [server_id@]<remote server IP or hostname>
        (to connect to remote debug server)

where <option> is one of:
    <none>               to print same info as Solaris pmap
    -heap                to print java heap summary
    -histo[:live]        to print histogram of java object heap; if the "live"
                         suboption is specified, only count live objects
    -clstats             to print class loader statistics
    -finalizerinfo       to print information on objects awaiting finalization
    -dump:<dump-options> to dump java heap in hprof binary format
                         dump-options:
                           live         dump only live objects; if not specified,
                                        all objects in the heap are dumped.
                           format=b     binary format
                           file=<file>  dump heap to <file>
                         Example: jmap -dump:live,format=b,file=heap.bin <pid>
    -F                   force. Use with -dump:<dump-options> <pid> or -histo
                         to force a heap dump or histogram when <pid> does not
                         respond. The "live" suboption is not supported
                         in this mode.
    -h | -help           to print this help message
    -J<flag>             to pass <flag> directly to the runtime system
````

一般比较常用的是：`jmap [option] <pid>`。

option的值和含义如下：

| option | 含义    |
| :------------- | :------------- |
|    -heap   |     显示java堆的详细信息，如使用何种收集器，参数配置，分代情况等。只在Linux/Unix平台有效  |
|   -histo[:live]    |    显示堆中对象统计信息，包括类、实例数量、合计容量。如果指定了"live"选项，只显示活着的对象   |
|    -clstats   |   输出类加载器的统计信息    |
|     -finalizerinfo  |   输出等待finalization的对象信息    |
|    -dump:<dump-options>   |    生成dump堆转储快照。格式为:-dump:[:live] format=b,file=<filename>.live参数用于指定是否只dump出存活的对象   |
|    -F   |    强制:force   |
|    -h | -help   |    帮助信息   |
|    -J<flag>   |   传递<flag>参数到 Runtime，比如：-XX:+HeapDumpOnOutOfMemoryError    |

示例：

1. `-heap`

````
dengyunhuideMacBook-Pro:~ dengyunhui$ jmap -heap 3747
Attaching to process ID 3747, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.66-b17

using thread-local object allocation.
Parallel GC with 8 thread(s)

Heap Configuration:
   MinHeapFreeRatio         = 0
   MaxHeapFreeRatio         = 100
   MaxHeapSize              = 4294967296 (4096.0MB)
   NewSize                  = 89128960 (85.0MB)
   MaxNewSize               = 1431306240 (1365.0MB)
   OldSize                  = 179306496 (171.0MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)

Heap Usage:
PS Young Generation
Eden Space:
   capacity = 67108864 (64.0MB)
   used     = 20155784 (19.22205352783203MB)
   free     = 46953080 (44.77794647216797MB)
   30.03445863723755% used
From Space:
   capacity = 11010048 (10.5MB)
   used     = 0 (0.0MB)
   free     = 11010048 (10.5MB)
   0.0% used
To Space:
   capacity = 11010048 (10.5MB)
   used     = 0 (0.0MB)
   free     = 11010048 (10.5MB)
   0.0% used
PS Old Generation
   capacity = 179306496 (171.0MB)
   used     = 0 (0.0MB)
   free     = 179306496 (171.0MB)
   0.0% used
````

从上面我们可以看出，对象的分配使用的是`thread-local object allocation`,使用的垃圾算法是`Parallel GC with 8 thread(s)`，所以使用的垃圾收集器是Parallel + Serial Old/Parallel Old。

2. `-finalizerinfo`

````
dengyunhuideMacBook-Pro:~ dengyunhui$ jmap -finalizerinfo 3747
Attaching to process ID 3747, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.66-b17
Number of objects pending for finalization: 0
````

3. `-histo`
这个命令打印出来的信息有点多，所以用nohup转储到文件中。

````
dengyunhuideMacBook-Pro:~ dengyunhui$ nohup jmap -histo 3747
appending output to nohup.out
````

以下是一些信息：

````
Attaching to process ID 3747, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.66-b17
Number of objects pending for finalization: 0

 num     #instances         #bytes  class name
----------------------------------------------
   1:          2693       10978312  [I
   2:         10884        3861368  [B
   3:         24381        2410776  [C
   4:         15276         366624  java.lang.String
   5:           813         326776  [S
   6:          3284         184096  [Ljava.lang.Object;
   7:          3231         180936  jdk.internal.org.objectweb.asm.Item
   8:          1400         158056  java.lang.Class
   9:          2979         119160  java.util.HashMap$KeyIterator
  10:           184         106880  [Ljdk.internal.org.objectweb.asm.Item;
  11:          1792          86016  java.nio.HeapByteBuffer
  12:          2608          83456  java.util.HashMap$Node
  13:          2451          63504  [Ljava.lang.Class;
  14:          2012          48288  java.lang.StringBuilder
  15:           501          44088  java.lang.reflect.Method
....
````

4. `-dump`

````
dengyunhuideMacBook-Pro:~ dengyunhui$ jmap -dump:format=b,file=dump-2016.txt 3747
Dumping heap to /Users/dengyunhui/dump-2016.txt ...
Heap dump file created
````

5. `-clstats`

````
dengyunhuideMacBook-Pro:~ dengyunhui$ jmap -clstats 3747
Attaching to process ID 3747, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.66-b17
finding class loader instances ..done.
computing per loader stat ..done.
please wait.. computing liveness.liveness analysis may be inaccurate ...
class_loader   	classes	bytes  	parent_loader  	alive? 	type

<bootstrap>    	990    	1785567	  null         	live   	<internal>
0x000000076b11bf50     	1      	1474   	  null         	dead   	sun/reflect/DelegatingClassLoader@0x00000007c0009df8
0x000000076ab588e0     	300    	635682 	0x000000076ab3dd88     	dead   	sun/misc/Launcher$AppClassLoader@0x00000007c000f688
0x000000076b115d98     	1      	880    	  null         	dead   	sun/reflect/DelegatingClassLoader@0x00000007c0009df8
0x000000076b117a38     	1      	880    	  null         	dead   	sun/reflect/DelegatingClassLoader@0x00000007c0009df8
0x000000076b119408     	1      	880    	  null         	dead   	sun/reflect/DelegatingClassLoader@0x00000007c0009df8
0x000000076ab3dd88     	3      	2574   	  null         	dead   	sun/misc/Launcher$ExtClassLoader@0x00000007c000fa30
0x000000076b030658     	0      	0      	0x000000076ab588e0     	dead   	java/util/ResourceBundle$RBClassLoader@0x00000007c007b1f8
0x000000076b0fb970     	1      	880    	  null         	dead   	sun/reflect/DelegatingClassLoader@0x00000007c0009df8

total = 9      	1298   	2428817	    N/A        	alive=1, dead=8	    N/A
````

### 5. jhat：虚拟机堆转储快照分析工具
Sun JDK 提供了jhat命令与jmap搭配使用，用来分析jmap生成的dump文件。

Usage：

````
Usage:  jhat [-stack <bool>] [-refs <bool>] [-port <port>] [-baseline <file>] [-debug <int>] [-version] [-h|-help] <file>

       	-J<flag>          Pass <flag> directly to the runtime system. For
       			  example, -J-mx512m to use a maximum heap size of 512MB
       	-stack false:     Turn off tracking object allocation call stack.
       	-refs false:      Turn off tracking of references to objects
       	-port <port>:     Set the port for the HTTP server.  Defaults to 7000
       	-exclude <file>:  Specify a file that lists data members that should
       			  be excluded from the reachableFrom query.
       	-baseline <file>: Specify a baseline object dump.  Objects in
       			  both heap dumps with the same ID and same class will
       			  be marked as not being "new".
       	-debug <int>:     Set debug level.
       			    0:  No debug output
       			    1:  Debug hprof file parsing
       			    2:  Debug hprof file parsing, no server
       	-version          Report version number
       	-h|-help          Print this help and exit
       	<file>            The file to read

For a dump file that contains multiple heap dumps,
you may specify which dump in the file
by appending "#<number>" to the file name, i.e. "foo.hprof#3".

All boolean options default to "true"
````

各种用法如上所示，不细说了。

示例：用jhat分析dump文件.

````
dengyunhuideMacBook-Pro:~ dengyunhui$ jhat /Users/dengyunhui/dump.txt
Reading from /Users/dengyunhui/dump.txt...
Dump file created Fri Dec 02 17:22:06 CST 2016
Snapshot read, resolving...
Resolving 105534 objects...
Chasing references, expect 21 dots.....................
Eliminating duplicate references.....................
Snapshot resolved.
Started HTTP server on port 7000
Server is ready.
````

显示`Server is ready`之后，在浏览器中键入`http://localhost:7000`就可以看到分析结果。

### jstack:Java 堆栈跟踪工具
jstack命令用于生成虚拟机当前时刻的线程快照（一般称为thread dump 文件或java core文件）。线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈的集合。生成线程快照的目的是定位线程出现长时间停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间等待等都是导致线程长时间停顿的常见原因。

Usage:

````
Usage:
    jstack [-l] <pid>
        (to connect to running process)
    jstack -F [-m] [-l] <pid>
        (to connect to a hung process)
    jstack [-m] [-l] <executable> <core>
        (to connect to a core file)
    jstack [-m] [-l] [server_id@]<remote server IP or hostname>
        (to connect to a remote debug server)

Options:
    -F  to force a thread dump. Use when jstack <pid> does not respond (process is hung)
    -m  to print both java and native frames (mixed mode)
    -l  long listing. Prints additional information about locks
    -h or -help to print this help message
````    

+ pid
待追踪的java进程ID，可使用jps等方式获得

+ executable
产生core dump的java可执行程序(jar文件)

+ core
打印栈追踪信息的核心文件

| option | 含义     |
| :------------- | :------------- |
|   -F   |   当正常输出的请求不被响应时，强制输出线程堆栈     |
|   -l   |  除堆栈外，显示关于锁的附加信息      |
|    -m  |  如果调用到本地方法的话，可以显示C/C++ 的堆栈      |

jstack在分析死锁，阻塞等性能问题上非常有用，根据打印的堆栈信息可以定位到出问题的代码段。定位问题的思路根据要解决的问题而发生不同，比如可以首先找到java进程中最耗cpu的线程，再根据线程id在jstack的输出中定位，或者使用指定的线程名称定位。我们看看jstack的输出格式：

````
"Thread-1" #12 prio=5 os_prio=31 tid=0x00007f9fbf843800 nid=0x5b03 waiting for monitor entry [0x00007000079f9000]
   java.lang.Thread.State: BLOCKED (on object monitor)
       	at com.dyh.tutorial.DeadLockDemo$2.run(DeadLockDemo.java:39)
       	- waiting to lock <0x000000076abfd9b8> (a java.lang.Object)
       	- locked <0x000000076abfd9c8> (a java.lang.Object)
       	at java.lang.Thread.run(Thread.java:745)

   Locked ownable synchronizers:
       	- None
````

+ “Thread-1”,线程名称

+ "prio=5",线程优先级

+ "os_prio=31",对应的操作系统线程的优先级

+ "tid=0x00007f9fbf843800",jvm中线程标识符

+ "nid=0x5b03",16进制表示的本地线程标识符

+ "waiting for monitor entry",线程状态

上述信息中最重要的状态莫过于线程的状态，当程序发生问题时往往能够据此帮我们找到问题的所在。在jstack的输出中，线程所处的状态包括：

+ Runnable: 正在运行

+ Wait on condition: 该状态出现在线程等待某个条件的发生。具体是什么原因，可以结合 stacktrace来分析。最常见的情况是线程在等待网络的读写。如果网络数据没准备好，线程就等待在那里。另外一种出现 Wait on condition的常见情况是该线程在 sleep，等待 sleep的时间到了时候，将被唤醒。

+ Wait for monitor entry: Java通过对象监视器来进行线程的互斥与同步的，每个对象都有一个对象监视器，在对对象加锁的过程中，任何时刻只有一个线程拥有这个对象监视器，其他请求获得此资源的线程将会被分为两种：在线程获取到对象监视器，但等待的资源仍未到达时，线程可能调用Object.wait()，释放锁并进入Object.wait()的状态，重新等待；而那些从未获得过此对象监视器的线程就将被标识为Wait for monitor entry的状态。

+ Object.wait: 等待对象监视器（锁）的状态。


示例：

用jstack显示死锁信息。

我们先写一段导致死锁的代码：

````
public static void main(String[] args) throws IOException {

    //显示进程ID
    String name = ManagementFactory.getRuntimeMXBean().getName();
    System.out.println("pid is: " + name.split("@")[0]);

    final Object a = new Object();
    final Object b = new Object();

    Thread t1 = new Thread(new Runnable() {
        public void run() {
            synchronized (a){
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                synchronized (b){
                    System.out.println("t1");
                }
            }
        }
    });

    Thread t2 = new Thread(new Runnable() {
        public void run() {
            synchronized (b){
                synchronized (a){
                    System.out.print("t2");
                }
            }
        }
    });

    t1.start();
    t2.start();

}
````

进程Id为682:

````
dengyunhuideMacBook-Pro:~ dengyunhui$ jstack -l 682
2016-12-02 19:06:57
Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.66-b17 mixed mode):

"Attach Listener" #14 daemon prio=9 os_prio=31 tid=0x00007f9fc1051800 nid=0x1407 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

   Locked ownable synchronizers:
       	- None

"DestroyJavaVM" #13 prio=5 os_prio=31 tid=0x00007f9fbf844800 nid=0x1c03 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

   Locked ownable synchronizers:
       	- None

"Thread-1" #12 prio=5 os_prio=31 tid=0x00007f9fbf843800 nid=0x5b03 waiting for monitor entry [0x00007000079f9000]
   java.lang.Thread.State: BLOCKED (on object monitor)
       	at com.dyh.tutorial.DeadLockDemo$2.run(DeadLockDemo.java:39)
       	- waiting to lock <0x000000076abfd9b8> (a java.lang.Object)
       	- locked <0x000000076abfd9c8> (a java.lang.Object)
       	at java.lang.Thread.run(Thread.java:745)

   Locked ownable synchronizers:
       	- None

"Thread-0" #11 prio=5 os_prio=31 tid=0x00007f9fbf82d800 nid=0x5903 waiting for monitor entry [0x00007000078f6000]
   java.lang.Thread.State: BLOCKED (on object monitor)
       	at com.dyh.tutorial.DeadLockDemo$1.run(DeadLockDemo.java:29)
       	- waiting to lock <0x000000076abfd9c8> (a java.lang.Object)
       	- locked <0x000000076abfd9b8> (a java.lang.Object)
       	at java.lang.Thread.run(Thread.java:745)

   Locked ownable synchronizers:
       	- None

"Monitor Ctrl-Break" #10 daemon prio=5 os_prio=31 tid=0x00007f9fc104f000 nid=0x5703 runnable [0x00007000077f3000]
   java.lang.Thread.State: RUNNABLE
       	at java.net.PlainSocketImpl.socketAccept(Native Method)
       	at java.net.AbstractPlainSocketImpl.accept(AbstractPlainSocketImpl.java:409)
       	at java.net.ServerSocket.implAccept(ServerSocket.java:545)
       	at java.net.ServerSocket.accept(ServerSocket.java:513)
       	at com.intellij.rt.execution.application.AppMain$1.run(AppMain.java:90)
       	at java.lang.Thread.run(Thread.java:745)

   Locked ownable synchronizers:
       	- None

"Service Thread" #9 daemon prio=9 os_prio=31 tid=0x00007f9fbf032000 nid=0x5303 runnable [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

   Locked ownable synchronizers:
       	- None

"C1 CompilerThread3" #8 daemon prio=9 os_prio=31 tid=0x00007f9fbf010800 nid=0x5103 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

   Locked ownable synchronizers:
       	- None

"C2 CompilerThread2" #7 daemon prio=9 os_prio=31 tid=0x00007f9fbf010000 nid=0x4f03 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

   Locked ownable synchronizers:
       	- None

"C2 CompilerThread1" #6 daemon prio=9 os_prio=31 tid=0x00007f9fbe009800 nid=0x4d03 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

   Locked ownable synchronizers:
       	- None

"C2 CompilerThread0" #5 daemon prio=9 os_prio=31 tid=0x00007f9fc1001000 nid=0x4b03 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

   Locked ownable synchronizers:
       	- None

"Signal Dispatcher" #4 daemon prio=9 os_prio=31 tid=0x00007f9fc1000000 nid=0x4903 runnable [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

   Locked ownable synchronizers:
       	- None

"Finalizer" #3 daemon prio=8 os_prio=31 tid=0x00007f9fbd008800 nid=0x3903 in Object.wait() [0x0000700006fdb000]
   java.lang.Thread.State: WAITING (on object monitor)
       	at java.lang.Object.wait(Native Method)
       	- waiting on <0x000000076ab070b8> (a java.lang.ref.ReferenceQueue$Lock)
       	at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:143)
       	- locked <0x000000076ab070b8> (a java.lang.ref.ReferenceQueue$Lock)
       	at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:164)
       	at java.lang.ref.Finalizer$FinalizerThread.run(Finalizer.java:209)

   Locked ownable synchronizers:
       	- None

"Reference Handler" #2 daemon prio=10 os_prio=31 tid=0x00007f9fbd003800 nid=0x3703 in Object.wait() [0x0000700006ed8000]
   java.lang.Thread.State: WAITING (on object monitor)
       	at java.lang.Object.wait(Native Method)
       	- waiting on <0x000000076ab06af8> (a java.lang.ref.Reference$Lock)
       	at java.lang.Object.wait(Object.java:502)
       	at java.lang.ref.Reference$ReferenceHandler.run(Reference.java:157)
       	- locked <0x000000076ab06af8> (a java.lang.ref.Reference$Lock)

   Locked ownable synchronizers:
       	- None

"VM Thread" os_prio=31 tid=0x00007f9fbf00f800 nid=0x3503 runnable

"GC task thread#0 (ParallelGC)" os_prio=31 tid=0x00007f9fbe004800 nid=0x2503 runnable

"GC task thread#1 (ParallelGC)" os_prio=31 tid=0x00007f9fbf801000 nid=0x2703 runnable

"GC task thread#2 (ParallelGC)" os_prio=31 tid=0x00007f9fbe005800 nid=0x2903 runnable

"GC task thread#3 (ParallelGC)" os_prio=31 tid=0x00007f9fbe006000 nid=0x2b03 runnable

"GC task thread#4 (ParallelGC)" os_prio=31 tid=0x00007f9fbe006800 nid=0x2d03 runnable

"GC task thread#5 (ParallelGC)" os_prio=31 tid=0x00007f9fbf00b800 nid=0x2f03 runnable

"GC task thread#6 (ParallelGC)" os_prio=31 tid=0x00007f9fbf00c800 nid=0x3103 runnable

"GC task thread#7 (ParallelGC)" os_prio=31 tid=0x00007f9fbf801800 nid=0x3303 runnable

"VM Periodic Task Thread" os_prio=31 tid=0x00007f9fc1044000 nid=0x5503 waiting on condition

JNI global references: 32


Found one Java-level deadlock:
=============================
"Thread-1":
  waiting to lock monitor 0x00007f9fc00100a8 (object 0x000000076abfd9b8, a java.lang.Object),
  which is held by "Thread-0"
"Thread-0":
  waiting to lock monitor 0x00007f9fc00116a8 (object 0x000000076abfd9c8, a java.lang.Object),
  which is held by "Thread-1"

Java stack information for the threads listed above:
===================================================
"Thread-1":
       	at com.dyh.tutorial.DeadLockDemo$2.run(DeadLockDemo.java:39)
       	- waiting to lock <0x000000076abfd9b8> (a java.lang.Object)
       	- locked <0x000000076abfd9c8> (a java.lang.Object)
       	at java.lang.Thread.run(Thread.java:745)
"Thread-0":
       	at com.dyh.tutorial.DeadLockDemo$1.run(DeadLockDemo.java:29)
       	- waiting to lock <0x000000076abfd9c8> (a java.lang.Object)
       	- locked <0x000000076abfd9b8> (a java.lang.Object)
       	at java.lang.Thread.run(Thread.java:745)

Found 1 deadlock.
````
如上所示，检测到了死锁。

参考：

1. http://www.codeceo.com/article/jvm-debug-tools-guide.html
2. http://docs.oracle.com/javase/7/docs/technotes/tools/share/jstack.html
3. 深入理解java虚拟机-JVM高级特性与最佳实践 周志明著
