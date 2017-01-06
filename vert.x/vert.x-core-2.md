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
在message handler中接收到的对象是[message](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/Message.html)。


message 中的[body](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/Message.html#body--)对应你发送或发布的对象。

message中的headers可以通过[headers](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/Message.html#headers--)获取.

#### Acknowledging messages/Sending replies
当使用[send](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/EventBus.html#send-java.lang.String-java.lang.Object-)时，event bus想传递message 到注册到event bus的一个[MessageConsumer](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/MessageConsumer.html)。

在某些情况下，对于发送者来说，知道consumer已经接收到消息并且处理了是有用的。

为了确认消息已经处理了，consumer可以调用[reply](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/Message.html#reply-java.lang.Object-)回复消息。

当这发生时（调用reply回复了），它导致一个回复发送到sender，然后reply handler被调用。

一个例子将会使得这很清楚：

接收者：

````
MessageConsumer<String> consumer = eventBus.consumer("news.uk.sport");
consumer.handler(message -> {
  System.out.println("I have received a message: " + message.body());
  message.reply("how interesting!");
});
````

发送者：

````
eventBus.send("news.uk.sport", "Yay! Someone kicked a ball across a patch of grass", ar -> {
  if (ar.succeeded()) {
    System.out.println("Received reply: " + ar.result().body());
  }
});
````

reply可以包含一个包含有用信息的body。

处理的含义由应用定义，并且完全取决于message consumer，Vert.x event bus不关心。

一些例子：

+ A simple message consumer which implements a service which returns the time of the day would acknowledge with a message containing the time of day in the reply body

+ A message consumer which implements a persistent queue, might acknowledge with true if the message was successfully persisted in storage, or false if not.

+ A message consumer which processes an order might acknowledge with true when the order has been successfully processed so it can be deleted from the database

#### Sending with timeouts
当发送一个有reply handler的消息时，你可以在[DeliveryOptions](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/DeliveryOptions.html)指定一个超时时间。

如果在哪个时间内没有收到reply，reply handler将会以失败调用。

默认的超时时间是30 s。

#### Send Failures
消息发送可能失败，包括如下原因：

+ There are no handlers available to send the message to
+ The recipient has explicitly failed the message using `fail`

在所有的情况中，reply handler将会以一个特定失败调用。

#### Message Codecs
你可以发送你想发送的任何对象到event bus，如果你定义和为这个对象注册了一个[message codec](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/MessageCodec.html)。

Message codecs有一个名字，在send或publih时你可以在[DeliveryOptions](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/DeliveryOptions.html)中指定这个名字。

````
eventBus.registerCodec(myCodec);

DeliveryOptions options = new DeliveryOptions().setCodecName(myCodec.name());

eventBus.send("orders", new MyPOJO(), options);
````

如果你对于某一个特定的类型总是想使用同一个codec，你可以为他注册一个默认的codec，然后你不用在每一个send的 delivery options上指定codec。

````
eventBus.registerDefaultCodec(MyPOJO.class, myCodec);

eventBus.send("orders", new MyPOJO());
````

使用[unregisterCodec](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/EventBus.html#unregisterCodec-java.lang.String-)unregister a message codec。

#### Clustered Event Bus
event Bus不仅仅存在于一个Vert.x 实例。通过集群不同的Vert.x 实例，我们可以形成一个分布式的event bus。

#### Clustering programmatically
如果你在以编程方式创建你的Vert.x 实例，你可以通过配置 Vert.x 实例为集群来获取一个集群event bus。

````
VertxOptions options = new VertxOptions();
Vertx.clusteredVertx(options, res -> {
  if (res.succeeded()) {
    Vertx vertx = res.result();
    EventBus eventBus = vertx.eventBus();
    System.out.println("We now have a clustered event bus: " + eventBus);
  } else {
    System.out.println("Failed: " + res.cause());
  }
});
````

你应该确保在你的classpath下有一个[ClusterManager](http://vertx.io/docs/apidocs/io/vertx/core/spi/cluster/ClusterManager.html)的实现，例如，默认的`HazelcastClusterManager`。

#### Clustering on the command line
你可以在命令行中运行Vert.x 集群，使用如下的命令：

````
vertx run my-verticle.js -cluster
````

#### Automatic clean-up in Verticles
如果你从内部verticles注册了一个event bus handler，这些handler将会在verticle undeploy时自动unregister。

### Configureing the event bus
event bus可以配置。当event bus是集群时，这通常时很有效的。在底层，event bus使用TCP 连接发送和接收消息，所以[EventBusOptions](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/EventBusOptions.html)让你配置TCP连接的所有方面。正如event bus可以做为客户端和服务端，配置也分为[NetClientOptions](http://vertx.io/docs/apidocs/io/vertx/core/net/NetClientOptions.html)和[NetServerOptions](http://vertx.io/docs/apidocs/io/vertx/core/net/NetServerOptions.html)。

````
VertxOptions options = new VertxOptions()
    .setEventBusOptions(new EventBusOptions()
        .setSsl(true)
        .setKeyStoreOptions(new JksOptions().setPath("keystore.jks").setPassword("wibble"))
        .setTrustStoreOptions(new JksOptions().setPath("keystore.jks").setPassword("wibble"))
        .setClientAuth(ClientAuth.REQUIRED)
    );

Vertx.clusteredVertx(options, res -> {
  if (res.succeeded()) {
    Vertx vertx = res.result();
    EventBus eventBus = vertx.eventBus();
    System.out.println("We now have a clustered event bus: " + eventBus);
  } else {
    System.out.println("Failed: " + res.cause());
  }
});
````

前一小段代码描绘了你可以使用给event bus使用SSL connections，包括plain TCP connections。

**WARNING**：为了加强集群模式下的安全性，你必须配置使用加密配置集群管理。参考文档中的集群管理，了解更多细节。

event bus配置需要在集群中的所有节点保持一致。

[EventBusOptions](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/EventBusOptions.html)也允许你指定event bus 是否是集群，端口和主机，正如你可以通过[setClustered](http://vertx.io/docs/apidocs/io/vertx/core/VertxOptions.html#setClustered-boolean-),[getClusterHost](http://vertx.io/docs/apidocs/io/vertx/core/VertxOptions.html#getClusterHost--),[getClusterPort](http://vertx.io/docs/apidocs/io/vertx/core/VertxOptions.html#getClusterPort--)。

当在容器中使用时，你可以配置公共的host 和 port:

````
VertxOptions options = new VertxOptions()
    .setEventBusOptions(new EventBusOptions()
        .setClusterPublicHost("whatever")
        .setClusterPublicPort(1234)
    );

Vertx.clusteredVertx(options, res -> {
  if (res.succeeded()) {
    Vertx vertx = res.result();
    EventBus eventBus = vertx.eventBus();
    System.out.println("We now have a clustered event bus: " + eventBus);
  } else {
    System.out.println("Failed: " + res.cause());
  }
});
````

### JSON
不像其他语言，Java没有对JSON的一流支持，所以我们提供两个类使得在你的Vert.x 应用中处理JSON更加简单。

#### JSON Objects
[JsonObject](http://vertx.io/docs/apidocs/io/vertx/core/json/JsonObject.html)表示JSON 对象。

一个JSON对象本质上是一个有键值对的map.

JSON 对象也支持null值。

##### Creating JSON Objects
空的JSON对象可以以默认的构造器创建。

你可以从一个string中创建一个JSON 对象：

````
String jsonString = "{\"foo\":\"bar\"}";
JsonObject object = new JsonObject(jsonString);
````

##### Putting entries into a JSON object
使用[put](http://vertx.io/docs/apidocs/io/vertx/core/json/JsonObject.html#put-java.lang.String-java.lang.Enum-)方法put 值到JSON 对象。

方法调用可以chained 因为fluent API:

````
JsonObject object = new JsonObject();
object.put("foo", "bar").put("num", 123).put("mybool", true);
````

##### Getting values from a JSON object
你可以使用`getXXX`方法从JSON对象中获取值，例如：

````
String val = jsonObject.getString("some-key");
int intVal = jsonObject.getInteger("some-other-key");
````

##### Encoding the JOSN object to a String
使用[encode](http://vertx.io/docs/apidocs/io/vertx/core/json/JsonObject.html#encode--)编码一个对象为String.

#### JSON Arrays
[JsonArray](http://vertx.io/docs/apidocs/io/vertx/core/json/JsonArray.html)表示JSON Arrays.

一个JSON Array是一序列的值(string,number,boolean)

Json Array也可以包含null值。

##### Creating Json Arrays
可以使用默认的构造函数创建空的Json 对象。

你可以从一个string中创建一个JSON Array，如下所示：

````
String jsonString = "[\"foo\",\"bar\"]";
JsonArray array = new JsonArray(jsonString);
````

##### Adding entries into a JSON Array
你可以add entries到一个Json Array，使用[add](http://vertx.io/docs/apidocs/io/vertx/core/json/JsonArray.html#add-java.lang.Enum-)方法。

````
JsonArray array = new JsonArray();
array.add("foo").add(123).add(false);
````

##### Getting values from a Json array
你可以使用`getXXX`方法从JSON array对象中获取值，例如：

````
String val = array.getString(0);
Integer intVal = array.getInteger(1);
Boolean boolVal = array.getBoolean(2);
````

##### Encoding the JSON array to a String
使用[encode](http://vertx.io/docs/apidocs/io/vertx/core/json/JsonArray.html#encode--)方法，将一个array encode为String。

### Buffers
Vert.x 中的大多数数据移动使用buffers。

一个buffer是一序列的零个或多个bytes，可以写入或从中读取，根据需要自动扩容来适应写入的bytes。你也可以把buffer想象为智能byte array。

#### Creating buffers
使用[Buffer.buffer](http://vertx.io/docs/apidocs/io/vertx/core/buffer/Buffer.html#buffer--)静态方法中的一个可以创建Buffers。

Buffers可以从string 或 byte array中初始化，也可以创建空的buffers。

下面是创建Buffer的例子：

Create a new empty buffer:

````
Buffer buff = Buffer.buffer();
````

Create a buffer from a string.String 将会用UTF-8编码：

````
Buffer buff = Buffer.buffer("some string");
````

Create a buffer from a String:String will be encoded 使用指定的编码，例如：

````
Buffer buff = Buffer.buffer("some string", "UTF-16");
````

Create a buffer from a byte[]

````
byte[] bytes = new byte[] {1, 3, 5};
Buffer buff = Buffer.buffer(bytes);
````

创建一个具有初始大小的buffer。如果你知道你的buffer有一个确定数量的数据写入，你可以创建一个指定大小的buffer。这使得buffer初始分配足够的内存，比起buffer随着写入的数据自动扩容多次要更有效。

记住，这样创建的buffer是empty。它没有创建一个buffer，用零填充到指定的大小。

````
Buffer buff = Buffer.buffer(10000);
````

##### Writing to a Buffer
有两种方式写入到一个buffer：appending，random access。在每种情况下，buffer将总是自动扩充来包含bytes。buffer不可能出现`IndexOutOfBoundsException`。

##### Appending to a Buffer
你可以使用`appendXXX`方法，append 到一个buffer。Append 方法可以appending 多种不同类型。

`appendXXX`方法的返回值是buffer，所以可以chained:

````
Buffer buff = Buffer.buffer();

buff.appendInt(123).appendString("hello\n");

socket.write(buff);
````

##### Random access buffer writes
你也可以写入到buffer中的特定index，通过使用`setXXX`方法。Set 方法也可以为多种不同的数据类型。所有的set 方法接收一个index 作为第一个参数-这代表buffer中开始写入的index。

buffer总是会在必要时自动扩容来容纳数据。

````
Buffer buff = Buffer.buffer();

buff.setInt(1000, 123);
buff.setString(0, "hello");
````

##### Reading from a Buffer
使用`getXXX`方法可以从buffer中读取数据。有各种数据类型的get方法。第一个参数是一个index，从buffer中的这个位置开始读取数据。

````
Buffer buff = Buffer.buffer();
for (int i = 0; i < buff.length(); i += 4) {
  System.out.println("int value at " + i + " is " + buff.getInt(i));
}
````

##### Working with unsigned numbers
无符号数可以通过`getUnsignedXXX`,`appendUnsignedXXX`,`setUnsignedXXX`方法读取或append/set。在为一个网络协议实现一个codec时，最小化带宽消耗是很有用的。

在下面的例子中，值200在特定的位置被设置，只占用一个byte：

````
Buffer buff = Buffer.buffer(128);
int pos = 15;
buff.setUnsignedByte(pos, (short) 200);
System.out.println(buff.getUnsignedByte(pos));
````

##### Buffer length
使用[length](http://vertx.io/docs/apidocs/io/vertx/core/buffer/Buffer.html#length--)获取buffer的length。

##### Copying buffers
使用[copy](http://vertx.io/docs/apidocs/io/vertx/core/buffer/Buffer.html#copy--)复制buffer.

##### Slicing buffers
一个sliced buffer是一个在原buffer上创建的新的buffer，例如，他不会copy 底层数据。使用[slice](http://vertx.io/docs/apidocs/io/vertx/core/buffer/Buffer.html#slice--)创建一个sliced buffer。

##### Buffer re-use
在将buffer写入到socket 或类似的地方后，他们不能被重用。

### Writing TCP servers and clients
Vert.x 允许你很容易的写非阻塞的TCP 客户端和服务端。

#### Creating a TCP server
最简单的使用所有默认选项的创建TCP server的方式如下所示：

````
NetServer server = vertx.createNetServer();
````

#### Configuring a TCP server
通过[NetServerOptions](http://vertx.io/docs/apidocs/io/vertx/core/net/NetServerOptions.html)配置server.

````
NetServerOptions options = new NetServerOptions().setPort(4321);
NetServer server = vertx.createNetServer(options);
````

#### Start the Server Listening
告诉server监听到来的请求，你可以使用[listen](http://vertx.io/docs/apidocs/io/vertx/core/net/NetServer.html#listen--)方法中的一个。

为了告诉server在options上指定的host和port上监听：

````
NetServer server = vertx.createNetServer();
server.listen();
````

或，在调用listen 是指定host和port，忽略options中的配置：

````
NetServer server = vertx.createNetServer();
server.listen(1234, "localhost");
````

默认的host是`0.0.0.0`,它意味者在所有可用的地址上监听，默认的port是`0`，这个特殊的值告诉server找到一个随机的未使用的本地端口，然后使用这个端口。


真正的绑定是异步的，所以server可能实际上没有监听直到调用listen返回后的一段时间。

如果你想在server实际监听时得到一个通知，你可以在调用listen是提供一个handler.例如：

````
NetServer server = vertx.createNetServer();
server.listen(1234, "localhost", res -> {
  if (res.succeeded()) {
    System.out.println("Server is now listening!");
  } else {
    System.out.println("Failed to bind!");
  }
});
````

#### Listening on random port
如果`0`被使用作为监听端口，server将找到一个未被使用的随机的端口来监听。

为了找到server监听的真实端口，调用[actualPort](http://vertx.io/docs/apidocs/io/vertx/core/net/NetServer.html#actualPort--)。

````
NetServer server = vertx.createNetServer();
server.listen(0, "localhost", res -> {
  if (res.succeeded()) {
    System.out.println("Server is now listening on actual port: " + server.actualPort());
  } else {
    System.out.println("Failed to bind!");
  }
});
````

#### Getting notified of incoming connections
为了在一个连接被创建时得到通知，你需要设置一个[connectHandler](http://vertx.io/docs/apidocs/io/vertx/core/net/NetServer.html#connectHandler-io.vertx.core.Handler-):

````
NetServer server = vertx.createNetServer();
server.connectHandler(socket -> {
  // Handle the connection in here
});
````

当一个连接被创建时，handler将会被调用，传入[NetSocket](http://vertx.io/docs/apidocs/io/vertx/core/net/NetSocket.html)的实例。

这是一个到实际连接的像socket 的接口，允许你读取和写入数据还有很多其他事情，比如关闭socket。

#### Reading data from the socket
为了从socket中读取出数据，你可以在socket 上设置一个[handler](http://vertx.io/docs/apidocs/io/vertx/core/net/NetSocket.html#handler-io.vertx.core.Handler-)。

这个handler将会在每次数据到达socket时调用，以[Buffer](http://vertx.io/docs/apidocs/io/vertx/core/buffer/Buffer.html)的实例为参数。

````
NetServer server = vertx.createNetServer();
server.connectHandler(socket -> {
  socket.handler(buffer -> {
    System.out.println("I received some bytes: " + buffer.length());
  });
});
````

#### Writing data to a socket
使用[write](http://vertx.io/docs/apidocs/io/vertx/core/net/NetSocket.html#write-io.vertx.core.buffer.Buffer-)方法中的一个写数据到socket。

````
Buffer buffer = Buffer.buffer().appendFloat(12.34f).appendInt(123);
socket.write(buffer);

// Write a string in UTF-8 encoding
socket.write("some data");

// Write a string using the specified encoding
socket.write("some data", "UTF-16");
````

写操作是异步的，可能在调用write方法返回之后的一段时间才发生。

#### Closed Handler
如果你想在一个socker 被关闭时得到通知，你可以在它上面设置一个[closeHandler](http://vertx.io/docs/apidocs/io/vertx/core/net/NetSocket.html#closeHandler-io.vertx.core.Handler-):

````
socket.closeHandler(v -> {
  System.out.println("The socket has been closed");
});
````

#### Handling Exceptions
你可以设置一个[exceptionHandler](http://vertx.io/docs/apidocs/io/vertx/core/net/NetSocket.html#exceptionHandler-io.vertx.core.Handler-)来接收发生在socket上的任何异常。

#### Event bus write handler
每一个socket 自动的注册一个handler 到event bus，并且当这个handler接收到任何的buffers，它将buffers写到自己。

这使得你可以将数据写入到一个socket,这个socket可能在一个完全不同的verticle，或者甚至在一个不同的Vert.x 实例，通过发送buffer到这个handler的地址。

handler的地址由[writeHandlerID](http://vertx.io/docs/apidocs/io/vertx/core/net/NetSocket.html#writeHandlerID--)给出。

#### Local and remote addresses
一个`NetSocket`的本地地址可以使用[localAddress](http://vertx.io/docs/apidocs/io/vertx/core/net/NetSocket.html#localAddress--)获取。

一个`NetSocket`的远程地址可以通过[remoteAddress](http://vertx.io/docs/apidocs/io/vertx/core/net/NetSocket.html#remoteAddress--)获取。

#### Sending files or resources from the classpath
文件和classpath resources可以直接写入到socket，使用[sendFile](http://vertx.io/docs/apidocs/io/vertx/core/net/NetSocket.html#sendFile-java.lang.String-)。这是一种很有效的发送文件的方式，因为他可以由支持被操作系统支持的操作系统内核支持。

请看关于[serving files from the classpath](http://vertx.io/docs/vertx-core/java/#classpath)的章节，了解classpath 解决方案的约束。

```
socket.sendFile("myfile.dat");
```

#### Streaming sockets
[NetSocket](http://vertx.io/docs/apidocs/io/vertx/core/net/NetSocket.html)的实例也是`ReadStream`和`WriteStream`的实例，所以他们也可以被用于写入或从其他read/write stream 中读取数据。

更多详细信息，请看[streams and pumps](http://vertx.io/docs/vertx-core/java/#streams)

#### Upgrading connections to SSL/TLS
一个非SSL/TLS连接可以升级为SSL/TLS，使用[upgradeToSsl](http://vertx.io/docs/apidocs/io/vertx/core/net/NetSocket.html#upgradeToSsl-io.vertx.core.Handler-)。

更多信息，请看[SSL/TLS](http://vertx.io/docs/vertx-core/java/#ssl)章节。

#### Closing a TCP Server
调用[close](http://vertx.io/docs/apidocs/io/vertx/core/net/NetServer.html#close--)关闭Server。关闭server，关闭任何打开的连接，释放所有资源。

close是异步的，在调用返回后可能还没有完成。如果你想在真正的关闭完成时得到通知，你可以设置一个handler.

这个handler将会在关闭完全完成时被调用。

````
server.close(res -> {
  if (res.succeeded()) {
    System.out.println("Server is now closed");
  } else {
    System.out.println("close failed");
  }
});
````

#### Automatic clean-up in verticles
如果你从内部verticles创建了TCP servers和clients，这些servers和clients会在verticle undeploy时自动关闭。

#### Scaling-sharing TCP servers
任何TCP server的handler总是在同一个event loop thread里执行。

这意味着，如果你的server运行在多核上，并且你只有一个实例部署，那么你最多只会利用server上的一个CPU。

为了利用server上的更多核，你将需要在server上部署更多的实例。

你可以以编程方式初始化更多的实例：

````
for (int i = 0; i < 10; i++) {
  NetServer server = vertx.createNetServer();
  server.connectHandler(socket -> {
    socket.handler(buffer -> {
      // Just echo back the data
      socket.write(buffer);
    });
  });
  server.listen(1234, "localhost");
}
````

或者，如果你正在使用verticles，你可以简单的deploy更多的实例，通过在命令行中使用使用`-instances`选项。

````
vertx run com.mycompany.MyVerticle -instances 10
````

或者，以编程方式deploy 你的verticle:

````
DeploymentOptions options = new DeploymentOptions().setInstances(10);
vertx.deployVerticle("com.mycompany.MyVerticle", options);
````

一旦你这样做了，你会发现echo server 和之前一样工作，但是server上的所有核都利用了，更多的工作可以被处理。

此时，你可能会问自己 **为什么可以有多个server监听同一个host和port，肯定的是，当你想部署多于一个实例时，会发生port冲突**.

**Vert.x does a little magic here.**

当你在同一个host和port上部署另一个server时，因为已经存在server,所以它实际上并不会创建一个监听相同host和port的新的server。

相反的，它内部只维护一个server，当到来connections时，它以round-robin方式分发connection到所有connect handlers。

结果，Vert.x TCP servers可以随着可用CPU扩容，但是每一个实例仍然是单线程。

#### Creating a TCP client
最简单的创建TCP 客户端的方式，是使用所有的默认配置，如下所示：

````
NetClient client = vertx.createNetClient();
````

#### Configuring a TCP client
如果你不想要默认的配置，可以在创建客户端时传递一个[NetClientOptions](http://vertx.io/docs/apidocs/io/vertx/core/net/NetClientOptions.html)实例。

````
NetClientOptions options = new NetClientOptions().setConnectTimeout(10000);
NetClient client = vertx.createNetClient(options);
````

#### Making connections
为了连接到一个server，你可以用[connect](http://vertx.io/docs/apidocs/io/vertx/core/net/NetClient.html#connect-int-java.lang.String-io.vertx.core.Handler-),指定host和port，还有一个Handler，当连接成功时，这个Handler会被调用并传入一个[NetSocket](http://vertx.io/docs/apidocs/io/vertx/core/net/NetSocket.html)，在连接失败时，发生失败。

#### Configuring connection attempts
一个客户端可以配置为在它不能连接时自动重新连接到server。这就是配置[setReconnectInterval](http://vertx.io/docs/apidocs/io/vertx/core/net/NetClientOptions.html#setReconnectInterval-long-),[setReconnectAttempts](http://vertx.io/docs/apidocs/io/vertx/core/net/NetClientOptions.html#setReconnectAttempts-int-)

> NOTE
>
>Currently Vert.x will not attempt to reconnect if a connection fails, reconnect attempts and interval only apply to creating initial connections.

````
NetClientOptions options = new NetClientOptions().
    setReconnectAttempts(10).
    setReconnectInterval(500);

NetClient client = vertx.createNetClient(options);
````

默认的，不允许尝试多次连接.

#### Logging network activity
为了debug，网络活动可以被记录：

````
NetServerOptions options = new NetServerOptions().setLogActivity(true);

NetServer server = vertx.createNetServer(options);
````

对于客户端来说：

````
NetClientOptions options = new NetClientOptions().setLogActivity(true);

NetClient client = vertx.createNetClient(options);
````

网络活动被Netty记录，以DEBUG level，以`io.netty.handler.logging.LoggingHandler`名。当使用网络活动日志时，有一些事要记住：

+ 记日志是由Netty做而不是Vert.x
+ 这不是一个生产环境特性

Netty 将会定位logger 实现，以下面的顺序：

+ SLf4j
+ Log4j
+ JDK

SLf4j或Log4j classes出现在classpath对于选出logging implementation是足够的。

也可以指定一个具体的实现：

````
// Force logging to SLF4J
InternalLoggerFactory.setDefaultFactory(Log4JLoggerFactory.INSTANCE);
````

#### Configuring servers and clients to work with SSL/TLS
TCP 客户端和服务端可以配置为使用[Transport_Layer_Security](http://en.wikipedia.org/wiki/Transport_Layer_Security)--一个早期的TLS版本，被称为SSL。

服务端和客户端的API是相同的，不管是否使用了SSL/TLS,可以通过配置`NetClientOptions`或`NetServerOptions`实例。

#### Enabling SSL/TLS on the server
通过[ssl](http://vertx.io/docs/apidocs/io/vertx/core/net/NetServerOptions.html#setSsl-boolean-)可以启用SSL/TLS。

默认的，它是不启用的。

#### Specifying key/certificate for the server
SSL/TLS 服务端通常提供了为客户端提供了证书，为了验证客户端的身份。

certificate/keys 可以通过几种方式为服务端配置：

第一种方式是通过指定一个包含证书和private key的Java key-store的位置。

Java key stores 可以通过JDK 提供的[keytool](http://docs.oracle.com/javase/6/docs/technotes/tools/solaris/keytool.html)来管理。

key store的秘法也需要提供：

````
NetServerOptions options = new NetServerOptions().setSsl(true).setKeyStoreOptions(
    new JksOptions().
        setPath("/path/to/your/server-keystore.jks").
        setPassword("password-of-your-keystore")
);
NetServer server = vertx.createNetServer(options);
````

可选的，你可以自己读取key store为buffer,并且直接提供：

````
Buffer myKeyStoreAsABuffer = vertx.fileSystem().readFileBlocking("/path/to/your/server-keystore.jks");
JksOptions jksOptions = new JksOptions().
    setValue(myKeyStoreAsABuffer).
    setPassword("password-of-your-keystore");
NetServerOptions options = new NetServerOptions().
    setSsl(true).
    setKeyStoreOptions(jksOptions);
NetServer server = vertx.createNetServer(options);
````

PKCS#12(https://en.wikipedia.org/wiki/PKCS_12) 格式的Key/certificate，通常以`.pfx`或`.p12`为后缀，也可以以相似的方式加载。

````
NetServerOptions options = new NetServerOptions().setSsl(true).setPfxKeyCertOptions(
    new PfxOptions().
        setPath("/path/to/your/server-keystore.pfx").
        setPassword("password-of-your-keystore")
);
NetServer server = vertx.createNetServer(options);
````

Buffer 配置也是支持的：

````
NetServerOptions options = new NetServerOptions().setSsl(true).setPfxKeyCertOptions(
    new PfxOptions().
        setPath("/path/to/your/server-keystore.pfx").
        setPassword("password-of-your-keystore")
);
NetServer server = vertx.createNetServer(options);
````

另一种提供server private key和证书的方式是分开的`.pem`文件。

````
NetServerOptions options = new NetServerOptions().setSsl(true).setPemKeyCertOptions(
    new PemKeyCertOptions().
        setKeyPath("/path/to/your/server-key.pem").
        setCertPath("/path/to/your/server-cert.pem")
);
NetServer server = vertx.createNetServer(options);
````

Buffer 配置也是支持的：

````
Buffer myKeyAsABuffer = vertx.fileSystem().readFileBlocking("/path/to/your/server-key.pem");
Buffer myCertAsABuffer = vertx.fileSystem().readFileBlocking("/path/to/your/server-cert.pem");
PemKeyCertOptions pemOptions = new PemKeyCertOptions().
    setKeyValue(myKeyAsABuffer).
    setCertValue(myCertAsABuffer);
NetServerOptions options = new NetServerOptions().
    setSsl(true).
    setPemKeyCertOptions(pemOptions);
NetServer server = vertx.createNetServer(options);
````

记住，pem配置，private key是未加密的。

#### Specifying trust for the server
SSL/TLS server可以使用一个证书授权来验证客户端的身份。

证书验证可以通过几种方式在server配置：

Java trust store可以通过JDK提供的[keytool](http://docs.oracle.com/javase/6/docs/technotes/tools/solaris/keytool.html)。

trust store的密码也要提供：

````
NetServerOptions options = new NetServerOptions().
    setSsl(true).
    setClientAuth(ClientAuth.REQUIRED).
    setTrustStoreOptions(
        new JksOptions().
            setPath("/path/to/your/truststore.jks").
            setPassword("password-of-your-truststore")
    );
NetServer server = vertx.createNetServer(options);
````

可选的，可以自己读取trust store为buffer，直接提供：

````
Buffer myTrustStoreAsABuffer = vertx.fileSystem().readFileBlocking("/path/to/your/truststore.jks");
NetServerOptions options = new NetServerOptions().
    setSsl(true).
    setClientAuth(ClientAuth.REQUIRED).
    setTrustStoreOptions(
        new JksOptions().
            setValue(myTrustStoreAsABuffer).
            setPassword("password-of-your-truststore")
    );
NetServer server = vertx.createNetServer(options);
````

PKCS#12格式的证书授权，通常以`.pfx`或`.p12`为扩展名，也可以以相同的方式加载：

````
NetServerOptions options = new NetServerOptions().
    setSsl(true).
    setClientAuth(ClientAuth.REQUIRED).
    setPfxTrustOptions(
        new PfxOptions().
            setPath("/path/to/your/truststore.pfx").
            setPassword("password-of-your-truststore")
    );
NetServer server = vertx.createNetServer(options);
````

buffer 配置也是支持的：

````
Buffer myTrustStoreAsABuffer = vertx.fileSystem().readFileBlocking("/path/to/your/truststore.pfx");
NetServerOptions options = new NetServerOptions().
    setSsl(true).
    setClientAuth(ClientAuth.REQUIRED).
    setPfxTrustOptions(
        new PfxOptions().
            setValue(myTrustStoreAsABuffer).
            setPassword("password-of-your-truststore")
    );
NetServer server = vertx.createNetServer(options);
````

另一种提供server 证书授权的方式是使用`.pem`文件。

````
NetServerOptions options = new NetServerOptions().
    setSsl(true).
    setClientAuth(ClientAuth.REQUIRED).
    setPemTrustOptions(
        new PemTrustOptions().
            addCertPath("/path/to/your/server-ca.pem")
    );
NetServer server = vertx.createNetServer(options);
````

buffer 配置也是支持的：

````
Buffer myCaAsABuffer = vertx.fileSystem().readFileBlocking("/path/to/your/server-ca.pfx");
NetServerOptions options = new NetServerOptions().
    setSsl(true).
    setClientAuth(ClientAuth.REQUIRED).
    setPemTrustOptions(
        new PemTrustOptions().
            addCertValue(myCaAsABuffer)
    );
NetServer server = vertx.createNetServer(options);
````

#### Enabling SSL/TLS on the client
TCP 客户端也可以配置为使用SSL。They have the exact same API when using SSL as when using standard sockets.

为了在一个NetClient上启用SSL，调用`setSsl(true)`

#### Client trust configuration
如果[trustAll](http://vertx.io/docs/apidocs/io/vertx/core/net/ClientOptionsBase.html#setTrustAll-boolean-)被设置为true，那么客户端将会trust all server certificate。connection将仍然是加密的，但是这种模式是脆弱的。例如，你不能保证你连接的是谁。谨慎的使用。默认的值是false。

````
NetClientOptions options = new NetClientOptions().
    setSsl(true).
    setTrustAll(true);
NetClient client = vertx.createNetClient(options);
````

如果`trustAll`没有设置，那么客户端trust store 必须配置，并且应该包含客户端相信的server的证书。

默认的，客户端的host verification 是没有启用的。为了启用host verification，设置客户端使用的算法（目前支持 HTTPS 和 LDAPS）.

````
NetClientOptions options = new NetClientOptions().
        setSsl(true).
        setHostnameVerificationAlgorithm("HTTPS");
NetClient client = vertx.createNetClient(options);
````

和server 配置一样，客户端trust 也可以通过几种方式配置：

第一种方式是通过指定包含certificate 授权的Java trust store的位置。

它是一个标准的Java key store，和服务端的key store一样。客户端trust store 位置通过使用方法[path](http://vertx.io/docs/apidocs/io/vertx/core/net/JksOptions.html#setPath-java.lang.String-)和[jks options](http://vertx.io/docs/apidocs/io/vertx/core/net/JksOptions.html)。如果server 在连接时出现了一个客户端不相信的证书，连接将不会成功。

````
NetClientOptions options = new NetClientOptions().
    setSsl(true).
    setTrustStoreOptions(
        new JksOptions().
            setPath("/path/to/your/truststore.jks").
            setPassword("password-of-your-truststore")
    );
NetClient client = vertx.createNetClient(options);
````

buffer 配置也是支持的：

````
Buffer myTrustStoreAsABuffer = vertx.fileSystem().readFileBlocking("/path/to/your/truststore.jks");
NetClientOptions options = new NetClientOptions().
    setSsl(true).
    setTrustStoreOptions(
        new JksOptions().
            setValue(myTrustStoreAsABuffer).
            setPassword("password-of-your-truststore")
    );
NetClient client = vertx.createNetClient(options);
````

PKCS#12格式的证书，通常以`.pfx`或`.p12`为扩展名，也可以以相同的方式加载：

````
NetClientOptions options = new NetClientOptions().
    setSsl(true).
    setPfxTrustOptions(
        new PfxOptions().
            setPath("/path/to/your/truststore.pfx").
            setPassword("password-of-your-truststore")
    );
NetClient client = vertx.createNetClient(options);
````

Buffer配置也是支持的：

````
Buffer myTrustStoreAsABuffer = vertx.fileSystem().readFileBlocking("/path/to/your/truststore.pfx");
NetClientOptions options = new NetClientOptions().
    setSsl(true).
    setPfxTrustOptions(
        new PfxOptions().
            setValue(myTrustStoreAsABuffer).
            setPassword("password-of-your-truststore")
    );
NetClient client = vertx.createNetClient(options);
````

另一种提供证书验证的方式是使用`.pem`文件：

````
NetClientOptions options = new NetClientOptions().
    setSsl(true).
    setPemTrustOptions(
        new PemTrustOptions().
            addCertPath("/path/to/your/ca-cert.pem")
    );
NetClient client = vertx.createNetClient(options);
````

Buffer配置也是支持的：

````
Buffer myTrustStoreAsABuffer = vertx.fileSystem().readFileBlocking("/path/to/your/ca-cert.pem");
NetClientOptions options = new NetClientOptions().
    setSsl(true).
    setPemTrustOptions(
        new PemTrustOptions().
            addCertValue(myTrustStoreAsABuffer)
    );
NetClient client = vertx.createNetClient(options);
````

#### Specifying key/certificate for the client
If the server requires client authentication then the client must present its own certificate to the server when connecting. The client can be configured in several ways:

The first method is by specifying the location of a Java key-store which contains the key and certificate. Again it’s just a regular Java key store. The client keystore location is set by using the function [path](http://vertx.io/docs/apidocs/io/vertx/core/net/JksOptions.html#setPath-java.lang.String-) on the [jks options](http://vertx.io/docs/apidocs/io/vertx/core/net/JksOptions.html).

````
NetClientOptions options = new NetClientOptions().setSsl(true).setKeyStoreOptions(
    new JksOptions().
        setPath("/path/to/your/client-keystore.jks").
        setPassword("password-of-your-keystore")
);
NetClient client = vertx.createNetClient(options);
````

Buffer configuration is also supported:

````
Buffer myKeyStoreAsABuffer = vertx.fileSystem().readFileBlocking("/path/to/your/client-keystore.jks");
JksOptions jksOptions = new JksOptions().
    setValue(myKeyStoreAsABuffer).
    setPassword("password-of-your-keystore");
NetClientOptions options = new NetClientOptions().
    setSsl(true).
    setKeyStoreOptions(jksOptions);
NetClient client = vertx.createNetClient(options);
````

Key/certificate in PKCS#12 format (http://en.wikipedia.org/wiki/PKCS_12), usually with the `.pfx` or the `.p12` extension can also be loaded in a similar fashion than JKS key stores:

````
NetClientOptions options = new NetClientOptions().setSsl(true).setPfxKeyCertOptions(
    new PfxOptions().
        setPath("/path/to/your/client-keystore.pfx").
        setPassword("password-of-your-keystore")
);
NetClient client = vertx.createNetClient(options);
````

Buffer configuration is also supported:

````
Buffer myKeyStoreAsABuffer = vertx.fileSystem().readFileBlocking("/path/to/your/client-keystore.pfx");
PfxOptions pfxOptions = new PfxOptions().
    setValue(myKeyStoreAsABuffer).
    setPassword("password-of-your-keystore");
NetClientOptions options = new NetClientOptions().
    setSsl(true).
    setPfxKeyCertOptions(pfxOptions);
NetClient client = vertx.createNetClient(options);
````

Another way of providing server private key and certificate separately using `.pem` files.

````
NetClientOptions options = new NetClientOptions().setSsl(true).setPemKeyCertOptions(
    new PemKeyCertOptions().
        setKeyPath("/path/to/your/client-key.pem").
        setCertPath("/path/to/your/client-cert.pem")
);
NetClient client = vertx.createNetClient(options);
````

Buffer configuration is also supported:

````
Buffer myKeyAsABuffer = vertx.fileSystem().readFileBlocking("/path/to/your/client-key.pem");
Buffer myCertAsABuffer = vertx.fileSystem().readFileBlocking("/path/to/your/client-cert.pem");
PemKeyCertOptions pemOptions = new PemKeyCertOptions().
    setKeyValue(myKeyAsABuffer).
    setCertValue(myCertAsABuffer);
NetClientOptions options = new NetClientOptions().
    setSsl(true).
    setPemKeyCertOptions(pemOptions);
NetClient client = vertx.createNetClient(options);
````

记住，在pem 配置中，private key是没有加密的。

#### Revoking certificate authorities
使用一个certificate 吊销 列表废除不再信任的证书。[crlPath](http://vertx.io/docs/apidocs/io/vertx/core/net/NetClientOptions.html#addCrlPath-java.lang.String-)配置使用的crl list:

````
NetClientOptions options = new NetClientOptions().
    setSsl(true).
    setTrustStoreOptions(trustOptions).
    addCrlPath("/path/to/your/crl.pem");
NetClient client = vertx.createNetClient(options);
````

Buffer 配置也是支持的：

````
Buffer myCrlAsABuffer = vertx.fileSystem().readFileBlocking("/path/to/your/crl.pem");
NetClientOptions options = new NetClientOptions().
    setSsl(true).
    setTrustStoreOptions(trustOptions).
    addCrlValue(myCrlAsABuffer);
NetClient client = vertx.createNetClient(options);
````

#### Configuring the Cipher suite
默认的，TLS配置将使用Vert.x运行的JVM的Cipher suite。这个Cipher suite可以配置为 a suite of enabled Ciphers:

````
NetServerOptions options = new NetServerOptions().
    setSsl(true).
    setKeyStoreOptions(keyStoreOptions).
    addEnabledCipherSuite("ECDHE-RSA-AES128-GCM-SHA256").
    addEnabledCipherSuite("ECDHE-ECDSA-AES128-GCM-SHA256").
    addEnabledCipherSuite("ECDHE-RSA-AES256-GCM-SHA384").
    addEnabledCipherSuite("CDHE-ECDSA-AES256-GCM-SHA384");
NetServer server = vertx.createNetServer(options);
````

#### Configuring TLS protocol versions
默认的，TLS配置将使用下面的协议：SSLv2Hello，TLSv1，TLSv1.1 ，TLSv1.2。协议版本可以显式的配置增加允许的协议：

````
NetServerOptions options = new NetServerOptions().
    setSsl(true).
    setKeyStoreOptions(keyStoreOptions).
    addEnabledSecureTransportProtocol("TLSv1.1").
    addEnabledSecureTransportProtocol("TLSv1.2");
    NetServer server = vertx.createNetServer(options);
````

协议版本可以在`NetServerOptions`和`NetClientOptions`配置中指定。

#### SSL engine
engine实现可以配置为使用[OpenSSL](https://www.openssl.org/)而不是JDK实现。OpenSSL别JDK engine提供了更好的性能和CPU利用率。

engine options to use is:

+ `getSslEngineOptions` options 当被设置
+ 否则，`JdkSSLEngineOptions`

````
NetServerOptions options = new NetServerOptions().
    setSsl(true).
    setKeyStoreOptions(keyStoreOptions);

// Use JDK SSL engine explicitly
options = new NetServerOptions().
    setSsl(true).
    setKeyStoreOptions(keyStoreOptions).
    setJdkSslEngineOptions(new JdkSSLEngineOptions());

// Use OpenSSL engine
options = new NetServerOptions().
    setSsl(true).
    setKeyStoreOptions(keyStoreOptions).
    setOpenSslEngineOptions(new OpenSSLEngineOptions());
````

#### Application-Layer Protocol Negotiation
ALPN 是一个应用层协议的TLS扩展。它被用于HTTP/2:在TLS握手中，客户端提供一系列它支持的应用层协议，服务端响应它所支持的。

Java 8 没有提供ALPN的支持，所以ALPN可以通过其他方式启用：

+ OpenSSL 支持
+ Jetty-ALPN 支持

the engine options to use is:

+ [getSslEngineOptions](http://vertx.io/docs/apidocs/io/vertx/core/net/TCPSSLOptions.html#getSslEngineOptions--)
+ [JdkSSLEngineOptions](http://vertx.io/docs/apidocs/io/vertx/core/net/JdkSSLEngineOptions.html)
+ [OpenSSLEngineOptions](http://vertx.io/docs/apidocs/io/vertx/core/net/OpenSSLEngineOptions.html)
+ 否则失败

##### OpenSSL ALPN support
OpenSSL提供了原生的ALPN 支持。

OpenSSL要求配置[setOpenSslEngineOptions](http://vertx.io/docs/apidocs/io/vertx/core/net/TCPSSLOptions.html#setOpenSslEngineOptions-io.vertx.core.net.OpenSSLEngineOptions-),在classpath使用[netty-tcnative](http://netty.io/wiki/forked-tomcat-native.html)jar。使用tcnative可能要求OpenSSL安装在你的操作系统上，取决于tcnative实现。

##### Jetty-ALPN support
Jetty-ALPN 是一个small jar，重载了Java 8发行版中一些类对ALPN的支持。

JVM 启动时，必须有 `alpn-boot-${version}.jar`在它的`bootclasspath`:

````
-Xbootclasspath/p:/path/to/alpn-boot${version}.jar
````

`${version}`取决于JVM 版本，比如OpenJDK 1.8.0u74对应于 8.1.7.v20160121。完整的列表在[Jetty-ALPN page](http://www.eclipse.org/jetty/documentation/current/alpn-chapter.html)。

主要的缺点是，version取决于 JVM.

为了解决这个问题，可以安装[Jetty-ALPN agent](https://github.com/jetty-project/jetty-alpn-agent)。这个客户端是一个JVM 代理，将会为JVM挑选正确的ALPN 版本：

````
-javaagent:/path/to/alpn/agent
````

#### Using a proxy for client connections
[NetClient](http://vertx.io/docs/apidocs/io/vertx/core/net/NetClient.html)也支持 HTTP/1.x 连接，SOCKS4a 或 SOCKS5 proxy。

这个代理可以在[NetClientOptions](http://vertx.io/docs/apidocs/io/vertx/core/net/NetClientOptions.html)配置，通过设置一个[ProxyOptions](http://vertx.io/docs/apidocs/io/vertx/core/net/ProxyOptions.html)对象，包含代理类型，hostname,port and optionally username and password.

下面是例子：

````
NetClientOptions options = new NetClientOptions()
    .setProxyOptions(new ProxyOptions().setType(ProxyType.SOCKS5)
        .setHost("localhost").setPort(1080)
        .setUsername("username").setPassword("secret"));
NetClient client = vertx.createNetClient(options);
````

DNS 解决方案总是在代理服务器上完成，为了达到一个SOCKS4 客户端的功能，必须在本地解决DNS 地址。

### Writing HTTP servers and clients
Vert.x 允许你很容易的写非阻塞的HTTP 客户端和服务端。

Vert.x 支持 HTTP/1.0,HTTP/1.1,HTTP/2 协议。

HTTP 的基本API对于HTTP/1.x 和 HTTP/2是相同的，特殊的API 特性用于处理HTTP/2 协议。

#### Creating an HTTP Server
最简单的创建HTTP Server的方式是，使用所有的默认选项，如下所示：

````
HttpServer server = vertx.createHttpServer();
````

#### Configuring an HTTP Server
如果你不想默认的配置，可以在创建server 时传递一个[HttpServerOptions](http://vertx.io/docs/apidocs/io/vertx/core/http/HttpServerOptions.html)实例。

````
HttpServerOptions options = new HttpServerOptions().setMaxWebsocketFrameSize(1000000);

HttpServer server = vertx.createHttpServer(options);
````

#### Configuring an HTTP/2 Server
Vert.x支持HTTP/2 通过 TLS `h2`和 TCP `h2c`。

+ `h2` 定义了HTTP/2 协议，when used over TLS Negotiated by Application-Layer Protocol Negotiation.
+ `h2c` 定义了HTTP/2 协议，when using in clear text over TCP,such connections are established either with an HTTP/1.1 upgraded request or directly

为了处理`h2`请求，TLS 必须启用通过`setUseAlpn`:

````
HttpServerOptions options = new HttpServerOptions()
    .setUseAlpn(true)
    .setSsl(true)
    .setKeyStoreOptions(new JksOptions().setPath("/path/to/my/keystore"));

HttpServer server = vertx.createHttpServer(options);
````

ALPN 是TLS扩展，在客户端和服务端开始交换数据前协商协议。

不支持ALPN的客户端仍然可以使用经典的SSL 握手。

ALPN will usually agree on the `h2` protocol,although `http/1.1` can be used if the server or the client decides so.

为了处理`h2c`请求，TLS 必须禁用。server will upgrade to HTTP/2 any request HTTP/1.1 that wants to upgrade to HTTP/2。 it will also accept a direct `h2c` connection beginning with the `PRI * HTTP/2.0 \r\n SM \r\n`preface。

> WARNING
>
>大多数浏览器不支持`h2c`，所以为了服务web site你应该使用`h2`而不是`h2c`。

当一个服务端接受HTTP/2 connection,它将它的`initial Settings`发送给客户端。这些配置定义了客户端如何使用连接，服务端的默认配置是：

+ [getMaxConcurrentStreams](http://vertx.io/docs/apidocs/io/vertx/core/http/Http2Settings.html#getMaxConcurrentStreams--):100，HTTP/2 RFC 推荐。

+ the Default HTTP/2 Settings values for the others

#### Logging network server activity
为了debug目的，网络活动可以被记录。

````
HttpServerOptions options = new HttpServerOptions().setLogActivity(true);

HttpServer server = vertx.createHttpServer(options);
````

查看[logging_network_activity](http://vertx.io/docs/vertx-core/java/#logging_network_activity)章节，获取详细的解释。

#### Start the server Listening
为了告诉Server监听到来的请求，你使用`listen`方法中的一个。

为了告诉server监听options中指定的host 和 port：

````
HttpServer server = vertx.createHttpServer();
server.listen();
````

或者，在调用listen时指定host和port，忽略options中的配置：

````
HttpServer server = vertx.createHttpServer();
server.listen(8080, "myhost.com");
````

默认的host是`0.0.0.0`，意味着在所有可用的地址上监听，默认的port是80.

真正的绑定是异步的，所以Server可能没有真正开始监听直到调用listen方法返回后的一段时间。

如果你想在Server真正监听时得到通知，你可以在调用`listen`时提供一个handler。例如：

````
HttpServer server = vertx.createHttpServer();
server.listen(8080, "myhost.com", res -> {
  if (res.succeeded()) {
    System.out.println("Server is now listening!");
  } else {
    System.out.println("Failed to bind!");
  }
});
````

#### Getting notified of incoming requests
为了在request达到时得到通知，你需要设置一个`requestHandler`：

````
HttpServer server = vertx.createHttpServer();
server.requestHandler(request -> {
  // Handle the request in here
});
````

#### Handling requests
当一个request到达，request Handler 被调用，传递给一个[HttpServerRequest](http://vertx.io/docs/apidocs/io/vertx/core/http/HttpServerRequest.html).这个对象代表服务端的HTTP request.

当request的headers完全读出时，Handler被调用。

如果request 包含一个body，这个body将会在request Handler 被调用后的一段时间到达server。

Server request 对象允许你取出`uri`,`path`,`params`,`headers`,几乎所有的事。

每一个server request object和一个server response object关联。你可以用[response](http://vertx.io/docs/apidocs/io/vertx/core/http/HttpServerRequest.html#response--)获得一个`HttpServerResponse`对象的引用。

下面是一个server 处理一个请求并且返回"hello world"。

````
vertx.createHttpServer().requestHandler(request -> {
  request.response().end("Hello world");
}).listen(8080);
````

##### Request version
request 指定的HTTP version 可以通过[version](http://vertx.io/docs/apidocs/io/vertx/core/http/HttpServerRequest.html#version--)获取。

##### Request method
使用[method](http://vertx.io/docs/apidocs/io/vertx/core/http/HttpServerRequest.html#method--)获取一个请求的HTTP method(GET, POST, PUT, DELETE, HEAD, OPTIONS).

##### Request URI
使用[uri](http://vertx.io/docs/apidocs/io/vertx/core/http/HttpServerRequest.html#uri--)获取request 的 URI.

记住，这是HTTP request里的真实URI，大多数情况下它是一个相对URI。

URI 在[Section 5.1.2 of the HTTP specification - Request-URI](http://www.w3.org/Protocols/rfc2616/rfc2616-sec5.html)定义。

##### Request path
使用[path](http://vertx.io/docs/apidocs/io/vertx/core/http/HttpServerRequest.html#path--)返回URI的path 部分。

例如，如果request uri 是：`a/b/c/page.html?param1=abc&param2=xyz`

那么path 部分是：`/a/b/c/page.html`

##### Request query
使用[query](http://vertx.io/docs/apidocs/io/vertx/core/http/HttpServerRequest.html#query--)返回URI的query部分。

例如，如果request uri是：`a/b/c/page.html?param1=abc&param2=xyz`

那么，query部分是：`param1=abc&param2=xyz`

##### Request headers
使用[headers](http://vertx.io/docs/apidocs/io/vertx/core/http/HttpServerRequest.html#headers--)返回HTTP request的headers.

这将返回[MultiMap](http://vertx.io/docs/apidocs/io/vertx/core/MultiMap.html)的一个实例--它就像一个普通的Map或Hash，但是允许同一个key有多个values。--这是因为HTTP 允许同一个key有多个header values。

它也有不区分大小写的key,意味着你可以做以下的事：

````
MultiMap headers = request.headers();

// Get the User-Agent:
System.out.println("User agent is " + headers.get("user-agent"));

// You can also do this and get the same result:
System.out.println("User agent is " + headers.get("User-Agent"));
````

##### Request host
使用[host](http://vertx.io/docs/apidocs/io/vertx/core/http/HttpServerRequest.html#host--)返回HTTP request的host。

对于HTTP/1.x request，`host` header被返回。对于HTTP/1 请求，`:authority` pseudo header被返回。

##### Request parameters
使用[params](http://vertx.io/docs/apidocs/io/vertx/core/http/HttpServerRequest.html#params--)，返回HTTP request的parameters。

就像`headers`，它也返回一个`MultiMap`的实例。

request parameters
