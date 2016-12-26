at the heart of Vert.x is a set of Java APIs that we call Vert.x Core.

[Repository](https://github.com/eclipse/vert.x)

Vert.x Core 为下列的事情提供了功能：

+ 写TCP客户端和服务端
+ 写HTTP客户端和服务端，包括支持WebSocket.
+ Event Bus
+ 共享数据，local maps 和 集群的分布式maps。
+ 周期性的和延迟的actions
+ 部署和取消部署 Verticles
+ Datagram Sockets
+ DNS 客户端
+ 访问文件系统
+ 高可用性
+ 集群

Core中的功能是相当低层次的-在这里，你不会找到像数据库访问，授权或者高层次的web功能-这些功能你将在`Vert.x ext`里找到。

Vert.x Core 很小并且轻量级。你可以只使用你需要的部分。它也可以完全嵌入到现存的应用中-我们没有强迫你以一种特殊的方式组织你的应用，才可以使用Vert.x

你可以从任何其他的 Vert.x 支持的语言来使用Core.

从现在开始，我们使用`Core`来称呼`Vert.x Core`.

如果你使用Maven 或 Gradle，添加下面的依赖到你的项目的描述的`dependencies`区来访问Vert.x Core API。

+ Maven (在pom.xml)：

````
<dependency>
  <groupId>io.vertx</groupId>
  <artifactId>vertx-core</artifactId>
  <version>3.3.3</version>
</dependency>
````

+ Gradle (在build.gradle)：

````
compile io.vertx:vertx-core:3.3.3
````

我们开始讨论Core中的不同概念和特性。

### In the beginning there was Vert.x
在Vert.x之地，你不能做太多的事，除非你可以和一个`Vertx`对象沟通。

它是Vert.x的控制中心，通过它你可以做其他的很多事。包括创建客户端和服务端，获得Event bus的引用，设置定时器，等等。

那么，我们如何获取一个实例呢？

如果你已经嵌入了Vert.x,你可以像下面一样简单的创建一个实例：

`Vertx vertx = Vertx.vertx();`

如果你正在使用 Verticles

> NOTE
>
>大多数应用仅仅需要一个Vert.x实例，但是如果你需要的话，也可以创建多个Vert.x实例，例如，event bus之间的隔离或不同组的服务端和客户端

### Specifying options when creating a Vertx Object
当创建一个`Vertx`对象时，如果默认的选项不符合你的话，你可以指定具体的选项：

`Vertx vertx = Vertx.vertx(new VertxOptions().setWorkerPoolSize(40));`

`VertxOptions`对象有很多设置允许你配置，比如集群，高可用性，池的大小，和很多其他的设置。Javadoc 详细描述了所有的设置。

### Creating a clustered Vert.x Object
如果你正在创建一个`clustered Vert.x`（查看[event bus](http://vertx.io/docs/vertx-core/java/#event_bus)中的区域，了解更多关于clustering event bus的信息）,那么你通常会使用异步的版本来创建`Vertx`对象。

这是因为一个集群中的不同`Vert.x`实例通常要花费一些时间（（也许是几秒钟)来组成群组。在这段时间内，我们不想阻塞调用线程，所以我们异步的给你调用的结果。

### Are you fluent
你可能在之前的例子中已经注意到，使用了fluent API。

fluent API是多个方法调用链接到一起。比如：

`request.response().putHeader("Content-Type","text/plain").write("some text").end();`

贯穿Vert.x API 中，这是一种普遍的模式，所以习惯它。

将调用像这样链接起来允许你写出的代码相对不是很啰嗦。当然，如果你不喜欢fluent 方法，我们也不强迫你这样做。你可以高兴的忽略它，如果你更喜欢像下面这样写代码：

````
HttpServerResponse response = request.response();
response.putHeader("Content-Type", "text/plain");
response.write("some text");
response.end();
````

### Don't call us,we'll call you
Vert.x API主要是事件驱动的。这意味着，在Vert.x 中当你感兴趣的事件发生时，Vert.x 将调用你发送事件给你。

一些实例事件如下：

+ 一个定时器被触发
+ socket上有数据到达
+ 一些数据从磁盘中读出
+ 发生异常
+ 一个HTTP Server接收到一个请求

通过提供 handlers给 Vert.x API，你可以处理事件。例如，为了每秒接收一个定时器事件，你可以这样做：

````
vertx.setPeriodic(1000, id -> {
  // This handler will get called every second
  System.out.println("timer fired!");
});
````

一段时间之后，当Vert.x有一个事件要传递给你的handler时，Vert.x 将异步的调用它。

这引导我们Vert.x 中的一些重要的概念:

### Don't block me！
除了极少数一些例外（比如文件系统操作在`Sync`上等待结束）,Vert.x中的任何API都不会阻塞调用线程。

如果结果可以立即被提供，它将会立即返回，否则，你将通常提供一个handler来接收稍后的事件。

因为没有Vert.x API会阻塞线程，这意味着你可以使用Vert.x 只使用少量线程来处理大量并发。

传统的阻塞API将阻塞调用线程，当：

+ 从socket中读取数据
+ 写入数据到磁盘
+ 发送数据到一个接收者，并且等待答复
+ 。。。很多其他的情况

在上面所有的例子中，当你的线程在等待一个结果，它就不能做其他的事了。

这意味着，如果你想大量并发的使用阻塞API，你需要大量的线程来阻止你的应用完全停下来。

线程有开支，考虑到他们需要的内存（比如他们的栈）和线程上下文切换。

For the levels of concurrency required in many modern applications, a blocking approach just doesn’t scale.

### Reactor and Multi-Reactor
我们在之前已经提到过 Vert.x的API是事件驱动的--当它们可用时，Vert.x 传递事件到handler。

在大多数例子中，Vert.x 使用一个被称为`event loop`的线程调用你的handler.
