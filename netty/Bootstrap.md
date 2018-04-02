1. 初始化EventLoopGroup，和ServerBootstrap一样，不细说了。
EventLoopGroup实例为：MultithreadEventExecutorGroup
Executor为ThreadPerTaskExecutor
EventExecutor实例为NioEventLoop

如果没有指定nThreads，那么默认为:Runtime.getRuntime().availableProcessors() * 2

2. 设置`Bootstrap`的group.

3. Bootstrap.channel(NioSocketChannel.class)
设置ChannelFactory为`BootstrapChannelFactory`.channelClass为`NioSocketChannel.class`

4. connect()  
### 1. initAndRegister();

#### 1.1 channelFactory().newChannel();
初始化`Unsafe`和`ChannelPipeline`。`Unsafe`为`NioByteUnsafe`,`ChannelPipeline`为`DefaultChannelPipeline`.这里`DefaultChannelPipeline`的初始化过程和`ServerBootstrap`一样。

#### 1.2 init(channel);

````
ChannelPipeline p = channel.pipeline();
p.addLast(handler());
````

#### 1.3 ChannelFuture regFuture = group().register(channel);
和`ServerBootstrap`一样。

### 2. doConnect0()
从`tail`开始，调用`((ChannelOutboundHandler) handler()).connect(this, remoteAddress, localAddress, promise);`
