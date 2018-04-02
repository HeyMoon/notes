Name Server是专为RocketMQ设计的轻量级名称服务，具有简单、可集群横向扩展、无状态等特点。

## Name Server存储的信息
在Name Server中，存储了这几类信息：

+ topic和topic对应的QueueData list。用`HashMap<String/* topic */, List<QueueData>> topicQueueTable`表示。其中QueueData中存储了brokerName和权限（可读，可写）等信息。
+ brokerName和brokerName 对应的broker data。用`HashMap<String/* brokerName */, BrokerData> brokerAddrTable`表示。

>在RocketMQ中，Broker分为Master和Slave,一个Master可以对应多个Slave，但是一个Slave只能对应一个Master，Master和Slave的对应关系通过指定相同的 BrokerName,不同的 BrokerId 来定义,BrokerId为 0 表示 Master,非 0 表示 Slave。Master 也可以部署多个。

+ 一个集群和其中的Broker的信息。用`HashMap<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable`表示。
+ 一个Broker的实时信息。用`HashMap<String/* brokerAddr */, BrokerLiveInfo> brokerLiveTable`表示。
+ Broker和其部署的Filter Server的信息。用`HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable`表示。


## Name Server启动。
NameServer启动分为两个步骤：

1. NamesrvController.initialize()-->

    1. kvConfigManager.load() 从`$user.home/namesrv/kvConfig.json`中加载KV配置。如果有，则加载。
    2. `new NettyRemotingServer(nettyServerConfig,brokerHousekeepingService)`
    3. 初始化remotingExecutor。remotingExecutor为Executors.newFixedThreadPool(NUMS),NUMS默认为8.
    4. registerProcessor()。注册DefaultRequestProcessor为请求处理类，处理Broker和Client的各类请求。
    5. 定时任务：初始延时5秒，每隔10秒重新执行,`routeInfoManager.scanNotActiveBroker()`，关闭不活跃时间超过2分钟的broker的channel。
    6. 定时任务：初始延时1分钟，每隔10分钟重新执行,`kvConfigManager.printallPeriodically()`,打印KV配置。

2. NamesrvController.start()--> remotingServer.start()-->
    1. 启动Netty服务端，监听**9876**端口。
    2. 定时任务：初始延时3秒，每一秒执行一次，从`responseTable`中遍历出超时时间小于一秒的`ResponseFuture`,调用`ResponseFuture.release()`,`ResponseFuture.executeInvokeCallback()`方法。

## 服务发现
1. 如果 Broker 指定了NAMESRV_ADDR，则更新name address。否则每两分钟获取name address(通过向这个地址：http://jmenv.tbsite.net:8080/rocketmq/nsaddr发送HTTP请求)
2. 每个 Broker 与 Name Server 集群中的所有节点建立长连接,定时注册 Topic 信息到所有 Name Server。

````
this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

    @Override
    public void run() {
        try {
            BrokerController.this.registerBrokerAll(true, false);
        } catch (Throwable e) {
            log.error("registerBrokerAll Exception", e);
        }
    }
}, 1000 * 10, 1000 * 30, TimeUnit.MILLISECONDS);
````

3. Producer 与 Name Server 集群中的其中一个节点(随机选择)建立长连接,定期从 Name Server 取 Topic 路由信息,并向提供 Topic 服务的 Master 建立长连接,且定时向 Master 发送心跳。

4. Consumer 与 Name Server 集群中的其中一个节点(随机选择)建立长连接,定期从 Name Server 取 Topic 路由信息,并向提供 Topic 服务的 Master、Slave 建立长连接,且定时向 Master、Slave 发送心跳。

定期从Name Server获取 TopicRouteData：

````
this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

    @Override
    public void run() {
        try {
            MQClientInstance.this.updateTopicRouteInfoFromNameServer();
        } catch (Exception e) {
            log.error("ScheduledTask updateTopicRouteInfoFromNameServer exception", e);
        }
    }
}, 10, this.clientConfig.getPollNameServerInteval(), TimeUnit.MILLISECONDS);
````

定期向 Master(Master、Slave) 发送心跳：

````
this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

    @Override
    public void run() {
        try {
            MQClientInstance.this.cleanOfflineBroker();
            MQClientInstance.this.sendHeartbeatToAllBrokerWithLock();
        } catch (Exception e) {
            log.error("ScheduledTask sendHeartbeatToAllBroker exception", e);
        }
    }
}, 1000, this.clientConfig.getHeartbeatBrokerInterval(), TimeUnit.MILLISECONDS);
````
