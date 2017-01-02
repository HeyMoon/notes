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

因为 Vert.x或者应用中的任何事都不会阻塞，event loop可以愉快的四处运行连续的传递事件到不同的handler，当事件到达时。

因为没有事情会阻塞，一个event loop有在很短时间内传递大量事件的潜能。例如，单个event loop可以很快的处理上成千上万个HTTP 请求。

我们把这叫做[Reactor Pattern](https://en.wikipedia.org/wiki/Reactor_pattern)

你可能已经听说过它--例如 Node.js 实现了这种模式

在一个标准的reactor实现中，有一个event loop线程在循环中运行，在事件到来时传递所有的事件到所有的handler。

单个线程的麻烦是，在任意时间它只能运行一个Core,所以，如果你想要你的单线程reactor 应用scale over你的多core 服务端，你需要启动和管理多个不同的进程。

Vert.x 在这方面不同。每一个Vertx 实例维护了几个event loop，而不是一个。默认的，我们选择event loop的数量是基于机器上可用的CUP的数量，不过，这是可以重载的。

这意味着，一个 Vertx 进程可以scale across your server,不像Node.js。

我们称这种模式为`Multi-Reactor Pattern`，为了区分单线程的reactor Pattern.

> NOTE
>
>即使一个 Vertx实例维护了多个event loop，任何一个具体的hanlder 都不可能并发执行，并且在大多数情况下(除了 [worker verticles](http://vertx.io/docs/vertx-core/java/#worker_verticles))，被调用时，总是使用同一个event loop。

### The Golden Rule - Don't Block the Event Loop
我们已经知道 Vert.x 的API是非阻塞的，并且不会阻塞 evnet loop，但是如果你自己在handler 中阻塞了event loop，这并没有多大的帮助。

如果你这样做了（在handler中阻塞了event loop）,那么当被阻塞时，那个event loop将不能做其他事。如果你阻塞了Vertx实例中的所有event loop,那么你的应用将完全停止。

所以不要这样做！**你已经被警告了.**

阻塞的例子包括：

+ Thread.sleep()
+ 在一个锁(lock)上等待
+ 在一个独占锁或监视器上等待（比如 synchronized 代码块）
+ 执行一个长时间的数据库操作，并且等待结果
+ 执行一个复杂的计算，花费了很多时间。
+ 在循环中自旋

如果上面的任何一个在`很长的时间`内阻止event loop做其他的事，then you should go immediately to the naughty step,and await further instructions.

那么什么是`很长的时间`呢？

一节绳子有多长？ 这实际上取决于你的应用和你要求的并发数。

如果你只有一个event loop，你想要每秒处理10000个HTTP 请求，那么很显然每一个请求的处理时间不能大于0.1 ms，所以你不能阻塞比这长的时间。

**The maths is not hard and shall be left as an exercise for the reader。**

如果你的应用不响应了，这可能是你在某些地方阻塞了一个event loop的标志。为了帮助你识别这些情况， 如果检测到一个event loop在某些时间内没有返回，Vert.x 将会自动的记录 `warnings`。如果你在日志里看到这类`warnings`，你应该调查一下。

````
Thread vertx-eventloop-thread-3 has been blocked for 20458 ms
````

Vert.x 也将会提供堆栈信息来准确的找到阻塞发生的地方。

如果你想关闭这些`warnings`或改变设置，你可以在创建`Vertx`对象之前，在`VertxOptions`对象里设置。

### Running blocking code
在一个完美的世界里将不会有战争或饥饿，所有的API都是异步的，在暖阳下，小兔子和小羊羔手牵手的走在绿色的草地上。

但是，现实世界不是这样的（你最近没看新闻吗？）

事实是，很多，如果不是大多数库，特别是在JVM 生态系统有同步API，并且很多的方法都是阻塞的。一个好的例子是 JDBC API--它内在的是阻塞的，无论它如何尽力的尝试，Vert.x 不能够撒一些魔法灰来使得它变成异步。

我们不打算重写一切来使得他一夜间变成异步的，所以我们需要提供一种方式在 Vert.x 应用中安全的使用传统的阻塞API。

正如之前讨论的，你不能在event loop里直接调用阻塞操作，因为它将会阻止我们做其他有用的工作。那么我们要怎么做呢？

这是通过调用`executeBlocking`指定需要执行的阻塞的代码和阻塞代码被执行后异步回调的result handler。如下所示：

````
vertx.executeBlocking(future -> {
  // Call some blocking API that takes a significant amount of time to return
  String result = someAPI.blockingMethod("hello");
  future.complete(result);
}, res -> {
  System.out.println("The result is: " + res.result());
});
````

默认的，如果`executeBlocking`在同一个context（例如，同一个 Verticle实例）里被调用多次，那么不同的`executeBlocking`将串行的被执行。

如果你不关心顺序，你可以在调用`executeBlocking`时指定参数`ordered`为`false`。在这种情况下，`executeBlocking`可能会在worker pool里并行执行。

一种可选的 运行阻塞代码的方式是使用[woker Verticle](http://vertx.io/docs/vertx-core/java/#worker_verticles)。

一个worker Verticle总是被worker pool中的一个线程执行。

默认的，阻塞代码仔 Vert.x 的阻塞代码池中执行，配置为`setWorkerPoolSize`。

其他的池也可以为了而不同的目的被创建：

````
WorkerExecutor executor = vertx.createSharedWorkerExecutor("my-worker-pool");
executor.executeBlocking(future -> {
  // Call some blocking API that takes a significant amount of time to return
  String result = someAPI.blockingMethod("hello");
  future.complete(result);
}, res -> {
  System.out.println("The result is: " + res.result());
});
````

当不在需要时，worker executor 必须被关闭。

````
executor.close();
````

当几个worker使用同一个名字被创建时，他们将会共享同一个pool。当使用worker pool的所有worker executor被关闭时，worker pool被摧毁（destroyed）。

当一个executor在一个Verticle中被创建时，Vert.x 将会在 Verticle取消部署时自动关闭executor.

worker executor可以在被创建时配置：

````
int poolSize = 10;

// 2 minutes
long maxExecuteTime = 120000;

WorkerExecutor executor = vertx.createSharedWorkerExecutor("my-worker-pool", poolSize, maxExecuteTime);
````

> NOTE
>
>当worker pool创建时，配置被设置。

### Async coordination
多个异步result 的协同可以通过 Vert.x 的`futures`做到。它支持concurrent composition（并行运行多个异步操作）和顺序composition（chain async operations）

### concurrent composition
`CompositeFuture.all`可以传递多个参数(最多6个),当所有的futures成功时，返回一个future，当至少一个futures失败时，返回failed。

````
Future<HttpServer> httpServerFuture = Future.future();
httpServer.listen(httpServerFuture.completer());

Future<NetServer> netServerFuture = Future.future();
netServer.listen(netServerFuture.completer());

CompositeFuture.all(httpServerFuture, netServerFuture).setHandler(ar -> {
  if (ar.succeeded()) {
    // All servers started
  } else {
    // At least one server failed
  }
});
````

操作并发运行，
