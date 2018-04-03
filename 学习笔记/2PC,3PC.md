## Two-phase Commit
二阶段提交是指，为了使基于分布式系统架构下的所有节点在进行事务提交时保持一致性而设计的一种算法。通常，二阶段提交也被称为一种协议(Protocol)。在分布式系统中，每个节点可以知晓自己的操作是成功或失败，却无法知道其他节点的操作是否成功。当一个事务跨越多个节点时，为了保持事务的[ACID特性](https://zh.wikipedia.org/wiki/ACID)，需要引入一个作为协调者的组件来统一掌控所有节点（称作参与者）的操作结果，并最终指示这些节点是否要把操作结果进行真正的提交。因此二阶段提交的算法思路可以概括为：参与者将操作成败通知协调者，再由协调者根据所有参与者的反馈情况决定各参与者是否要提交操作还是终止操作。

### 前提
二阶段提交算法的成立基于以下假设：

+ 该分布式系统中，存在一个节点作为协调者(Coordinator)，其他节点作为参与者(Cohorts)。且节点之间可以进行网络通信。

+ 所有节点都采用[预写式日志](https://zh.wikipedia.org/wiki/%E9%A2%84%E5%86%99%E5%BC%8F%E6%97%A5%E5%BF%97),且日志被写入后即被保存在可靠的存储设备上，即使节点损坏不会导致日志数据的消失。

+ 所有节点不会永久性损坏，即使损坏后仍然可以恢复。

### 基本算法
以下对二阶段提交算法分阶段说明。

#### 第一阶段（提交请求阶段）
1. 协调者节点向所有参与者节点询问是否可以执行提交操作，并开始等待各参与者节点的响应。

2. 参与者节点执行询问发起为止的所有事务操作，并将Undo信息和Redo信息写入日志。

3. 各参与者节点回复协调者节点发起的询问。如果参与者节点的事务操作实际执行成功，则它返回一个“同意”信息；如果各参与者节点的事务操作实际执行失败，则它返回一个”中止“信息。

有时候第一阶段也称为投票阶段，即各参与者投票是否要继续接下来的提交操作。

#### 第二阶段（提交执行阶段）

**成功**

当协调者节点从所有参与者节点获得的相应消息都为”同意”时：

1. 协调者节点向所有参与者节点发出“正式提交”的请求。
2. 参与者节点正式完成操作，并释放在整个事务期间内占用的资源。
3. 参与者节点向协调者节点发送”完成“消息。
4. 协调者节点收到所有参与者节点反馈的“完成”信息后，完成事务。

**失败**

如果任一参与者节点在第一阶段返回的响应信息为”终止“,或者协调者节点在第一阶段的询问超时之前无法获取所有参与者节点的响应信息时：

1. 协调者节点向所有参与者节点发出“回滚操作”的请求。
2. 参与者节点利用之前写入的Undo信息执行回滚，并释放在整个事务期间占用的资源。
3. 参与者节点向协调者节点发送“回滚完成”信息
4. 协调者节点收到所有参与者节点反馈的“回滚完成”信息后，取消事务。

有时候，第二阶段也被称为完成阶段，因为无论结果怎样，协调者都必须在此阶段结束当前事务。

### 算法示意
下述流程图简单示意了二阶段提交算法中协调者和参与者之间的通信流程

````
协调者                                              参与者
                              QUERY TO COMMIT
                -------------------------------->
                              VOTE YES/NO           prepare*/abort*
                <-------------------------------
commit*/abort*                COMMIT/ROLLBACK
                -------------------------------->
                              ACKNOWLEDGMENT        commit*/abort*
                <--------------------------------  
end
````

`*`标记的操作意味着此类操作必须记录在稳固存储上。

### 缺点
二阶段提交算法的最大缺点就在于它的执行过程中间，节点都处于阻塞状态。节点之间在等待对方的相应消息时，它将什么也做不了。特别的是，当一个节点在已经占有了某项资源的情况下，为了等待其他节点的响应信息而陷入阻塞状态时，当第三个节点尝试访问该节点占用的资源时，这个节点也将连带陷入阻塞状态。

另外，协调者节点指示参与者节点进行提交等操作时，如有参与者节点出现了崩溃等情况而导致协调者始终无法获取所有参与者的响应信息，这时协调者将只能依赖协调者自身的超时机制来生效。但往往超时机制生效时，协调者都会指示参与者进行回滚操作。这样的策略显得比较保守。

### 协议优化
Database research has been done on ways to get most of the benefits of the two-phase commit protocol while reducing costs by protocol optimizations and protocol operations saving under certain system's behavior assumptions.

#### Presumed Abort and Presumed Commit
假定中止和假定提交是常见的优化手段。假定事务的结果，要么是commit,要么是Abort,可以节省2PC协议执行过程中参与方的消息和记日志操作。例如，当假定abort,如果在系统从failure恢复期间，没有任何的某些事务的commit记录的证据被恢复过程发现，那么他假设这个事务已经被abort了，然后执行相应的操作。这意味着，abort有没有被记日志都不重要，并且这类日志信息在这种假设下可以免去。通常在系统从failure恢复期间，有很多额外的操作要做，取决于优化的类型。因此，优化的最好的变体，如果有的话，是根据failure和事务结果统计来选择。

#### Tree two-phase commit protocol
The Tree 2PC 协议（也被称为嵌套2PC,或者递归2PC）是2PC在计算机网络中的一个常见变体，它更好的利用了底层的通信设施。The participants in a distributed transaction are typically invoked in an order which defines a tree structure, the invocation tree, where the participants are the nodes and the edges are the invocations (communication links). The same tree is commonly utilized to complete the transaction by a 2PC protocol, but also another communication tree can be utilized for this, in principle. In a tree 2PC the coordinator is considered the root ("top") of a communication tree (inverted tree), while the cohorts are the other nodes. The coordinator can be the node that originated the transaction (invoked recursively (transitively) the other participants), but also another node in the same tree can take the coordinator role instead. 2PC messages from the coordinator are propagated "down" the tree, while messages to the coordinator are "collected" by a cohort from all the cohorts below it, before it sends the appropriate message "up" the tree (except an abort message, which is propagated "up" immediately upon receiving it or if the current cohort initiates the abort).

#### The Dynamic two-phase commit (Dynamic two-phase commitment, D2PC) protocol
The Dynamic two-phase commit (Dynamic two-phase commitment, D2PC) protocol[2][6] is a variant of Tree 2PC with no predetermined coordinator. It subsumes several optimizations that have been proposed earlier. Agreement messages (Yes votes) start to propagate from all the leaves, each leaf when completing its tasks on behalf of the transaction (becoming ready). An intermediate (non leaf) node sends when ready an agreement message to the last (single) neighboring node from which agreement message has not yet been received. The coordinator is determined dynamically by racing agreement messages over the transaction tree, at the place where they collide. They collide either at a transaction tree node, to be the coordinator, or on a tree edge. In the latter case one of the two edge's nodes is elected as a coordinator (any node). D2PC is time optimal (among all the instances of a specific transaction tree, and any specific Tree 2PC protocol implementation; all instances have the same tree; each instance has a different node as coordinator): By choosing an optimal coordinator D2PC commits both the coordinator and each cohort in minimum possible time, allowing the earliest possible release of locked resources in each transaction participant (tree node).

## Three-phase commit
三阶段提交（英语：Three-phase commit），也叫三阶段提交协议（英语：Three-phase commit protocol），是在计算机网络及数据库的范畴下，使得一个分布式系统内的所有节点能够执行事务的提交的一种分布式算法。三阶段提交是为解决两阶段提交协议|的缺点而设计的。

与两阶段提交不同的是，三阶段提交是“非阻塞”协议。三阶段提交在两阶段提交的第一阶段与第二阶段之间插入了一个准备阶段，使得原先在两阶段提交中，参与者在投票之后，由于协调者发生崩溃或错误，而导致参与者处于无法知晓是否提交或者中止的“不确定状态”所产生的可能相当长的延时的问题得以解决。

![](./Three-phase_commit_diagram.png)

**协调者**

1. 协调者受到一个事务请求。如果此时failure，协调者abort这个事务（例如，一旦恢复，它会认为事务已经中止了）。否则，协调者发送一个`canCommit?`消息到参与者，并且转变到等待状态

2. 如果有failure,timeout,或者协调者在等待状态收到一个`No`消息，协调者中止事务，并且发送一个abort消息到所有参与者。否则，协调者将在时间窗口之内从所有参与者收到`Yes`消息，所以它发送`preCommit`消息到所有的参与者，并且转变到`prepared`状态。

3. 如果协调者在`prepared`状态成功了，它将转变到`commit`状态。然而，如果协调者在等待某个参与者的确认信息(ack)时超时，它将中止这个事务。在收到大多数参与者的ack信息的情况下，协调者也可以转变为`commit`状态。

**参与者**

1. 参与者从协调者收到一个`canCommit?`信息。如果参与者同意，则发送一个`Yes`消息到协调者并且转变到`prepared`状态。否则，它发送一个`No`消息并且中止。如果failure了，他转变为abort状态。

2. 在`prepared`状态，如果参与者从协调者收到一个abort信息，fails,或者等待commit信息超时，它abort。如果参与者收到一个`preCommit`信息，它返回一个ack信息并且等待最终的commit或abort.

3. 如果在参与者接收到`preCommit`信息后，协调者fail或time out，参与者继续commit。

### 动机
在Commit 阶段，二阶段提交协议不能独立从协调者或者一个参与者的failure中恢复。如果只有协调者failed，并且没有参与者收到commit信息，可以推测没有commit发生了。然而，如果协调者和一个参与者failed,如果这个参与者是第一个得到通知的，并且已经执行了commit。即使一个新的协调者选举了， 它也不能继续这个操作直到它从所有参与者收到同意提交的信息，因此，必须阻塞直到所有的参与者回复了。

3PC协议通过引入了`prepared commit`状态消除了这个问题。如果协调者在发送`preCommit`消息之前fail，参与者将一致的认为这个操作已经被中止了。协调者不会发送`doCommit`消息直到所有的参与者已经确认了他们`prepared` to commit。这消除了任何参与者在其他参与者接收到决定之前已经完成了这个事务的可能性。

### 缺点
主要的缺点是它不能从网络分割中恢复过来。最初的3PC算法假设了一个fail-stop的模型，在这个模型里，crash可以被精确的检测到。但是在网络分割或异步通信的情况下并不能奏效。

协议要求完成至少要3 round-trip，需要至少3 round-trip的时间。这对于完成整个事务可能是很大的延迟。


参考：

https://en.wikipedia.org/wiki/Two-phase_commit_protocol

https://en.wikipedia.org/wiki/Three-phase_commit_protocol
