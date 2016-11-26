## 1.下载，安装
1. 从github上clone最新的rocketmq源码，[项目地址](https://github.com/alibaba/RocketMQ)
2. 进入源码根目录，运行： mvn clean package -Dmaven.test.skip=true install assembly:assembly -U
3. 编译成功后，在target目录下会有 alibaba-rocketmq-3.2.6-alibaba-rocketmq.tar.gz,该压缩包就是安装包。
4. 安装
   解压：tar -zxvf alibaba-rocketmq-3.2.6-alibaba-rocketmq.tar.gz

## 部署NameServer
1. 安装jdk并设置JAVA_HOME环境变量。
2. cd /alibaba-rocketmq/bin 进入RocketMQ的bin目录。
3. 调用 screen sh mqnamesrv &,启动NameServer.
   如果看到:The Name Server boot success.则表示启动成功

   按Ctrl + A,然后按D退出session。

   可以使用nohup sh mqnamesrv -p 打印default配置，打印出的default配置如下：

````
rocketmqHome=/Users/dengyunhui/Documents/dev/alibaba-rocketmq
kvConfigPath=/Users/dengyunhui/namesrv/kvConfig.json
listenPort=9876
serverWorkerThreads=8
serverCallbackExecutorThreads=0
serverSelectorThreads=3
serverOnewaySemaphoreValue=256
serverAsyncSemaphoreValue=64
serverChannelMaxIdleTimeSeconds=120
serverSocketSndBufSize=2048
serverSocketRcvBufSize=1024
serverPooledByteBufAllocatorEnable=false
````

## 停止namesrv
进入bin目录,输入如下命令：`sh mqshutdown namesrv`

## namesrv的日志路径
namesrv的logback配置路径：$ROCKETMQ_HOME/conf/logback_namesrv.xml

namesrv的日志路径：~/logs/rocketmqlogs/namesrv.log

## 启动Broker
进入bin目录，输入如下命令：

`screen sh mqbroker -n localhost:9876`。

如果看到`The broker[dengyunhuideMacBook-Pro.local, 192.168.1.101:10911] boot success. and name server is localhost:9876`，则Broker启动成功。

或者可以`tail -f ~/logs/rocketmqlogs/broker.log`,看日志

## 停止Broker
进入bin目录，输入如下命令：

`sh mqshutdown broker`

## 发送和接收消息
在发送和接收消息之前，我们需要指定namesrv 的地址。我们可以在代码中指定:

`producer.setNamesrvAddr("localhost:9876;192.168.0.2:9876");`

或

`consumer.setNamesrvAddr("localhost:9876;192.168.0.2:9876");`

也可以通过环境变量指定：

`export NAMESRV_ADDR=localhost:9876`

生产者代码：

````
public class Producer {

    public static void main(String[] args) throws MQClientException,
            InterruptedException {
        /**
         * 一个应用创建一个Producer，由应用来维护此对象，可以设置为全局对象或者单例<br>
         * 注意：ProducerGroupName需要由应用来保证唯一<br>
         * ProducerGroup这个概念发送普通的消息时，作用不大，但是发送分布式事务消息时，比较关键，
         * 因为服务器会回查这个Group下的任意一个Producer
         */
        final DefaultMQProducer producer = new DefaultMQProducer(
                "ProducerGroupName");
        //name server服务,多个以;分开
        producer.setNamesrvAddr("localhost:9876");
        producer.setInstanceName("Producer");

        /**
         * Producer对象在使用之前必须要调用start初始化，初始化一次即可<br>
         * 注意：切记不可以在每次发送消息时，都调用start方法
         */
        producer.start();

        /**
         * 下面这段代码表明一个Producer对象可以发送多个topic，多个tag的消息。
         * 注意：send方法是同步调用，只要不抛异常就标识成功。但是发送成功也可会有多种状态，<br>
         * 例如消息写入Master成功，但是Slave不成功，这种情况消息属于成功，但是对于个别应用如果对消息可靠性要求极高，<br>
         * 需要对这种情况做处理。另外，消息可能会存在发送失败的情况，失败重试由应用来处理。
         */
        for (int i = 0; i < 10; i++) {
            try {
                {      //通过topic订阅消息，tag过滤消息
                    Message msg = new Message("TopicTest1",// topic
                            "TagA",// tag 消息标签，只支持设置一个Tag（服务端消息过滤使用）
                            "OrderID001",// key 消息关键词，多个Key用KEY_SEPARATOR隔开（查询消息使用）
                            ("Hello MetaQA").getBytes());// body
                    SendResult sendResult = producer.send(msg);
                    System.out.println(sendResult);
                }

                {
                    Message msg = new Message("TopicTest2",// topic
                            "TagB",// tag
                            "OrderID0034",// key
                            ("Hello MetaQB").getBytes());// body
                    SendResult sendResult = producer.send(msg);
                    System.out.println(sendResult);
                }

                {
                    Message msg = new Message("TopicTest3",// topic
                            "TagC",// tag
                            "OrderID061",// key
                            ("Hello MetaQC").getBytes());// body
                    SendResult sendResult = producer.send(msg);
                    System.out.println(sendResult);
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
            TimeUnit.MILLISECONDS.sleep(1000);
        }

        /**
         * 应用退出时，要调用shutdown来清理资源，关闭网络连接，从MetaQ服务器上注销自己
         * 注意：我们建议应用在JBOSS、Tomcat等容器的退出钩子里调用shutdown方法
         */
        // producer.shutdown();
        Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {
            public void run() {
                producer.shutdown();
            }
        }));
        System.exit(0);
    }
}
````

消费者代码：

````
public class Consumer {
    /**
     * 当前例子是Consumer用法，使用方式给用户感觉是消息从RocketMQ服务器推到了应用客户端。<br>
     * 但是实际Consumer内部是使用长轮询Pull方式从MetaQ服务器拉消息，然后再回调用户Listener方法<br>
     *
     * @throws MQClientException
     */
    public static void main(String[] args) throws MQClientException {
        /**
         * 一个应用创建一个Consumer，由应用来维护此对象，可以设置为全局对象或者单例<br>
         * 注意：ConsumerGroupName需要由应用来保证唯一 ,
         * 最好使用服务的包名区分同一服务,一类Consumer集合的名称，这类Consumer通常消费一类消息，且消费逻辑一致
         * PushConsumer：Consumer的一种，应用通常向Consumer注册一个Listener监听器，
         * Consumer收到消息立刻回调Listener监听器
         * PullConsumer：Consumer的一种，应用通常主动调用Consumer的拉取消息方法从Broker拉消息，主动权由应用控制
         */
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer(
                "ConsumerGroupName");
        // //nameserver服务
        consumer.setNamesrvAddr("localhost:9876");
        consumer.setInstanceName("Consumer");
        //设置批量消费个数
        //consumer.setConsumeMessageBatchMaxSize(10);

        /**
         * 订阅指定topic下tags分别等于TagA或TagC或TagD
         */
        consumer.subscribe("TopicTest1", "TagA || TagC || TagD");
        /**
         * 订阅指定topic下所有消息<br>
         * 注意：一个consumer对象可以订阅多个topic
         *
         */
        consumer.subscribe("TopicTest2", "*");
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            public ConsumeConcurrentlyStatus consumeMessage(
                    List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                ///接收消息个数msgs.size()
                System.out.println(Thread.currentThread().getName()
                        + " Receive New Messages: " + msgs.size());
                MessageExt msg = msgs.get(0);
                if (msg.getTopic().equals("TopicTest1")) {
                    // 执行TopicTest1的消费逻辑
                    if (msg.getTags() != null && msg.getTags().equals("TagA")) {
                        // 执行TagA的消费
                        System.out.println(new String(msg.getBody()));
                    } else if (msg.getTags() != null
                            && msg.getTags().equals("TagC")) {
                        // 执行TagC的消费
                        System.out.println(new String(msg.getBody()));
                    } else if (msg.getTags() != null
                            && msg.getTags().equals("TagD")) {
                        // 执行TagD的消费
                        System.out.println(new String(msg.getBody()));
                    }
                } else if (msg.getTopic().equals("TopicTest2")) {
                    System.out.println(new String(msg.getBody()));
                }
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });

        /**
         * Consumer对象在使用之前必须要调用start初始化，初始化一次即可<br>
         */
        consumer.start();
        System.out.println("ConsumerStarted.");
    }
````

broadcast 消费者：

````
public class BroadCastPushConsumer {
    public static void main(String[] args) throws MQClientException {
        //
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("consumer");
        consumer.setNamesrvAddr("localhost:9876");
        consumer.setInstanceName("broadcastpushconsumer");
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
        consumer.setMessageModel(MessageModel.BROADCASTING);
        consumer.subscribe("TopicTest1","*");

        consumer.registerMessageListener(new MessageListenerConcurrently() {

            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs,
                                                            ConsumeConcurrentlyContext context) {
                System.out.println(Thread.currentThread().getName() + " Receive New Messages: " + msgs);

                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });

        consumer.start();
        System.out.println("Broadcast Consumer Started.");
    }
}
````

## 集群配置
推荐的几种 Broker 集群部署方式，这里的Slave 不可写，但可读，类似于 Mysql 主备方式。
### 1.单个 Master
  这种方式风险较大，一旦Broker 重启或者宕机时，会导致整个服务不可用，不建议线上环境使用。
### 2.多 Master 模式
  一个集群无 Slave，全是 Master，例如 2 个 Master 或者 3 个 Master

  优点：配置简单，单个Master 宕机或重启维护对应用无影响，在磁盘配置为 RAID10 时，即使机器宕机不可恢复情况下，由与 RAID10 磁盘非常可靠，消息也不会丢（异步刷盘丢失少量消息，同步刷盘一条不丢）。性能最高。

  缺点：单台机器宕机期间，这台机器上未被消费的消息在机器恢复之前不可订阅，消息实时性会受到受到影响。

####  先启动 NameServer，例如机器 IP 为：192.168.1.1:9876

`nohup sh mqnamesrv &`

####  在机器 A，启动第一个 Master

`nohup sh mqbroker -n 192.168.1.1:9876 -c$ROCKETMQ_HOME/conf/2m-noslave/broker-a.properties &`

####  在机器 B，启动第二个 Master

`nohup sh mqbroker -n 192.168.1.1:9876 -c$ROCKETMQ_HOME/conf/2m-noslave/broker-b.properties &`

### 3.多 Master 多 Slave 模式，异步复制
每个 Master 配置一个 Slave，有多对Master-Slave，HA 采用异步复制方式，主备有短暂消息延迟，毫秒级。

优点：即使磁盘损坏，消息丢失的非常少，且消息实时性不会受影响，因为 Master 宕机后，消费者仍然可以从 Slave 消费，此过程对应用透明。不需要人工干预。性能同多 Master 模式几乎一样。

缺点：Master 宕机，磁盘损坏情况，会丢失少量消息。
####  先启动 NameServer，例如机器 IP 为：192.168.1.1:9876
`nohup sh mqnamesrv &`

####  在机器 A，启动第一个 Master
`nohup sh mqbroker -n 192.168.1.1:9876 -c$ROCKETMQ_HOME/conf/2m-2s-async/broker-a.properties &`

####  在机器 B，启动第二个 Master
`nohup sh mqbroker -n 192.168.1.1:9876 -c$ROCKETMQ_HOME/conf/2m-2s-async/broker-b.properties &`

####  在机器 C，启动第一个 Slave
`nohup sh mqbroker -n 192.168.1.1:9876 -c$ROCKETMQ_HOME/conf/2m-2s-async/broker-a-s.properties &`

####  在机器 D，启动第二个 Slave
`nohup sh mqbroker -n 192.168.1.1:9876 -c$ROCKETMQ_HOME/conf/2m-2s-async/broker-b-s.properties &`

### 4.多 Master 多 Slave 模式，同步双写
每个 Master 配置一个 Slave，有多对Master-Slave，HA 采用同步双写方式，主备都写成功，向应用返回成功。

优点：数据与服务都无单点，Master宕机情况下，消息无延迟，服务可用性与数据可用性都非常高

缺点：性能比异步复制模式略低，大约低 10%左右，发送单个消息的 RT 会略高。目前主宕机后，备机不能自动切换为主机，后续会支持自动切换功能。

####  先启动 NameServer，例如机器 IP 为：192.168.1.1:9876
`nohup sh mqnamesrv &`

####  在机器 A，启动第一个 Master
`nohup sh mqbroker -n 192.168.1.1:9876 -c$ROCKETMQ_HOME/conf/2m-2s-sync/broker-a.properties &`

####  在机器 B，启动第二个 Master
`nohup sh mqbroker -n 192.168.1.1:9876 -c$ROCKETMQ_HOME/conf/2m-2s-sync/broker-b.properties &`

####  在机器 C，启动第一个 Slave
`nohup sh mqbroker -n 192.168.1.1:9876 -c$ROCKETMQ_HOME/conf/2m-2s-sync/broker-a-s.properties &`

####  在机器 D，启动第二个 Slave
`nohup sh mqbroker -n 192.168.1.1:9876 -c$ROCKETMQ_HOME/conf/2m-2s-sync/broker-b-s.properties &`

以上 Broker 与 Slave 配对是通过指定相同的brokerName 参数来配对，Master 的 BrokerId 必须是 0，Slave 的BrokerId 必须是大与 0 的数。另外一个 Master 下面可以挂载多个 Slave，同一 Master 下的多个 Slave 通过指定不同的 BrokerId 来区分。

$ROCKETMQ_HOST 指的 RocketMQ 安装目录，需要用户自己设置此环境变量

参考：

[RocketMQ安装和小试](http://blog.csdn.net/zhu_tianwei/article/details/40948447)

[消息中间件RocketMQ的安装与部署](http://singlefly.blog.51cto.com/4658189/1368579)
