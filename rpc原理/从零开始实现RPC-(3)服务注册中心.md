在上一篇中，我们实现了基本的RPC客户端调用服务端。

在启动阶段，客户端从注册中心拉取对应服务的列表，服务端进行服务注册。

本篇，我们打算使用Zookeeper实现服务注册中心。服务注册中心用来维护服务器列表，并且在服务器上下线、扩容的时候同步变更，定时通知客户端更新列表。

关于注册中心，在第二篇中提过：注册中心必须保证高可用，不能因为一两台机器挂机就不可用，也就是需要集群冗余。对此，业界已经有成熟的解决方案Zookeeper。

Zookeeper是Apache下的一个分布式协作框架，它的特点如下：
* 它有一套选举机制可以保证在zookeeper集群只要有一半以上机器可用就可以正常提供服务，可用性非常高
* 其数据存储在内存当中，读写速度很快
* zookeeper使用起来很像一个文件系统，其节点Znode就像是文件系统中的文件夹和文件，节点的路径很像文件的路径比如/root/node1/node1_1，一个Znode下有自己的数据，同时还可以有子节点
* 订阅者/发布者 当客户端连接zookeeper并注册了所关注的节点/路径，在节点/路径发生变化时，客户端会收到一个通知
* 可以用来保存分布式应用的一些状态，实现统一的配置中心，分布式锁等

综上，其高可用性，支持集群冗余，订阅/发布的功能都是注册中心所需要的。

### 服务注册中心的实现
我们的服务注册中心的主要功能是：
+ 服务端启动的时候进行注册，关闭时注销
+ 客户端启动时从注册中心拉取对应服务的列表，在服务端列表发生变更时同步变更

如何用Zookeeper实现呢？思路是我们在Zookeeper服务器上建立对应的一服务全名命名的节点，路径如：

````
/[项目名]/provider/com.XX.XX.api.XXService/
/[项目名]/consumer/com.XX.XX.api.XXService/
````

然后在该节点下建立子节点代表提供该服务的服务器的信息

````
/[项目名]/provider/com.XX.XX.api.XXService/127.0.0.1:9090
/[项目名]/consumer/com.XX.XX.api.XXService/127.0.0.1:9090
````

新服务上线时就新建对应的节点，下线时移除对应的节点。客户端启动时，拉取service对应zookeeper下的节点，初始化服务列表，在节点发生变化时，客户端收到对应的变更事件，同步更新这个列表。

假设我们已经有一个zookeeper集群在运行了，要操作zookeeper，我们需要在代码中调用zookeeper的客户端，本框架中使用的是Netfilx的Curator框架，它在zookeeper客户端的基础上做了很多封装，可以简化zookeeper客户端的使用。

### 1.服务端启动的时候进行注册，下线时注销
首先，服务端启动时需要连接zookeeper:
````
public class RegistryManager{
  ...
  public static void start() throws Exception{
      String zookeeperAddress = ConfigManager.getInstance().getProperty(Constants.ZOOKEEPER_ADDRESS);
      client = CuratorFrameworkFactory.newClient(zookeeperAddress, new ExponentialBackoffRetry(1000, 3));
      client.start();
      ...
      started = client.blockUntilConnected(1000, TimeUnit.MILLISECONDS);
    }
}
````

这里的zookeeper地址及端口需要通过配置文件的形式填入，ConfigManager会在初始化时读取classpath下的zing.properties配置文件registry.zookeeper.address的值，形式如”registry.zookeeper.address=127.0.0.1:2181”

在服务端启动后，加载各个service实现类时，即可向服务注册中心注册该服务器可提供的服务：

````
...
String currentServerPath = ZookeeperPathUtils.formatProviderPath(serviceName, ip, port);
String result = client.create().creatingParentsIfNeeded().withMode(CreateMode.EPHEMERAL).forPath(currentServerPath);
...
````

根据当前加载的serviceBean得到serviceName、当前机器IP和rpc端口生成对应的zookeeper路径，调用curator创建节点，这里创建的zookeeper节点类型是EPHEMERAL的，即临时节点，当服务端和zookeeper的连接断开时，该节点即会自动从zookeeper去除，这样在服务端下线、宕机、网络连接出现问题而不能正常提供服务时，注册中心就会自动将该服务器地址移除。

### 2.客户端上线时从注册中心拉取对应服务的列表，在服务端列表发生变更时同步变更
客户端上线时需要拉到service对应的provider列表，可以在客户端的service proxy bean在spring初始化的时候来进行

````
Class<?> serviceClass = Class.forName(serviceName);
ServiceProviderManager.initServerListOfService(serviceName);
return Reflection.newProxy(serviceClass, new ServiceProxy(serviceName));
````

底层调用了curator拉取了/zing/service/org.zxc.zing.demo.api.DemoService下的所有子节点，转换成service provider list存储在内存中，为了保证线程安全，这里用一个以serviceName为key, 包含对应service provider的set为value的ConcurrentHashMap来存储，每次列表发生变动时，生成新的set替换掉map中的值。

````
List<String> stringList = client.getChildren().watched().forPath(String.format(Constants.SERVICE_ZK_PATH_FORMAT, serviceName));
````

在客户端启动的时候需要监控zookeeper的/zing/service路径，当这个路径下的节点发生变化时，客户端需要能够得到通知。

````
String zookeeperAddress = ConfigManager.getInstance().getProperty(Constants.ZOOKEEPER_ADDRESS);
RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
client = CuratorFrameworkFactory.newClient(zookeeperAddress, retryPolicy);
client.start();

TreeCache treeCache = TreeCache.newBuilder(client, Constants.SERVICE_ZK_PATH_PREFIX).setCacheData(false).build();
treeCache.getListenable().addListener(new ProviderNodeEventListener(), curatorEventThreadPool);
treeCache.start();

started = client.blockUntilConnected(1000, TimeUnit.MILLISECONDS);
````

这里在启动，注册一个TreeCache，关注/zing/service路径下节点的变化，当这个路径下有节点创建/去除时，ProviderNodeEventListener会收到对应的事件。

````
public void childEvent(CuratorFramework client, TreeCacheEvent event) throws Exception {
if (!isServiceProviderNodeChangeEvent(event)) {
    return;
}
String serviceName = ZookeeperPathUtils.getServiceNameFromProviderZkPath(event.getData().getPath());
if (Strings.isNullOrEmpty(serviceName)) {
    return;
}
ProviderStateListenerManager.getInstance().onProviderChange(serviceName);
}
````

收到zookeeper节点变动事件时过滤一层，只关心`/[项目名]/`路径下的，且必须是服务器地址节点这一层的新增/删除事件，这样拿到事件就可以知道是哪个service对应的provider list发生了变化，和启动时一样，重新拉取一次该service下的节点，覆盖原来的provider list。

至此，一个rpc框架的基本功能就已经实现了，这个版本的tag是”registry-center”。 一个强大的rpc框架还需要一些诸如服务监控、限流、隔离、降级、心跳检测等，之后有需要会逐步完善。
