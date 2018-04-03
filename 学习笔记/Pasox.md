## Paxos
Paxos算法是Leslie Lamport于1990年提出的一种基于消息传递且具有高度容错特性的一致性算法。

先解释一个名词，拜占庭错误：消息可能会延迟，可能会重复，但不能被篡改。

分布式系统中的节点通信存在两种模型：共享内存(shared memory)和消息传递(message passing)。基于消息传递的分布式系统，不可避免的会发生一下错误：进程可能会变慢，可能被杀死(killed)或者crash，消息可能延迟，丢失，重复，在basic Paxos场景中，先不考虑可能出现消息篡改即拜占庭错误的情况。Paxos算法解决的问题是在一个可能发生上述异常的分布式系统中如何就某个值达成一致，保证即使发生以上任何异常，都不会破坏决议的一致性。

Paxos的推导过程可以参考Leslie Lamport的[paxos-simple](http://lamport.azurewebsites.net/pubs/paxos-simple.pdf)和[Paxos算法](https://zh.wikipedia.org/wiki/Paxos%E7%AE%97%E6%B3%95)

### 角色
在Paxos中有五种角色：client,acceptor,proposer,learner和leader（只存在于multi-Paxos中）。在通常的实现中，一个processor可能同时扮演一个或多个角色。这并不影响协议的正确性--通常会合并角色来改善延迟和协议中的消息数。

**Client**

客户端发起一个request到分布式系统，并且等待response.比如，对分布式文件服务器上的文件的写请求。

**Acceptor(Voters)**

acceptor作为协议的容错"memory"。acceptors可以组成Quorums(Quorum的定义在下面).任何发送到一个acceptor的消息必须发送到a Quorum of acceptors。从一个acceptor收到的任何消息会被忽略除非接收到来自一个Quorum中每一个acceptor的拷贝（比如，Promises sent to a Proposer, or Accepted messages sent to a Learner）。

**Proposer**

一个Proposer支持Client的request,试图说服Acceptors同意，并且在冲突发生时作为协调者将协议继续。

**Learner**

Learner作为协议中的复制要素(replication factor)。一旦一个Clinet request被Acceptors同意，Learner可以采取措施(比如，执行这个request并且发送response到Clinet)。为了提高处理的可用性，可以添加额外的Learners。

**Leader**

Paxos要求一个distinguished Proposer（称为leader）来make progress。很多processes可能相信它们是leaders,但是协议只保证它们中的一个被选中。如果两个相信它们是leaders，它们可能通过不停的发起冲突的提议从而拖延协议。然而，安全属性在这种情况下仍然是保持的。

**Quorums**
Quorums express the safety properties of Paxos by ensuring at least some surviving processor retains knowledge of the results.

Quorums are defined as subsets of the set of Acceptors such that any two subsets (that is, any two Quorums) share at least one member. Typically, a Quorum is any majority of participating Acceptors. For example, given the set of Acceptors {A,B,C,D}, a majority Quorum would be any three Acceptors: {A,B,C}, {A,C,D}, {A,B,D}, {B,C,D}. More generally, arbitrary positive weights can be assigned to Acceptors and a Quorum defined as any subset of Acceptors with the summary weight greater than half of the total weight of all Acceptors.

**Proposal Number & Agreed Value**

一个acceptor可能接受不同的Proposal，为了区别不同的Proposal,以一个整数表示他的number。一个Proposal的Value是Proposer指定的值。所以一个Proposal包括一个Number和一个Value。

### Safety and liveness properties
为了保证安全性，Paxos定义了三种安全属性并且保证他们总是保持，无论哪种failure:

**Non-triviality**

只有被提出的值可以被learned.

**Safety**

最多只有一个值被learned

**Liveness**
如果值C已经被提出，那么最终learner L 将learn到某个值（如果足够多的process 仍然没有出错）

### Typical deployment
在大多数的Paxos的部署中，每一个参与的process扮演三个角色：Proposer,Acceptor和Learner。这大大的减少了消息复杂性，但是没有牺牲正确性：

> 在Paxos中，客户端发送命令到一个leader。在正常的运行中，leader接收客户端的命令，分配一个新的命令number i,然后开始一致性算法的第i个实例，通过发送消息到一个acceptor process集合。

通过合并角色，协议合并成一个有效的 client-master-replica 型的部署，典型的数据库。The benefit of the Paxos protocols (including implementations with merged roles) is the guarantee of its safety properties.

一个典型的实现的消息流包括在Multi-Paxos小节。

### Basic Paxos
这个协议是Paxos家族中最基本的。Basic Paxos协议的每一个实例决定一个输出值。协议进行了几轮(round)。一个成功的round有两个阶段。一个提议者(Proposer)不应该开始Paxos，如果它不能和a Quorum of Acceptors通信。

**阶段 1a: Prepare**

一个Proposer(the leader)创建一个number为n的提议。这个number必须比这个Proposer任何之前提议的number大。然后，它发送一个包含这个提议的`Prepare`消息到a Quorum of Acceptors。提议者决定哪些acceptor在这个Quorum中。

**阶段 1b: Promise**

如果这个提议的number比这个acceptor之前收到的来自任何提议者的提议number都大，那么这个Acceptor必须返回一个Promise：它将忽略将来的number比n小的提议。如果这个Acceptor在过去的某个点已经接受了一个提议，那么它必须将之前的提议的number和value包含在response里返回给Proposer。

**阶段 2a: Accept Request**

如果一个提议者从a Quorum of Acceptors收到足够多的`Promise`，他需要设置一个value到它的提议。如果任何Acceptors之前已经接受了任何的提议，那么这个Acceptor将发送它的value给这个提议者，提议者将之前提议中最大的number最大的提议的value设置为它的提议的value。如果直到此时没有任何Acceptors已经接受了一个提议，那么提议者可以自己选择一个值作为提议的值。

提议者发送一个Accept Request（包含了提议）消息到a Quorum of Acceptors。

**阶段 2b:Accepted**

如果一个Acceptor收到了提议N（number为n）的Accept Request,当且仅当它还没有`Promise`任何number大于n的 `Prepare`提议时，任何它必须接受这个提议。在这种情况下，它应该记住对应的value v并且发送一个`Accepted`消息到提议者和每一个Learner。否则它可以忽略这个Accept Request。

记住，一个Acceptor可以接受多个提议。由于某些failure,这些提议可能有不同的值。然而，Paxos协议保证Acceptors将只会批准一个值。

当多个提议者发送冲突的`Prepare`消息时，或者当提议者没有收到a Quorum of response（Promise或Accepted）时，round fail。在这种情况下，另一个round必须以更大的number开始。

注意，当Acceptors接受一个request时，他们也确认了这个提议者的leadership。因此，Paxos可以用于选择集群中的leader。

A Learner learns a value only if it receives a Quorum of Accepted messages with the same value.

### Message flow: Basic Paxos
(first round is successful)

````
Client   Proposer      Acceptor     Learner
   |         |          |  |  |       |  |
   X-------->|          |  |  |       |  |  Request
   |         X--------->|->|->|       |  |  Prepare(1)
   |         |<---------X--X--X       |  |  Promise(1,{Va,Vb,Vc})
   |         X--------->|->|->|       |  |  Accept!(1,Vn)
   |         |<---------X--X--X------>|->|  Accepted(1,Vn)
   |<---------------------------------X--X  Response
   |         |          |  |  |       |  |
````

Vn = highest of (Va,Vb,Vc)

### Error cases in Basic Paxos
最简单的错误的情况是，一个冗余Learner fail或者一个Acceptor fail（当a Quorum of Acceptors 仍然live）。在这种情况下，协议无须恢复。没有任何额外的round或消息是需要的，如下所示：

#### Message flow: Basic Paxos, failure of Acceptor

(Quorum size = 2 Acceptors)

````
Client   Proposer      Acceptor     Learner
   |         |          |  |  |       |  |
   X-------->|          |  |  |       |  |  Request
   |         X--------->|->|->|       |  |  Prepare(1)
   |         |          |  |  !       |  |  !! FAIL !!
   |         |<---------X--X          |  |  Promise(1,{null,null})
   |         X--------->|->|          |  |  Accept!(1,V)
   |         |<---------X--X--------->|->|  Accepted(1,V)
   |<---------------------------------X--X  Response
   |         |          |  |          |  |
````

#### Message flow: Basic Paxos, failure of redundant Learner

````
Client   Proposer      Acceptor     Learner
   |         |          |  |  |       |  |
   X-------->|          |  |  |       |  |  Request
   |         X--------->|->|->|       |  |  Prepare(1)
   |         |<---------X--X--X       |  |  Promise(1,{null,null,null})
   |         X--------->|->|->|       |  |  Accept!(1,V)
   |         |<---------X--X--X------>|->|  Accepted(1,V)
   |         |          |  |  |       |  !  !! FAIL !!
   |<---------------------------------X     Response
   |         |          |  |  |       |   
````

The next failure case is when a Proposer fails after proposing a value, but before agreement is reached. Ignoring Leader election, an example message flow is as follows:

#### Message flow: Basic Paxos, failure of Proposer
(re-election not shown, one instance, two rounds)

````
Client  Proposer        Acceptor     Learner
   |      |             |  |  |       |  |
   X----->|             |  |  |       |  |  Request
   |      X------------>|->|->|       |  |  Prepare(1)
   |      |<------------X--X--X       |  |  Promise(1,{null, null, null})
   |      |             |  |  |       |  |
   |      |             |  |  |       |  |  !! Leader fails during broadcast !!
   |      X------------>|  |  |       |  |  Accept!(1,Va)
   |      !             |  |  |       |  |
   |         |          |  |  |       |  |  !! NEW LEADER !!
   |         X--------->|->|->|       |  |  Prepare(2)
   |         |<---------X--X--X       |  |  Promise(2,{null, null, null})
   |         X--------->|->|->|       |  |  Accept!(2,V)
   |         |<---------X--X--X------>|->|  Accepted(2,V)
   |<---------------------------------X--X  Response
   |         |          |  |  |       |  |
````

The most complex case is when multiple Proposers believe themselves to be Leaders. For instance the current leader may fail and later recover, but the other Proposers have already re-elected a new leader. The recovered leader has not learned this yet and attempts to begin a round in conflict with the current leader.

#### Message flow: Basic Paxos, dueling Proposers
(one instance, four unsuccessful rounds)

````
Client   Proposer        Acceptor     Learner
   |      |             |  |  |       |  |
   X----->|             |  |  |       |  |  Request
   |      X------------>|->|->|       |  |  Prepare(1)
   |      |<------------X--X--X       |  |  Promise(1,{null,null,null})
   |      !             |  |  |       |  |  !! LEADER FAILS
   |         |          |  |  |       |  |  !! NEW LEADER (knows last number was 1)
   |         X--------->|->|->|       |  |  Prepare(2)
   |         |<---------X--X--X       |  |  Promise(2,{null,null,null})
   |      |  |          |  |  |       |  |  !! OLD LEADER recovers
   |      |  |          |  |  |       |  |  !! OLD LEADER tries 2, denied
   |      X------------>|->|->|       |  |  Prepare(2)
   |      |<------------X--X--X       |  |  Nack(2)
   |      |  |          |  |  |       |  |  !! OLD LEADER tries 3
   |      X------------>|->|->|       |  |  Prepare(3)
   |      |<------------X--X--X       |  |  Promise(3,{null,null,null})
   |      |  |          |  |  |       |  |  !! NEW LEADER proposes, denied
   |      |  X--------->|->|->|       |  |  Accept!(2,Va)
   |      |  |<---------X--X--X       |  |  Nack(3)
   |      |  |          |  |  |       |  |  !! NEW LEADER tries 4
   |      |  X--------->|->|->|       |  |  Prepare(4)
   |      |  |<---------X--X--X       |  |  Promise(4,{null,null,null})
   |      |  |          |  |  |       |  |  !! OLD LEADER proposes, denied
   |      X------------>|->|->|       |  |  Accept!(3,Vb)
   |      |<------------X--X--X       |  |  Nack(4)
   |      |  |          |  |  |       |  |  ... and so on ...
````

### Multi-Paxos
在Multi-Paxos里，我们需要对一系列值进行决策，将决策的值append 到log中。并且为了保证在某个Server宕机后，系统仍然可用，我们需要在多个Servers上有完整的log。

选择一系列独立的log entry然后将它们合并成一个log.

1. 连续决策多个值，同时确保每个Servers上的顺序完全一致。

2. 以相同的顺序执行命令，所有Servers状态一致。

3. Servers可以并发的处理客户端的请求。其实这里并发只是append log 的并发提高, 应用的state machine 还是必须是有序的)

4. Raft选主是有限制的, 必须有最新, 最全的日志节点才可以当选. 而multi-paxos 是随意的 所以raft 可以看成是简化版本的multi paxos(这里multi-paxos 因为允许并发的写log, 因此不存在一个最新, 最全的日志节点, 因此只能这么做. 这样带来的麻烦就是选主以后, 需要将主里面没有的log 给补全, 并执行commit 过程)

如果每一个命令是Basic Paxos协议的一个实例的结果，那么会产生很大的开支。论文`Paxos Made Simple`定义了通常被称为`Multi-Paxos`的Paxos，它在稳定的状态时使用一个`distinguished` leader来协调一连串的命令。Paxos的一个典型部署使用一连续的批准的value作为命令来更新一个分布式的状态机。

如果leader相对稳定，阶段1就变得没有必要了。因此，有同一个leader的实例可以跳过阶段1。

To achieve this, the instance number I is included along with each value. Multi-Paxos reduces the failure-free message delay (proposal to learning) from 4 delays to 2 delays.

#### Message flow: Multi-Paxos, start
(first instance with new leader)

````
Client   Proposer      Acceptor     Learner
   |         |          |  |  |       |  | --- First Request ---
   X-------->|          |  |  |       |  |  Request
   |         X--------->|->|->|       |  |  Prepare(N)
   |         |<---------X--X--X       |  |  Promise(N,I,{Va,Vb,Vc})
   |         X--------->|->|->|       |  |  Accept!(N,I,Vm)
   |         |<---------X--X--X------>|->|  Accepted(N,I,Vm)
   |<---------------------------------X--X  Response
   |         |          |  |  |       |  |
````

Vm = highest of (Va, Vb, Vc)

#### Message flow: Multi-Paxos, steady-state
(subsequent instances with same leader)

````
Client   Proposer       Acceptor     Learner
   |         |          |  |  |       |  |  --- Following Requests ---
   X-------->|          |  |  |       |  |  Request
   |         X--------->|->|->|       |  |  Accept!(N,I+1,W)
   |         |<---------X--X--X------>|->|  Accepted(N,I+1,W)
   |<---------------------------------X--X  Response
   |         |          |  |  |       |  |
````

#### Typical Multi-Paxos Collapsed Roles deployment
Paxos家族最常见的部署是Multi-Paxos,指定参与的processors每个都是Proposers,Acceptors和Learners。角色重叠的消息流可以被优化为如下所示：

#### Message flow: Multi-Paxos Collapsed Roles, start
(first instance with new leader)

````
Client      Servers
   |         |  |  | --- First Request ---
   X-------->|  |  |  Request
   |         X->|->|  Prepare(N)
   |         |<-X--X  Promise(N,I,{Va,Vb})
   |         X->|->|  Accept!(N,I,Vn)
   |         |<-X--X  Accepted(N,I)
   |<--------X  |  |  Response
   |         |  |  |
````

#### Message flow: Multi-Paxos Collapsed Roles, steady state
(subsequent instances with same leader)

````
Client      Servers
   X-------->|  |  |  Request
   |         X->|->|  Accept!(N,I+1,W)
   |         |<-X--X  Accepted(N,I+1)
   |<--------X  |  |  Response
   |         |  |  |
````
