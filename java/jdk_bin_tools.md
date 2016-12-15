JDK 的bin目录提供了很多工具，之前一直没有去了解，这次去面试被问到了，答不知道。所以借此机会了解一下。

JDK bin目录下的工具说明可以参考这篇文章[JDK Tools and Utilities](http://docs.oracle.com/javase/7/docs/technotes/tools/)。以下，对这篇文章做一个翻译。

标准JDK实用工具

+ 基本工具：appletviewer, apt, extcheck, jar, java, javac, javadoc, javah, javap, jdb

+ 安全工具：keytool,jarsigner,policytool,kinit,klist,ktab

+ 国际化工具：native2ascii

+ 远程方法执行（RMI）工具：rmic,rmiregistry,rmid,serialver

+ Java IDL和RMI-IIOP 工具：tnameserv,idlj,ordb,servertool

+ Java Web 启动工具：javaws

+ Java Troubleshooting, Profiling, Monitoring and Management Tools:jcmd,jconsole,jmc,jvisualvm

+ Java Web Services Tools:schemagen,wsgen,wsimport,xjc

实验性的JDK工具：

+ 监控工具：jps,jstat,jstatd

+ 故障分析工具：jinfo,jhat,jmap,jsadebugd,jstack

+ 脚本工具：jrunscript

### 基本工具
| 工具名     | 简介     |
| :------------- | :------------- |
|   appletviewer   |  不用浏览器运行和debug applet     |
|   apt   | 注释处理工具      |
|   extcheck   | 检测jar包冲突的工具      |
|   jar   |  创建和管理jar包的工具     |
|   java   | java应用的启动器      |
|   javac   |  java语言的编译器     |
|    javadoc  | API 文档生成      |
|   javah   | C语言头文件和stub生成器。在写native 方法时使用      |
|   javap   | Class 文件反汇编器      |
|   jdb   |    java 调试器   |


### 安全工具
| 工具名 | 简介     |
| :------------- | :------------- |
|  keytool      |  管理keystore 和证书     |
|   jarsigner     |   生成和验证jar包的签名    |
|   policytool     |   管理 policy 文件的GUI工具    |
|   kinit     |   获取Kerberos v5 ticket的工具    |
|   klist     |       |
|   ktab     |       |


### 国际化工具
| 工具名 | 简介     |
| :------------- | :------------- |
| native2ascii       | 把文本转化为 Unicode Lantin-1      |


### 远程方法调用（RMI）工具
| 工具名 | 简介    |
| :------------- | :------------- |
|  rmic      |   为远程对象生成stub和skeleton    |
|  rmiregistry      |    远程对象注册服务   |
|  rmid      |  RMI activation system daemon     |
|  serialver      |   返回类的 serialVersionUID    |


### Java IDL 和 RMI-IIOP 工具
| 工具名 | 简介     |
| :------------- | :------------- |
|    tnameserv    |    提供到name service 的访问    |
|    idlj    |   Generates .java files that map an OMG IDL interface and enable an application written in the Java programming language to use CORBA functionality.     |
|    ordb    |        |
|     servertool   |  Provides ease-of-use interface for the application programmers to register, unregister, startup, and shutdown a server.      |


### Java Web Start Tools
| 工具名 | 简介     |
| :------------- | :------------- |
|   javaws   |   启动java web和设置各种可选项的命令行工具    |


### Java Troubleshooting, Profiling, Monitoring and Management Tools
| 工具名 | 简介     |
| :------------- | :------------- |
|   jcmd   |    JVM诊断命令行工具，发送诊断命令请求到JVM   |
|   jconsole   |   java监视和管理控制台，基于JMX的可视化工具    |
|   jmc   |   Java任务控制台客户端，包括监视和管理java应用的工具    |
|  jvisualvm    |   是目前为止发布的最强大的运行监视和故障处理程序，提供了内存和CPU性能分析，heap dump 分析，内存泄漏检测，访问MBeans，垃圾收集等功能，All in One    |


### Java Web Services Tools
| 工具名 | 简介     |
| :------------- | :------------- |
|   schemagen   |   XML 绑定的Schema生成器    |
|   wsgen   |    生成JAX-WS可移植工件的工具     |
|   wsimport   |  生成JAX-WS可移植工件的工具      |
|  xjc    |     XML 绑定的编译器  |


### 监控工具
| 工具名 | 简介     |
| :------------- | :------------- |
|   jps   |   JVM Process Status Tool,虚拟机进程状况工具    |
|   jstat   |   JVM Statistics Monitoring Tool，虚拟机统计信息监视工具   |
|   jstatd   |   JVM jstat Daemon，jstatd工具是一个RMI服务器应用程序，用于监视已安装的HotSpot Java虚拟机（JVM）的创建和终止，并提供一个接口，以允许远程监视工具附加到在本地主机上运行的JVM    |


### 故障排查工具
| 工具名 | 简介     |
| :------------- | :------------- |
|   jinfo   |   Configuration Info for Java,Java配置信息工具    |
|   jhat   |  Heap Dump Browser，虚拟机堆转储快照分析工具     |
|   jmap   |   Memory Map for Java，Java内存映像工具    |
|  jsadebugd    |    Serviceability Agent Debug Daemon for Java    |
|  jstack    |   Stack Trace for Java，Java堆栈跟踪工具     |


### 脚本工具
| 工具名 | 简介     |
| :------------- | :------------- |
|   jrunscript   |   Script shell for Java - Runs a script    |


### jstatd
Usage: jstatd [options]

options如下：

````
-nr              Do not attempt to create an internal RMI registry within the jstatd process when an existing RMI registry is not found.

-p  port         Port number where the RMI registry is expected to be found, or, if not found, created if -nr is not specified.

-n  rminame      Name to which the remote RMI object is bound in the RMI registry. The default name is JStatRemoteHost. If multiple       jstatd servers are started on the same host, the name of the exported RMI object for each server can be made unique by specifying this option. However, doing so will require that the unique server name be included in the monitoring client's hostid and vmid strings.

-Joption        Pass option to the java launcher called by javac. For example, -J-Xms48m sets the startup memory to 48 megabytes. It is a common convention for -J to pass options to the underlying VM executing applications written in Java.
````

| option | 含义     |
| :------------- | :------------- |
|    -nr   |  当一个存在的RMI 注册中心没有找到时，不会在jstatd进程创建一个内部的RMI注册中心     |
|    -p port   |   port是要寻找的RMI注册中心的端口号，如果没有找到，并且没有指定-nr，则创建一个注册中心     |
|    -n rminame   |   默认为JStatRemoteHost；如果同一台主机上同时运行了多个jstatd服务，rminame可以用于唯一确定一个jstatd服务；这里需要注意一下，如果开启了这个选项，那么监控客户端远程连接时，必须同时指定hostid及vmid，才可以唯一确定要连接的服务。     |
|   -Joption    |    传递选项到javac调用的java启动器    |

#### 安全性

jstatd服务只能监视具有适当的本地访问权限的JVM。因此jstatd进程与被监控的JVM必须运行在相同的用户权限中。但是有一些特殊的用户权限，如基于UNIX（TM）为系统的root用户，它有权限访问系统中所有JVM的资源，如果jstatd进程运行在这种权限中，那么它可以监视系统中的所有JVM，但是这也带来了额外的安全问题。

jstatd服务不会对客户端进行任何的验证，因此运行了jstatd服务的JVMs，网络上的任何用户的都具有访问权限，这种暴露不是我们所希望的，因此在启动jstatd之前本地安全策略必须要加以考虑，特别是在生产环境中或者是在不安全的网络环境中。

如果没有其他安全管理器被安装，jstatd服务将会安装一个RMISecurityPolicy的实例，因此需要在一个安全策略文件中指定，该策略文件必须符合的默认策略实施的策略文件语法。

下面的这个示例策略将允许jstatd服务具有JVM全部的访问权限：

````
grant codebase "file:${java.home}/../lib/tools.jar" {
   permission java.security.AllPermission;
};
````

注：此处策略中的java.home，和JAVA_HOME不是一个概念，童鞋们不要搞错了，此处的java.home指的是JRE的路径，这个是Java的系统属性，不需要手工指定，通常是这个jdk下面的jre路径,即可以认为${java.home}和${JAVA_HOME}/jre是等价，如果想查看这个变量的值，可以任意找一个运行着的Java应用，找到它的PID，然后通过如下jinfo命令查看就可以查看到java.home的值：

`jinfo ${pid} | grep java.home`

也可以在Java代码中通过如下方式获取到：

````
System.out.println(System.getProperty("java.home"))
````

将上面的策略内容拷贝一个文件中，文件名可以随意取，为了形象我们将文件名命名为jstatd.all.policy，文件存放的路径也可以随意，只有你当前登陆的用户具有访问权限就可以，然后执行以下命令就可以启动jstatd服务：

````
jstatd -J-Djava.security.policy=jstatd.all.policy
````

如果是在具有安全限制的环境中，jstatd的策略安全一定要设置得当，并且只允许受信任的服务器或者网络访问，以免遭受网络攻击，如果存在安全隐患，最好不要启动jstatd服务，就在本地使用jstat及jps等工具对JVM进行监控了。

示例

1. 使用内部RMI注册中心
下面这个示例演示了通过内部RMI注册中心启动jstatd，这个示例假设没有其它的服务绑定到默认的RMI注册中心端口（默认端口是1099）。

````
jstatd -J-Djava.security.policy=jstatd.all.policy  
````

>如果基于默认端口1099的RMI注册中心原来没有被启动过，那么上面运行的命令首先会启动端口为1099的RMI注册中心，然后再启动jstatd服务，此时即使jstatd停止了，RMI注册中心也不会停止；如果是再次执行上面的命令，就不会再次启动RMI注册中心，jstatd会直接注册到注册中心。

2. 使用外部的RMI注册中心
这个示例演示了使用一个外部的RMI注册中心来启动jstatd，如果默认的内部注册中心已经被启动了，下面的这个示例就会抛出“端口1099已经被占用”的异常，因为它尝试在1099端口启动外部RMI注册中心：

````
rmiregistry&jstatd -J-Djava.security.policy=all.policy  
````

这个示例演示了使用一个外部的RMI注册中心来启动jstatd，此注册中心的端口为2020：

````
rmiregistry 2020&jstatd -J-Djava.security.policy=all.policy -p 2020  
````

这个示例演示了使用一个外部的RMI注册中心来启动jstatd，此注册中心的端口为2020，并且绑定到RMI注册中心的名为AlternateJstatdServerName：

````
rmiregistry 2020&jstatd -J-Djava.security.policy=all.policy -p 2020 -n AlternateJstatdServerName  
````

3. 禁止内部RMI注册中心的创建
这个示例演示了jstatd在启动的时候，如果没有找到默认的RMI注册中心，也不会创建默认的注册中心。这个示例中如果没有RMI注册中心在运行，此示例就会报错，如果存在就会正常运行：  

````
jstatd -J-Djava.security.policy=all.policy -nr
````

4. 开启RMI日记记录
这个示例演示的是jstatd运行在开启了日志记录功能的RMI注册中，这个对于问题查找或监控服务状态非常有用：

````
jstatd -J-Djava.security.policy=all.policy -J-Djava.rmi.server.logCalls=true
````

5. 注册RMI时，指定hostname

````
jstatd -J-Djava.security.policy=jstatd.all.policy -J-Djava.rmi.server.hostname=192.168.0.23
````

参考：

1. [](http://docs.oracle.com/javase/7/docs/technotes/tools/share/jstatd.html)
2. [](http://mybeautiful.iteye.com/blog/1162555)
