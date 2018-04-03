## 1. 连接篇
### 1.1 Netty Native
Netty Native用C语言编写JNI调用的Socket Transport,比JDK的NIO更省CPU，也许有人问，JDK的NIO也是用epoll啊，大家有什么不同

+ Netty的epoll 是使用边沿触发，而Jdk nio 是使用水平触发。

+ C代码，更少GC，更少同步

+ 暴露了更多的Socket 配置参数

Netty Native还包含了Google 的boringssl（从OpenSSL fork的），JDK原生SSL实现比OpenSSL慢很多很多，而大家把SSL provider配置成OpenSSL时，又要担心操作系统有没有装OpenSSL，现在好了。

### 1.2 异步传输，异步连接
connect方法和网络读写方法都是非阻塞的。

### Channel 参数设定
TCP/Socket的设置，SO_REUSEADDR， TCP_NODELAY， SO_KEEPALIVE 。另外还有SO_LINGER ， SO_TIMEOUT, SO_BACKLOG, SO_SNDBUF, SO_RCVBUF。

而用了Native后又加了TCP_CORK和KeepAlive包发送的时间间隔(默认2小时)，详见EpoolSocketChannelConfig的JavaDoc。

而Netty自己的参数,CONNECT_TIMEOUT_MILLIS，是Netty自己起一个定时任务来监控建立连接是否超时，默认30秒太长谁也受不了，一般会弄短它。

## 2. 线程篇
### workGroup 和bossGroup
bossGroup用于服务端处理客户端的建立连接的请求，workGroup用于处理I/O

EventLoopGroup的默认大小是`2 * CPU核数`。

### 业务线程池
Netty线程的数量一般固定且较少，所以很怕线程被堵塞，比如同步的数据库查询，比如下游的服务调用（又来罗嗦，future.get()式的异步在执行future.get()时还是堵住当前线程的啊）。

所以，此时就要把处理放到一个业务线程池里操作，即使要付出线程上下文切换的代价，甚至还有些ThreadLocal需要复制。

### 定时任务
像发送超时控制之类的一次性任务，不要使用JDK的ScheduledExecutorService，而是如下`ctx.executor().schedule(new MyTimeoutTask(p), 30, TimeUnit.SECONDS)`。首先，JDK的ScheduledExecutorService是一个大池子，多线程争抢并发锁。而上面的写法，TimeoutTask只属于当前的EventLoop，没有任何锁。

其次，如果发送成功，需要从长长Queue里找回任务来取消掉它。现在每个EventLoop一条Queue，明显长度只有原来的N分之一。

### Netty高性能线程池
Netty的线程池理念有点像ForkJoinPool，都不是一个大池子并发等待一条任务队列，而是每条线程有一个自己的任务队列。

不过Netty4的方法是建了N个只有一条线程的线程池，然后用前面说的选择器去选择。

Netty的线程，并不是简单地阻塞的拉取任务，而是非常辛苦的在每一个循环同时做三件事情：

+ 先处理NIO事件。

+ 再将线程的`scheduledTaskQueue`里的定时任务取出来放到`taskQueue`中

+ 再执行任务队列（taskQueue）里的任务

每个循环里处理I/O事件的其他任务的时间比例，可以通过ioRatio来控制。默认是各占50%。

taskQueue为有界的LinkedBlockingQueue

## 内存篇
### 堆外内存池
堆外内存是Netty被说的最多的部分。使用堆外内存不受heap大小限制。

+ Netty的接收和发送ByteBuffer使用direct buffer,使用堆外内存进行Socket读写，不需要进行字节缓冲区的二次拷贝。

零拷贝：

用CompositeByteBuffer将多个bytebuffer合成一个。scatter/gather

channel to channel transfer.

### 避免扩容：ByteBuf的大小预估与AdaptiveRecvByteBufAllocator
根据本次读取的实际字节数对下次接收的buffer容量进行预测。


JDK的ThreadLocal，实现原理是Thread对象里有个HashMap性质的数组，每个ThreadLocal的id是个Hashcode，算法是currentValue+0x61c88647，hashCode取模数组大小得到threadLocal存的位置，如果桶里已有其他元素，key.nextHashCode()找下一个桶.

而FastThreadLocal的id则是一个自增的int，FastThreadLocalThread里放一个数组，直接按下标获取，没有hash，没有比较，没有冲突。不过需要在Netty地界里用，业务线程池就要自己定义ThreadFactory，创建FastThreadLocalThread 而不是Thread。

1. ctx.writeAndFlush() 与 channel.writeAndFlush()的区别在于，channel要经过整条Pipeline，而ctx直接找下一个outboundHandler。

2. channel.writeAndFlush(buf, channel.voidPromise() )
writeAndFlush不管你用不用默认构造返回一个Promise(Future)，有点浪费内存。没有用的话，用一个公共的 voidPromise ，减少大家花费。但低版本的Netty不能用。

3. 空闲连接管理，因为刚才说的ctx.writeAndFlush()可能不经过IdleHander，所以只监控读空闲就够了。而且如果每次请求都要READ/WRITE/ALL IDEL三个值算一遍，也白白消耗性能。

4. writeAndFlush()不要太多，毕竟调用了系统调用。

5. Handler能共用就标上Shareable Annotation然后共用，不要每个Channel建一个。


### Recycler
Netty的另一个得意的设计值对象可以在线程内无锁的被回收重用。
