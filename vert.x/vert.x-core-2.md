### Concurrent composition
`CompositeFuture.all` 接收多个 `futures`参数（最多6个）并且当所有的futures `succeed`时返回一个future，当至少一个future失败时返回`failed`。

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

操作并发运行，附加到返回的future上的`Handler`在composition完成时执行。当一个操作失败了（在参数中传递的futures中的一个被标记为失败），结果的future也被标记为失败。当所有的操作成功了，结果的future也成功完成。

可以选择的是，你可以传递一个future的list（也可以为空）:

````
CompositeFuture.all(Arrays.asList(future1, future2, future3));
````

`all` 一直等待直到所有的future成功（或者一个失败），然而`any`等待直到第一个成功的future。`CompositeFuture.any`接收多个future参数（最多6个），当其中一个future成功时，返回一个成功的future,当所有的future失败时返回失败。

````
CompositeFuture.any(future1, future2).setHandler(ar -> {
  if (ar.succeeded()) {
    // At least one is succeeded
  } else {
    // All failed
  }
});
````

一个future的list可以使用如下：

````
CompositeFuture.any(Arrays.asList(f1, f2, f3));
````

`join`等待直到所有的future完成，要么成功要么失败。`CompositeFuture.join`接收多个future参数（最多6个）,当所有的future成功时，返回一个成功的future,当所有future完成并且其中至少一个失败时，返回失败。

````
CompositeFuture.join(future1, future2, future3).setHandler(ar -> {
  if (ar.succeeded()) {
    // All succeeded
  } else {
    // All completed and at least one failed
  }
});
````

一个future的list可以如下使用：

````
CompositeFuture.join(Arrays.asList(future1, future2, future3));
````

### Sequential composition
`all`和`any`实现了并发 composition，`compose`可以用于chaining futrures(so Sequential composition).

````
FileSystem fs = vertx.fileSystem();
Future<Void> startFuture = Future.future();

Future<Void> fut1 = Future.future();
fs.createFile("/foo", fut1.completer());

fut1.compose(v -> {
  // When the file is created (fut1), execute this:
  Future<Void> fut2 = Future.future();
  fs.writeFile("/foo", Buffer.buffer(), fut2.completer());
  return fut2;
}).compose(v -> {
          // When the file is written (fut2), execute this:
          Future<Void> fut3 = Future.future();
          fs.move("/foo", "/bar", fut3.completer());
        },
        // mark the start future as completed when all the chain has been completed,
        // or mark it as failed if any step fails.
        startFuture);
````

在这个例子中，3个操作链起来了：

1. 一个文件被创建了（fut1）
2. 一些东西被写入到文件(fut2)
3. 文件被移动(fut3)

当这3个操作成功时，最后的future(`startFuture`)成功完成。但是如果其中一个步骤失败了，最后的future也会失败。

这个例子使用：

+ `compose`： 当当前的future完成时，执行给定的function,返回一个future。当这个返回的future完成时，他完成了composition.
+ `compose`： 当当前的future完成时，执行给定的handler完成给定的future(next).

这第二个例子中，`Handler`应该完成`next`future,以报告它的成功或失败。

you can use `completer`that completes a future with the operation result or failure。it avoids having to write the traditional: if success then complete the future else fail the future。

### Verticles
Vert.x 带来了一种简单的、可扩容的、类似于actor的部署方式，并且提供了并发的模型。这样你不用自己写。

**This model is entirely optional and Vert.x does not force you to create your applications in this way if you don’t want to..**

这个模型不一定要是严格的actor模型实现，但是他们确实在并发，扩容和部署方面有相似性。

为了使用这种模型，你需要写a set of `Verticles`。

Verticles 是代码块，这些代码块会被部署并且被Vert.x运行。一个Vert.x 实例维护了 N 个（默认是`2 * CPU`）eventloop 线程。Verticles可以用任何Vert.x 支持的语言写，一个应用也可以包括用多个语言写的Verticles。

你可以把Verticle 想象成 [Actor Model](https://en.wikipedia.org/wiki/Actor_model_theory)中的actor.

一个应用通常由很多Verticle实例组成，这些实例被同一个Vert.x 实例同时运行。不同的 Verticle实例彼此之间交流通过 在[event bus](http://vertx.io/docs/vertx-core/java/#event_bus)上发送消息。

### Writing Verticle
Verticle 类必须实现`Verticle`接口。

可以直接实现`Verticle`接口，但是通常继承抽象类`AbstractVerticle`更简单。

下面是一个例子：

````
public class MyVerticle extends AbstractVerticle {

  // Called when verticle is deployed
  public void start() {
  }

  // Optional - called when verticle is undeployed
  public void stop() {
  }

}
````


一般，你将重写`start`方法。

当 Vert.x 部署Verticle时，将会调用`start`方法，当这个方法完成时，Verticle被认为是启动了。

你也可以选择重写`stop`方法。这个方法将会在Verticle取消部署时被Vert.x 调用，当这个方法完成时，verticle被认为已经停止了。

### Asynchronous Verticle start and stop
有时，你想在Verticle启动时做一下费时的工作，但是你不想Verticle在这些工作完成时才部署。例如，你可能想在`start`方法里部署其他的verticle。

你不能阻塞等待`start`方法中其他的Verticle部署，因为这可能打破了[Golden Role](http://vertx.io/docs/vertx-core/java/#golden_rule).

那么你该怎么做呢？

方法是实现异步`start`方法。这个版本的方法接收一个`Future`作为参数。当这个方法返回时，Verticle将不认为已经deployed。

一段时间后，当你已经做完你需要做的事之后（比如，启动其他的Verticle），你可以调用future的complete或者fail方法。

例子如下：

````
public class MyVerticle extends AbstractVerticle {

  public void start(Future<Void> startFuture) {
    // Now deploy some other verticle:

    vertx.deployVerticle("com.foo.OtherVerticle", res -> {
      if (res.succeeded()) {
        startFuture.complete();
      } else {
        startFuture.fail(res.cause());
      }
    });
  }
}
````

相似的，有一个异步的`stop`方法。如果你想做一些费时的清理工作你可以用这个：

````
public class MyVerticle extends AbstractVerticle {

  public void start() {
    // Do something
  }

  public void stop(Future<Void> stopFuture) {
    obj.doSomethingThatTakesTime(res -> {
      if (res.succeeded()) {
        stopFuture.complete();
      } else {
        stopFuture.fail();
      }
    });
  }
}
````

INFO: 在Verticle的`stop` 方法，你不需要手动的undeploy 被一个Verticle启动的子Verticle。Vert.x 将会自动的undeploy 任何的子Verticle，当他们的父Verticle undeployed.

### Verticle Types
有三种不同类型的Verticles:

+ Standard Verticles
他们是最有用和最普通的类型，他们总是使用一个event loop 线程执行。

+ Worker Verticles
使用Worker Pool里的一个线程运行。一个实例不会被多个线程并发运行。

+ Multi-threaded Worker Verticles
使用Worker Pool中的一个线程运行。一个实例可以被多个线程并发运行。

### Standard Verticles
Standard Verticle当他们创建时被分配到一个event loop thread，并且`start`方法在这个event loop中被调用。当你从event loop里调用其他的接收一个handler为参数的方法时，Vert.x 将会保证这些handler，当被调用时，会在同一个event loop中执行。

这意味着我们可以保证Verticle实例中的所有代码总是在同一个event loop中执行（只要你不自己创建线程，并且调用它）。

这意味着你可以把你应用中的所有代码写为单线程，让Vert.x 担心线程和容量的事。不需要担心synchronized 和 volatile，你可以避免很多其他的race condition和dead lock的情况。

### Worker Verticles
一个worker Verticle 就像一个Standard Verticle，但是它不是用一个event loop 执行，而是用一个来自Vert.x Worker 线程池中的线程。

Worker Verticle被设计来执行阻塞代码，因为他们不会阻塞任何的event loop。

如果你不想用Worker Verticle运行阻塞代码，你可以在一个event loop上直接运行[inline blocking code](http://vertx.io/docs/vertx-core/java/#blocking_code)。

如果你想deploy 一个Verticle为Worker Verticle，你可以通过`setWorker`做到：

````
DeploymentOptions options = new DeploymentOptions().setWorker(true);
vertx.deployVerticle("com.mycompany.MyOrderProcessorVerticle", options);
````

Worker Verticle实例永远不会被Vert.x 多于一个线程并发的执行，但是可以在不同的时间被不同的线程执行。

### Multi-threaded worker Verticles
一个Multi-threaded worker Verticles就像一个普通的worker Verticle 但是可以被不同的线程并发的执行。

> WARNING
>
> Multi-threaded worker Verticles是一个高级特性，大多数应用并不需要他们。因为在这些Verticles的并发，你需要用Java中的多线程编程技术很小心的保持Verticle 的一致性状态。

### Deploying verticles programmatically
你可以使用`deployVerticle`方法中的一个来部署Verticle,指定一个Verticle 名字或你可以传递一个你自己创建的Verticle实例。

> NOTE
>
>Deploying Verticle instances is Java only.

````
Verticle myVerticle = new MyVerticle();
vertx.deployVerticle(myVerticle);
````

你也可以通过指定Verticle name部署Verticle。

Verticle 名字被用于寻找具体的`VerticleFactory`，它将被用于初始化真正的Verticle 实例。

不同的Verticle factory可用来初始化不同语言的verticles，也可用于其他的原因比如 loading services 和getting verticles from maven at run-time。

这允许你deploy Vert.x 支持的任何语言的Verticles。

这是deploy 不同类型的verticles的例子：

````
vertx.deployVerticle("com.mycompany.MyOrderProcessorVerticle");

// Deploy a JavaScript verticle
vertx.deployVerticle("verticles/myverticle.js");

// Deploy a Ruby verticle verticle
vertx.deployVerticle("verticles/my_verticle.rb");
````

### Rules for mapping a verticle name to a verticle factory
当使用名字deploy verticle时，这个名字被用于选择真正的verticle factory来初始化verticle。

Verticle name可以有一个前缀--它是一个string，后面跟着一个冒号。如果有前缀的话，会被用于选择verticle factory。

````
js:foo.js // Use the JavaScript verticle factory
groovy:com.mycompany.SomeGroovyCompiledVerticle // Use the Groovy verticle factory
service:com.mycompany:myorderservice // Uses the service verticle factory
````

如果没有前缀，Vert.x 将会寻找后缀，并且使用这个来找到factory,例如：

````
foo.js // Will also use the JavaScript verticle factory
SomeScript.groovy // Will use the Groovy verticle factory
````

如果没有任何前缀和后缀，Vert.x 将会假设它是Java的全限定类名，然后试着初始化。

### How are Verticle Factories located？
大多数Verticle 工厂是从classpath 加载的，在Vert.x 启动时注册。

你可以以编程方式注册和取消注册Verticle factory，使用`registerVerticleFactory`和`unregisterVerticleFactory`.

### Waiting for deployment to complete
Verticle deployment 是异步的，可能在调用 deploy返回之后的某时完成了。

如果你想在deployment完成之后得到通知，你可以在deploy时指定一个completion handler：

````
vertx.deployVerticle("com.mycompany.MyOrderProcessorVerticle", res -> {
  if (res.succeeded()) {
    System.out.println("Deployment id is: " + res.result());
  } else {
    System.out.println("Deployment failed!");
  }
});
````

如果deployment 成功了，completion handler将会被传递一个包含deployment id 字符串的Result。

这个deployment id可以用于之后的undeploy。

### Undeploying Verticle deployments
deployments 可以被undeploy,使用`undeploy`。

un-deployment 也是异步的，如果你想在un-deployment完成之后得到通知，你可以deploy 一个特定的completion handler.

````
vertx.undeploy(deploymentID, res -> {
  if (res.succeeded()) {
    System.out.println("Undeployed ok");
  } else {
    System.out.println("Undeploy failed!");
  }
});
````

### Specifying number of verticle instances
当使用一个verticle name deploying 一个verticle时，你可以指定你想deploy 的verticle instance 的数量：

````
DeploymentOptions options = new DeploymentOptions().setInstances(16);
vertx.deployVerticle("com.mycompany.MyOrderProcessorVerticle", options);
````

This is useful for scaling easily across multiple cores。For example you might have a web-server verticle to deploy and multiple cores on your machine, so you want to deploy multiple instances to take utilise all the cores.

### Passing configuration to a verticle
以json形式的配置可以在deploy时传递给verticle:

````
JsonObject config = new JsonObject().put("name", "tim").put("directory", "/blah");
DeploymentOptions options = new DeploymentOptions().setConfig(config);
vertx.deployVerticle("com.mycompany.MyOrderProcessorVerticle", options);
````

这个配置现在可以通过[Context](http://vertx.io/docs/apidocs/io/vertx/core/Context.html)对象配置，或直接用[config](http://vertx.io/docs/apidocs/io/vertx/core/AbstractVerticle.html#config--)方法。

这个配置是以JSON对象的形式返回，所以你可以像下面这样取出数据：

````
System.out.println("Configuration: " + config().getString("name"));
````

### Accessing environment variables in a Verticle
环境变量和系统属性可以通过Java API访问：

````
System.getProperty("prop");
System.getenv("HOME");
````

### Verticle Isolation Groups
默认的，Vert.x 有一个平坦的classpath(flat classpath)，比如，当Vert.x deploys Verticle 时，它使用当前的classloader--它不会创建一个新的classloader。在大多数情况下，这是最简单，最清楚，最明智的。

但是，在一些情况下，你可能想deploy一个Verticle,这个Verticle中的类和应用中的其他Verticle隔离。

例如，如果你想在同一个Vert.x 实例中用相同的类名deploy两个不同的版本的Verticle，或者你有两个不同的Verticle使用同一个jar包的不同版本。

当使用一个隔离group时，使用[setIsolatedClasses](http://vertx.io/docs/apidocs/io/vertx/core/DeploymentOptions.html#setIsolatedClasses-java.util.List-)你可以提供你想隔离的class list--一个可以使用诸如`com.mycompany.myproject.engine.MyClass`的全类名的条目，或者可以是一个通配符，它可以匹配任何package和sub-package中的所有类，比如：`com.mycompany.myproject.*`将会匹配package`com.mycompany.myproject`和sub-package中的所有类。

请注意，只有匹配的classes 才会被隔离--其他所有的classes 将会被当前的class loader加载。

另外的classpath 可以通过[setExtraClasspath](http://vertx.io/docs/apidocs/io/vertx/core/DeploymentOptions.html#setExtraClasspath-java.util.List-)提供，所以如果你想加载已经不再main classpath上的classes 或 resources，你可以通过这样。

> WARNING
>
>Use this feature with caution. Class-loaders can be a can of worms, and can make debugging difficult, amongst other things.

下面是使用隔离group来隔离一个Verticle deployment的例子：

````
DeploymentOptions options = new DeploymentOptions().setIsolationGroup("mygroup");
options.setIsolatedClasses(Arrays.asList("com.mycompany.myverticle.*",
                   "com.mycompany.somepkg.SomeClass", "org.somelibrary.*"));
vertx.deployVerticle("com.mycompany.myverticle.VerticleClass", options);
````

### High Availability
Verticles可以在部署时，打开High Availability.在这个Context,当一个Vert.x 实例上的一个已经部署的Verticle突然死了，这个Verticle将会在集群中的其他Vert.x 实例上重新部署。

为了在运行Verticle时打开high Availability,只需追加`-ha`开关：

````
vertx run my-verticle.js -ha
````

当打开high Availability时，就不需要追加`-cluster`了。

关于High Availability的更多细节和配置将在[High Availability and Fail-Over](http://vertx.io/docs/vertx-core/java/#_high_availability_and_fail_over)。

### Running Verticles from the command line
你可以在你的Maven 或者Gradle 工程中直接以普通的增加一个Vert.x Core的依赖的方式使用Vert.x。

但是，你也可以从命令行中直接运行Vert.x Verticles.

为了这样做，你需要下载和安装一个Vert.x 的发行版，将你的安装目录的bin目录加到`PATH`环境变量。同时，也要确保你的`PATH`上有一个Java 8 JDK。

> NOTE
>
>The JDK is required to support on the fly compilation of Java code.

你可以使用`vertx run`命令运行Verticles,下面是一些例子：

````
# Run a JavaScript verticle
vertx run my_verticle.js

# Run a Ruby verticle
vertx run a_n_other_verticle.rb

# Run a Groovy script verticle, clustered
vertx run FooVerticle.groovy -cluster
````

你甚至可以允许Java源码的Verticles而不用先编译他们：

````
vertx run SomeJavaSourceFile.java
````

Vert.x 将会在运行Java源码之前编译他们。这对于快速的原型和demo是很有用的。不需要先设置一个Maven 或 Gradle build。

关于在命令行执行`vertx`的各种选项的全部信息，你可以在命令行敲入：`vertx`。

### Causing Vert.x to exit
被Vert.x 实例维护的线程不是守护线程，所以他们会阻止JVM退出。

如果你嵌入了Vert.x 并且你已经用完了，你可以调用[close](http://vertx.io/docs/apidocs/io/vertx/core/Vertx.html#close--)来关闭它。

这将会关闭所有的内部线程池和其他的资源，这将允许JVM 退出。

### The Context Object
当Vert.x 提供一个事件给一个handler 或 启动和停止Verticle的调用时，这个执行和一个`Context`关联。通常，一个context是一个 **event-loop context**，绑定到一个具体的event loop thread。所以这个context的执行都是在同一个event loop thread。在Worker Verticles运行阻塞代码的情况下，一个worker context 将会和使用Worker thread pool的线程的执行关联。

为了获取context,使用`getOrCreateContext`方法：

````
Context context = vertx.getOrCreateContext();
````

如果当前线程有一个context 和它关联，它会重用这个context 对象。如果没有的话，context的一个新的实例将会创建。你可以测试你获取的context的类型：

````
Context context = vertx.getOrCreateContext();
if (context.isEventLoopContext()) {
  System.out.println("Context attached to Event Loop");
} else if (context.isWorkerContext()) {
  System.out.println("Context attached to Worker Thread");
} else if (context.isMultiThreadedWorkerContext()) {
  System.out.println("Context attached to Worker Thread - multi threaded worker");
} else if (! Context.isOnVertxThread()) {
  System.out.println("Context not attached to a thread managed by vert.x");
}
````

当你获取了一个context对象，你可以异步的在这个context中运行代码。用其他的话来说，你提交一个任务，它最终将会在同一个context中运行，但是是之后：

````
vertx.getOrCreateContext().runOnContext( (v) -> {
  System.out.println("This will be executed asynchronously in the same context");
});
````

当几个handler在同一个context运行，他们可能想共享数据。context对象提供了方法存储和获取context中共享的数据。例如，它让你传递数据给一些Action,通过`runOnContext`:

````
final Context context = vertx.getOrCreateContext();
context.put("data", "hello");
context.runOnContext((v) -> {
  String hello = context.get("data");
});
````

context对象也让你访问使用`config`方法配置的Verticle 配置。

### Executing periodic and delayed actions
在Vert.x 想执行一个延迟或周期性的动作是很普遍的。

在Standard Verticles中，你不可以让thread sleep来引入一个delay，因为这将阻塞event loop thread.

相反，你使用Vert.x timers。Timers 可以是one-shot 或 periodic。

##### One-shot Timers
一个一次性的Timer 在一个确定的delay之后调用一个evnet handler，用毫秒表示。

为设置一个一次性Timer,你可以使用`setTimer`方法传递一个延迟时间和一个handler。

````
long timerID = vertx.setTimer(1000, id -> {
  System.out.println("And one second later this is printed");
});

System.out.println("First this is printed");
````

返回值是一个唯一Timer Id，它可以稍后用于取消这个Timer.handler也传入了这个timer id。

##### Periodic Timers
你也可以设置个周期触发的定时器，使用[setPeriodic](http://vertx.io/docs/apidocs/io/vertx/core/Vertx.html#setPeriodic-long-io.vertx.core.Handler-)。

一个初始delay等于周期。

`setPeriodic`的返回值是一个唯一的timer id。这个Id可以稍后用于取消。

传递给timer event handler 的参数也是这个唯一的timer id。

记住，timer 将会周期触发。如果你的周期性处理花费了很长时间，你的timer event可能连续触发或者更糟：重叠。

在这种情况下,你可以考虑使用`setTimer`。一旦你的处理完成，你可以设置下一个timer。

````
long timerID = vertx.setPeriodic(1000, id -> {
  System.out.println("And every second this is printed");
});

System.out.println("First this is printed");
````

##### Cancelling timers
取消一个周期性任务，调用`cancelTimer`指定timer id。例如：

````
vertx.cancelTimer(timerID);
````

##### Automatic clean-up in verticles
如果你从内部verticles创建了timer,这些timer将会在verticle undeploy时自动关闭。

### Verticle worker pool
Verticle使用Vert.x worker pool来执行阻塞操作，比如[executeBlocking](http://vertx.io/docs/apidocs/io/vertx/core/Context.html#executeBlocking-io.vertx.core.Handler-boolean-io.vertx.core.Handler-)或worker verticle。

一个不同的worker pool可以在deployment 选项中指定：

````
vertx.deployVerticle("the-verticle", new DeploymentOptions().setWorkerPoolName("the-specific-pool"));
````

### The Event Bus
[event bus](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/EventBus.html)是Vert.x 的神经系统。

每一个Vert.x 实例有一个event bus实例，可以通过[eventbus](http://vertx.io/docs/apidocs/io/vertx/core/Vertx.html#eventBus--)方法获取。

event bus允许应用的不同部分彼此通信，无论他们使用什么语言写的，无论他们是否在同一个Vert.x 实例或者不同的Vert.x 实例。

它甚至可以允许运行在浏览器中的客户端的JavaScript彼此通信，在同一个event bus上。

event bus形成了一个分布式的端对端消息系统，跨越了多个server 节点和多个浏览器。

event bus支持publish/subscribe，point to point，以及request-response 消息传递。

event bus API是很简单的。它总的来说包括注册handler,取消注册handler,以及发送和发布消息。

首先，是一些理论知识：

#### The Theory
##### Addressing
消息发送到event bus是发送到一个地址。

在Vert.x 中一个地址是一个简单的字符串。任何字符串都是有效的。然而，使用一些scheme是明智的，例如，使用句号来标明一个命名空间。

一些有效的地址的例子是：europe.news.feed1, acme.games.pacman, sausages, and X.

##### Handlers
消息在handlers中接收。你在地址上注册一个handler。

很多不同的handler可以注册到同一个地址。

一个handler可以注册到很多不同的地址。

##### Publish/subscribe messaging
event bus支持publishing 消息。

消息发布到一个地址。发布意味着传递消息到注册到那个地址的所有handler。

这和 发布/订阅模式很像。

##### Point to Point and Request-Response messaging
event bus也支持point to point 消息传递。

消息被发送到一个地址。Vert.x 将会路由它到注册到那个地址的其中一个handler。

如果多于一个handler注册到那个地址，一个handler将被选出，使用不严格的round-robin 算法。

对于point to point 消息传递，在发送消息时一个可选的reply handler可以指定。

当一个消息被接收者接收到，并且已经处理了，接收者可以选择决定回复这个消息。如果他们回复了，reply handler将会被调用。

当发送者接收到回复时，他也可以回复。这样可能永远的回复，并且允许在两个不同的verticle之间设置一个对话框。

这就是常见的被称为Request-Response模式的消息传递模式。

##### Best-effort delivery
Vert.x 尽最大可能传递消息，不会故意丢失消息。这被称为Best-effort 传递。

然而，如果event bus的部分或全部failure，有可能消息会丢失。

如果你的应用在乎丢失消息，你handlery应该是幂等的，并且发送者应该在恢复后重试。

##### Types of messages
Vert.x 允许任何的原始/简单类型，String,或者是[buffers](http://vertx.io/docs/apidocs/io/vertx/core/buffer/Buffer.html)作为发送的消息。

但是，通常在Vert.x 中发送JSON 为消息更为方便。

在Vert.x 支持的语言中，JSON容易构造，读取，和解析，所以他变成了通用语。

但是，如果你不想用的话，你可以不用JSON.

event bus 非常灵活，也支持发送任意的对象。你可以通过为你想发送的对象定义一个[codec](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/MessageCodec.html)来做到。


### The Event Bus API
#### Getting the event bus
你可以像下面这样获取event bus的引用：

````
EventBus eb = vertx.eventBus();
````

每一个Vert.x 实例只有一个event bus实例。

#### Registering Handlers
注册一个handler的最简单的方式是使用[consumer](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/EventBus.html#consumer-java.lang.String-io.vertx.core.Handler-).下面是例子：

````
EventBus eb = vertx.eventBus();

eb.consumer("news.uk.sport", message -> {
  System.out.println("I have received a message: " + message.body());
});
````

当一个消息到达你的handler时，你的handler会被调用，[message](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/Message.html)将会传入。

`consumer()`方法返回的对象是[MessageConsumer](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/MessageConsumer.html)的实例。

这个对象可以之后用于unregister handler,or use the handler as a stream.

另外，你也可以使用[consumer](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/EventBus.html#consumer-java.lang.String-io.vertx.core.Handler-)返回一个没有设置handler的MessageConsumer，然后再设置handler.例如：

````
EventBus eb = vertx.eventBus();

MessageConsumer<String> consumer = eb.consumer("news.uk.sport");
consumer.handler(message -> {
  System.out.println("I have received a message: " + message.body());
});
````

当注册handler到一个集群event bus，需要花费一定时间来使得注册到达集群中所有的节点。

如果你想在他完成时得到通知，你可以在MessageConsumer对象上注册一个[completionHandler](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/MessageConsumer.html#completionHandler-io.vertx.core.Handler-)

````
consumer.completionHandler(res -> {
  if (res.succeeded()) {
    System.out.println("The handler registration has reached all nodes");
  } else {
    System.out.println("Registration failed!");
  }
});
````

#### Un-registering Handlers
要unregister 一个handler，调用[unregister](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/MessageConsumer.html#unregister--).

如果你在一个集群event bus上，Un-registering 可能花费一定时间传送到节点，如果不想在他完成时得到通知，用[unregister](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/MessageConsumer.html#unregister-io.vertx.core.Handler-)。

````
consumer.unregister(res -> {
  if (res.succeeded()) {
    System.out.println("The handler un-registration has reached all nodes");
  } else {
    System.out.println("Un-registration failed!");
  }
});
````

#### Publishing messages
Publishing 消息很简单。使用[publish](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/EventBus.html#publish-java.lang.String-java.lang.Object-),指定要发布到的地址。

````
eventBus.publish("news.uk.sport", "Yay! Someone kicked a ball");
````

这消息将会被传递到注册到地址的所有handler.

#### Sending messages
Sending 消息将会导致注册到目的地址的其中一个handler接收到消息。这是point to point 模式。这个handler使用不严格的round-robin 方式挑选。

你可以使用[send](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/EventBus.html#send-java.lang.String-java.lang.Object-)发送消息：

````
eventBus.send("news.uk.sport", "Yay! Someone kicked a ball");
````

#### Setting headers on messages
在event bus中发送的messages 也可以包含headers。

通过在send或publis时提供一个[DeliveryOptions](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/DeliveryOptions.html)，可以指定。

````
DeliveryOptions options = new DeliveryOptions();
options.addHeader("some-header", "some-value");
eventBus.send("news.uk.sport", "Yay! Someone kicked a ball", options);
````

#### Message Ordering
Vert.x 将会传递消息到任何特定的handler以他们从任何特定的sender发送的顺序。

#### The Message Object
