### Configuring a Zookeeper Server
每一个Zookeeper server开始启动时从一个配置文件`zoo.cfg`中读取配置。`data`目录中的`myid`文件将server区分开来。每一个`data`目录对于server来说必须是唯一的，所以这是一个用来放区别性文件的方便的地方。包括在`myid`文件中的server Id作为配置文件的index，所以一个特定的Zookeeper server知道如何配置自己。当然，如果server有不同的设置（比如，他们存储transaction log到不同的地方），每一个必须有自己独特的配置文件。

These properties are set using the -D option when starting the server.

### Basic Configuration
有些参数没有默认值，必须为每一个部署配置。他们是：

+ clientPort
   客户端用于连接到服务器的TCP 端口。默认的端口是2181.

+ dataDir and dataLogDir
   dataDir是`fuzzy` snapshot存储的地方。如果这个服务器是集群的一部分，`myid`文件也会在这个目录。

   dataDir不需要位于一个专门的设备上。snapshot是被一个后台线程写入，不会lock database,并且写入存储不是同步的直到snapshot完成。

   除非`dataLogDir`选项被设置了，transaction log也存储在这个目录。

   最佳实践是使用一个专门的log 设备，设置`dataLogDir`指向这个设备上的一个目录。

+ tickTime
    一个tick的长度，单位毫秒。默认的tickTime是3000毫秒。
    The timeouts used by the ZooKeeper ensemble are specified in units of tickTime. This means, in effect, that the tickTime sets the lower bound on timeouts because the minimum timeout is a single tick. The minimum client session timeout is two ticks.

    降低tickTime允许更快的超时，但是也会导致更大的开支，在心跳和session bucket 处理。     

### Storage Configuration
+ preAlloSize
    事务log文件中预分配的大小（KB）.默认的预分配大小是64 MB。将这个值设小的一个原因是
    transaction log文件永远不会这么大。

+ snapCount

+ weight.x = n
和`group`选项一起使用，当组成Quorum时，设置server x的weight。值n是一个
Server投票时的weight。一个server默认的weight是1.如果配置文件里定义了group
但是没有定义weight，所有的servers的weight都是1.


### Cluster Configuration
+ initLimit
follower开始时连接到leader的超时时间，以tick为单位。

+ syncLimit
The timeout, specified in number of ticks, for a follower to sync with a leader.

+ server.x=[hostname]:n:n[:observer/participant]

Sets the configuration for server x.

ZooKeeper servers need to know how to communicate with each other. A config‐ uration entry of this form in the configuration file specifies the configuration for a given server x, where x is the ID of the server (an integer). When a server starts up, it gets its number from the myid file in the data directory. It then uses this number to find the server.x entry. It will configure itself using the data in this entry. If it needs to contact another server, y, it will use the information in the server.y entry to contact the server.

The hostname is the name of the server on the network n. There are two TCP port numbers. The first port is used to send transactions, and the second is for leader election. The ports typically used are 2888:3888. If observer is in the final field, the server entry represents an observer.

Note that it is quite important that all servers use the same server.x configuration; otherwise, the ensemble won’t work properly because servers might not be able to establish connections properly.


### Configuring a Zookeeper Ensemble
We can avoid the split-brain problem by requiring that all quorums have at least a majority of servers. (Note: half of the servers do not constitute a majority; you must have greater than half the number of servers to have a majority.)

The following configuration option creates a group:

+ group.x = n[:n]
    Enables a hierarchical quorum construction. x is a group identifier and the numbers following the equals sign correspond to server identifiers. The right side of the assignment is a colon-separated list of server identifiers. Note that groups must be disjoint and the union of all groups must be the ZooKeeper ensemble. In other words, every server in the ensemble must be listed once in some group.

    Here is an example of nine servers split across three different groups:
    group.1=1:2:3
    group.2=4:5:6
    group.3=7:8:9

    A configuration like this has a variety of benefits when we are deploying ZooKeeper across different data centers. For example, one group may represent a group of servers running in a different data center, and if that data center goes down, ZooKeeper can keep going.
