1. 初始化EventLoopGroup
EventLoopGroup workerGroup = new NioEventLoopGroup();

EventLoopGroup实例为：MultithreadEventExecutorGroup
Executor为ThreadPerTaskExecutor
EventExecutor实例为NioEventLoop

如果没有指定nThreads，那么默认为:Runtime.getRuntime().availableProcessors() * 2

2. 设置ServerBootstrap的group和childGroup
b.group(bossGroup, workerGroup)

3. 设置ChannelFactory为ReflectiveChannelFactory，Channel class为NioServerSocketChannel.class
b.channel(NioServerSocketChannel.class)

4. 设置SocketOption
b.option(ChannelOption.SO_BACKLOG, 100)

5. 设置Handler，这个Handler会添加到ServerSocketChannel的ChannelHandler上。
b.handler(new LoggingHandler(LogLevel.INFO))

6. 设置ChildHandler，这个Handler会添加到SocketChannel的ChannelHandler上。

7. 绑定地址，启动Server

```
ChannelFuture f = b.bind(PORT)
```

绑定地址这里会执行很多操作，比如初始化`Unsafe`,`ChannelPipeline`等。我们仔细看看。

### 1.1 注册
#### 1.1.1 final ChannelFuture regFuture = initAndRegister();
先调用`channel = channelFactory.newChannel();`。从上面的分析我们知道，ChannelFactory的类型为`ReflectiveChannelFactory`，他的`newChannel`方法如下：`clazz.newInstance();`。这会调用`NioServerSocketChannel`的无参构造方法。

NioServerSocketChannel的无参构造方法如下：

````
public NioServerSocketChannel() {
    this(newSocket(DEFAULT_SELECTOR_PROVIDER));
}
````

接下来调用：

````
public NioServerSocketChannel(ServerSocketChannel channel) {
    super(null, channel, SelectionKey.OP_ACCEPT);
    config = new NioServerSocketChannelConfig(this, javaChannel().socket());
}
````

再调用

````
protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    super(parent);
    this.ch = ch;
    this.readInterestOp = readInterestOp;
    try {
        ch.configureBlocking(false);
    } catch (IOException e) {
        try {
            ch.close();
        } catch (IOException e2) {
            if (logger.isWarnEnabled()) {
                logger.warn(
                        "Failed to close a partially initialized socket.", e2);
            }
        }

        throw new ChannelException("Failed to enter non-blocking mode.", e);
    }
}
````

最后调用

````
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    unsafe = newUnsafe();
    pipeline = newChannelPipeline();
}
````

初始化`Unsafe`和`ChannelPipeline`。`Unsafe`为`NioMessageUnsafe`,`ChannelPipeline`为`DefaultChannelPipeline`。看看ChannelPipeline的初始化过程：

````
protected DefaultChannelPipeline(Channel channel) {
    this.channel = ObjectUtil.checkNotNull(channel, "channel");

    tail = new TailContext(this);
    head = new HeadContext(this);

    head.next = tail;
    tail.prev = head;
}
````

AbstractChannelHandlerContext为双向链表.初始化之后，ChannelPipeline为：
-------------------------------
|   HeadContext -> TailContext |
-------------------------------

#### 1.1.2 init(channel);
a. 设置Option和ChildOption

b.

````
ChannelPipeline p = channel.pipeline();

p.addLast(new ChannelInitializer<Channel>() {
    @Override
    public void initChannel(Channel ch) throws Exception {
        final ChannelPipeline pipeline = ch.pipeline();
        ChannelHandler handler = handler();
        if (handler != null) {
            pipeline.addLast(handler);
        }

        // We add this handler via the EventLoop as the user may have used a ChannelInitializer as handler.
        // In this case the initChannel(...) method will only be called after this method returns. Because
        // of this we need to ensure we add our handler in a delayed fashion so all the users handler are
        // placed in front of the ServerBootstrapAcceptor.
        ch.eventLoop().execute(new Runnable() {
            @Override
            public void run() {
                pipeline.addLast(new ServerBootstrapAcceptor(
                        currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
            }
        });
    }
});
}
````

这里的Channel为ServerSocketChannel,ChannelPipeline为ServerSocketChannel的ChannelPipeline。

+ `pipeline.addLast(handler);`将开始设置的Handler添加到ServerSocketChannel的ChannelPipeline.
最终调用`DefaultChannelPipeline`的addLast:

````
public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
    final AbstractChannelHandlerContext newCtx;
    synchronized (this) {
        checkMultiplicity(handler);

        newCtx = newContext(group, filterName(name, handler), handler);

        addLast0(newCtx);

        // If the registered is false it means that the channel was not registered on an eventloop yet.
        // In this case we add the context to the pipeline and add a task that will call
        // ChannelHandler.handlerAdded(...) once the channel is registered.
        if (!registered) {
            newCtx.setAddPending();
            callHandlerCallbackLater(newCtx, true);
            return this;
        }

        EventExecutor executor = newCtx.executor();
        if (!executor.inEventLoop()) {
            newCtx.setAddPending();
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    callHandlerAdded0(newCtx);
                }
            });
            return this;
        }
    }
    callHandlerAdded0(newCtx);
    return this;
}
````

最终调用

````    
private void addLast0(AbstractChannelHandlerContext newCtx) {
        AbstractChannelHandlerContext prev = tail.prev;
        newCtx.prev = prev;
        newCtx.next = tail;
        prev.next = newCtx;
        tail.prev = newCtx;
}
````

如上可以看到，调用`addLast`之后，ChannelPipeline变为：

--------------------------------------------------
|   HeadContext -> ChannelHandler -> TailContext |
--------------------------------------------------

+ 给ServerSocketChannel的eventLoop提交一个任务：`pipeline.addLast(new ServerBootstrapAcceptor(
        currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));`。`ServerBootstrapAcceptor`是一个`ChannelInboundHandlerAdapter`.这个的作用是什么呢？

我们看看,ServerBootstrapAcceptor重载了`channelRead`：

````
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    final Channel child = (Channel) msg;

    child.pipeline().addLast(childHandler);

    for (Entry<ChannelOption<?>, Object> e: childOptions) {
        try {
            if (!child.config().setOption((ChannelOption<Object>) e.getKey(), e.getValue())) {
                logger.warn("Unknown channel option: " + e);
            }
        } catch (Throwable t) {
            logger.warn("Failed to set a channel option: " + child, t);
        }
    }

    for (Entry<AttributeKey<?>, Object> e: childAttrs) {
        child.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
    }

    try {
        childGroup.register(child).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) {
                    forceClose(child, future.cause());
                }
            }
        });
    } catch (Throwable t) {
        forceClose(child, t);
    }
}
````

我们可以看到,`child.pipeline().addLast(childHandler);....`等等代码，意图已经很明显了。原来这个ChannelHandler的作用是将`childHandler`添加到`SocketChannel`上面，并且设置`ChannelOption`和`childAttrs`。

init(channel);方法到这里就结束了。

#### 1.1.3 `ChannelFuture regFuture = group().register(channel);`,这里的channel为ServerSocketChannel.
通过上面的分析，我们知道，`EventLoopGroup`实例为：`MultithreadEventExecutorGroup`.所以：

````
public ChannelFuture register(Channel channel) {
    return next().register(channel);
}
````

在调用`SingleThreadEventLoop`的方法：

````
public ChannelFuture register(Channel channel) {
    return register(channel, new DefaultChannelPromise(channel, this));
}
````

再调用

````
public ChannelFuture register(final Channel channel, final ChannelPromise promise) {
    if (channel == null) {
        throw new NullPointerException("channel");
    }
    if (promise == null) {
        throw new NullPointerException("promise");
    }

    channel.unsafe().register(this, promise);
    return promise;
}
````

再调用`AbstractChannel`的register方法，关键步骤如下：

````
//1. 获取eventLoop
AbstractChannel.this.eventLoop = eventLoop;

//2. 判断当前线程是否是eventLoop中的线程
if (eventLoop.inEventLoop()) {
  //3. 如果是，则执行注册
    register0(promise);
} else {
  //4. 如果不是，则启动EventLoop中的线程，提交一个任务到eventLoop的taskQueue。
    try {
        eventLoop.execute(new Runnable() {
            @Override
            public void run() {
                register0(promise);
            }
        });
    } catch (Throwable t) {
        logger.warn(
                "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                AbstractChannel.this, t);
        closeForcibly();
        closeFuture.setClosed();
        safeSetFailure(promise, t);
    }
}
}
````

我们接下来看看`register0`方法：

1. 调用`doRegister()`方法。

````
protected void doRegister() throws Exception {
    boolean selected = false;
    for (;;) {
        try {
            selectionKey = javaChannel().register(eventLoop().selector, 0, this);
            return;
        } catch (CancelledKeyException e) {
            if (!selected) {
                // Force the Selector to select now as the "canceled" SelectionKey may still be
                // cached and not removed because no Select.select(..) operation was called yet.
                eventLoop().selectNow();
                selected = true;
            } else {
                // We forced a select operation on the selector before but the SelectionKey is still cached
                // for whatever reason. JDK bug ?
                throw e;
            }
        }
    }
}
````

我们注意到`selectionKey = javaChannel().register(eventLoop().selector, 0, this);`这里传的`int ops`为0.这有什么问题呢？这表示你注册一个Channel，但是没有对任何的操作感兴趣。所以当你调用`select()`方法时，没有任何的操作会被selected,这可能导致`select()`方法永远阻塞，取决于你调用的`select()`方法。

2. pipeline.fireChannelRegistered();

从`head`开始，调用

````
private void invokeChannelRegistered() {
    if (invokeHandler()) {
        try {
            ((ChannelInboundHandler) handler()).channelRegistered(this);
        } catch (Throwable t) {
            notifyHandlerException(t);
        }
    } else {
        fireChannelRegistered();
    }
}
````

3. pipeline.fireChannelActive();
从`head`开始，调用

````
private void invokeChannelActive() {
    if (invokeHandler()) {
        try {
            ((ChannelInboundHandler) handler()).channelActive(this);
        } catch (Throwable t) {
            notifyHandlerException(t);
        }
    } else {
        fireChannelActive();
    }
}
````

register0方法结束。

### 1.2 注册成功完成之后，绑定
从`tail`开始，最终调用：

````
private void invokeBind(SocketAddress localAddress, ChannelPromise promise) {
    if (invokeHandler()) {
        try {
            ((ChannelOutboundHandler) handler()).bind(this, localAddress, promise);
        } catch (Throwable t) {
            notifyOutboundHandlerException(t, promise);
        }
    } else {
        bind(localAddress, promise);
    }
}
````

到这里`bind`方法结束。
