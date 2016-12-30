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
