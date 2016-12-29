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
