## 通信协议
协议格式：

````
 <length> <header length> <header data> <body data>
    1            2               3           4
````            
 协议分4部分，含义分别如下

+ 1：4个字节整数，等于2、3、4长度总和
+ 2：4个字节整数，等于3的长度
+ 3：使用json序列化数据
+ 4：自定义二进制序列化数据

采用Netty的`LengthFieldBasedFrameDecoder`解码，每个Frame的最大长度为16777216（24 bit），构造器为`LengthFieldBasedFrameDecoder(16777216,0, 4, 0, 4)`。

````
             BEFORE DECODE                                                       AFTER DECODE
 +----------+---------------+----------------+------------+           +-----------------+---------------+-----------+
 |  Length  | Header Length | Header data    | body data  | ----->    |  Header Length  | Header data   | body data |
 |          |  0xCAFE       | "HELLO, WORLD" |            |           |     0xCAFE      | "HELLO, WORLD"|           |
 +----------+---------------+----------------+------------+           +-----------------+---------------+-----------+
````

编码之后length被舍弃掉了，你可能会想如何获取length呢？我们可以通过`ByteBuf.readableBytes()`获得。

## 消息堆积

消息中间件的主要功能是异步解耦,还有个重要功能是挡住前端的数据洪峰,保证后端系统的稳定性,这就要 求消息中间件具有一定的消息堆积能力,消息堆积分以下两种情况:

1. 消息堆积在内存 Buffer,一旦超过内存 Buffer,可以根据一定的丢弃策略来丢弃消息,如 CORBA Notification 规范中描述。适合能容忍丢弃消息的业务,这种情况消息的堆积能力主要在于内存 Buffer 大小,而且消息 堆积后,性能下降不会太大,因为内存中数据多少对于对外提供的访问能力影响有限。

2. 消息堆积到持久化存储系统中,例如DB,KV存储,文件记录形式。

当消息不能在内存 Cache 命中时,要不可避免的访问磁盘,会产生大量读 IO,读 IO 的吞吐量直接决定了 消息堆积后的访问能力。
评估消息堆积能力主要有以下四点:

1. 消息能堆积多少条,多少字节?即消息的堆积容量。

2. 消息堆积后,发消息的吞吐量大小,是否会受堆积影响?

3. 消息堆积后,正常消费的Consumer是否会受影响?

4. 消息堆积后,访问堆积在磁盘的消息时,吞吐量有多大?

|                | 堆积性能指标     |
| :------------- | :------------- |
| 消息的堆积容量      | 依赖磁盘大小       |
| 发消息的吞吐量大小受影响程度|无Slave情况，会受一定影响；有Slave情况，不受影响|
| 正常消费的Consumer是否会受影响|无Slave情况，会受一定影响；有Slave情况，不受影响|
| 访问堆积在磁盘的消息时,吞吐量有多大|与访问的并发有关,最慢会降到 5000 左右|

在有 Slave 情况下,Master 一旦发现 Consumer 访问堆积在磁盘的数据时,会向 Consumer 下达一个重定向指令(见“消息的发布”最后的几行),令 Consumer 从 Slave 拉取数据,这样正常的发消息与正常消费的 Consumer 都不会因为消息堆积受影响,因为系统将堆积场景与非堆积场景分割在了两个不同的节点处理。这里会产生另一个问题,Slave 会不会写性能下降, 答案是否定的。因为 Slave 的消息写入只追求吞吐量,不追求实时性,只要整体的吞吐量高就可以,而 Slave 每次 都是从 Master 拉取一批数据,如 1M,这种批量顺序写入方式即使堆积情况,整体吞吐量影响相对较小,只是写入 RT 会变长。

如何判断消费发生了堆积：

````
consumer.registerMessageListener(new MessageListenerConcurrently() {
            public ConsumeConcurrentlyStatus consumeMessage(
                    List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                ///接收消息个数msgs.size()
                System.out.println(Thread.currentThread().getName()
                        + " Receive New Messages: " + msgs.size());
                MessageExt msg = msgs.get(0);
                long offset = msg.getQueueOffset();
                long maxOffset = Long.parseLong(msg.getProperty(MessageConst.PROPERTY_MAX_OFFSET));
                if (maxOffset - offset > 10000){
                    //消息堆积情况的特殊处理
                }
````

## 长连接与连接复用
在`NettyRemotingClient`类中维护了一个`private final ConcurrentHashMap<String /* addr */, ChannelWrapper> channelTables = new ConcurrentHashMap<String, ChannelWrapper>();` channelTables，这里的地址是“ip:port”,当channelTables中不存在这个key时，则创建新的Channel:

````
ChannelFuture channelFuture = this.bootstrap.connect(RemotingHelper.string2SocketAddress(addr));
cw = new ChannelWrapper(channelFuture);
this.channelTables.put(addr, cw);

channelFuture.awaitUninterruptibly(this.nettyClientConfig.getConnectTimeoutMillis())//3000
````

从而与addr创建长连接.

另一方面，同一个网络连接,客户端多个线程可以同时发送请求,应答响应通过header 中的 opaque 字段来标识,从而达到连接复用。

为什么用ChannelWrapper,而不直接用ChannelFuture呢，封装。
