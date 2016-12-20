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

(1) final ChannelFuture regFuture = initAndRegister();
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
