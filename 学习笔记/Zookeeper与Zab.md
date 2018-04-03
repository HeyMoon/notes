客户端的每一个改变Zookeeper Server的request都会被封装为一个Transaction,Transaction构成了执行这个请求的步骤。举个例子，客户端在一个znode `/z`上提交一个`setData`请求。`setData`应该改变这个znode的数据，并且增加版本号。所以这个request的transaction应该包括两个部分：znode的新的数据和znode的新的版本号。

Transaction被当做一个单元，它包含的所有的改变必须被原子的应用。当一个Zookeeper集群应用一个Transaction时，他确保所有的改变被原子的改变，并且不会干涉其他的Transaction。

Transaction是幂等的。

当Leader生成一个新的Transaction时，他分配一个新的transaction id给这个Transaction，称为zxid。zxid标识了Transaction,所以他们被leader以创建的顺序应用于server。

zxid是一个long型的整数，分为两部分:`epoch`和`counter`。每一个部分是32位。epoch和counter的用处会在我们讨论Zab时变得清楚，Zab是我们用于广播状态更新到servers的协议。

## Leader Election
leader的目的是为改变Zookeeper状态的客户端请求排序:create,setData,delete。leader将每一个request转变为transaction，正如在上一节中解释的，将transaction提议给follower,以被leader提议的顺序应用他们。

为了获得leadership，一个Server必须获得大部分servers的支持。

每一个Server 启动时都是LOOKING状态，它要么选举一个新的leader,要么找到一个现有的leader.如果leader已经存在，其他servers告诉这个新加入的server哪个Server是leader。此时，这个新加入的server连接到leader,保证它的状态和leader一致。

如果一个集群中的server都是LOOKING状态，他们必须通信来选出一个新的leader。赢得选举的server进入LEADING状态，而集群中的其他servers进入FOLLOWING状态。

Leader节点也会检测Follower节点的状态，如果多数Follower节点不再响应Leader节点（可能是Leader节点与Follower节点之间产生了网络分区），那么Leader节点可能此时也不再是合法的Leader了，也必须要进行一次新的选主。

> Leader节点启动时会接收Follower的主动连接请求，对于每一个Follower的新连接，Leader会创建一个LearnerHandler对象来处理与该Follower的消息通信。

Leader选举消息被称为 `leader election Notification`或者简单的称为`Notification`。协议非常简单，当一个server进入LOOKING状态时，它发送信息到集群中的其他服务器。信息包括current vote,`Vote`包括Server的id(sid)和它最近执行的transaction的zxid，以及epoch（对于leader election，zxid是一个简单的number，但是在其他协议中zxid代表epoch 和 counter）。

接收到`Vote`的server可能处于LOOKING状态，或者也可能处于FOLLOWING或LEADING状态。我们在下面详述

### 处于LOOKING状态
如果接收到`Vote`的server处于LOOKING状态，那么这个server也可以发起自己的`Vote`。一旦接收到`Vote`,server根据下面的规则更改他们自己的`Vote`。

1. 假设`voteId`,`voteZxid`和`new epoch`是`Vote`中的id和zxid,`mySid`和`myZxid`,`current epoch`是接收者自己的id.

2. 如果`new epoch > current epoch`
   1. 如果`voteZxid > myZxid`或`voteZxid = myZxid && voteId > mySid`，则更新自己的`Vote`,将`voteId`复制给`mySid`,`voteZxid`复制给`myZxid`,`new epoch`复制给`current epoch`,发送`Leader election Notification`。

   2. 否则的话，保持自己的`mySid`,`myZxid`,`current epoch`，发送`Leader election Notification`。

3. 如果`new epoch < current epoch`,不回复。也不发送`leader election Notification`.

4. 如果`new epoch = current epoch`,那么比较`voteId`,`voteZxid`,`new epoch`和这个server前一个`Vote`的`proposedLeader, proposedZxid, proposedEpoch`。如果`voteZxid > proposedZxid`或`voteZxid = proposedZxid && voteId > proposedLeader`，则更新自己的`Vote`,将`voteId`复制给`proposedLeader`,`voteZxid`复制给`proposedZxid`,`new epoch`复制给`proposedEpoch`,发送`Leader election Notification`。

很快的，最新的server将会赢得选举，因为他有最新的zxid。如果多个server有最新的zxid，sid最大的那个赢得选举。

一旦一个server从集群中的大部分服务器接收到相同的`Vote`，这个Server宣称leader已经选出了。如果这个选出的leader是这个server自己，它开始执行leader角色。否则，它成为一个follower并且开始连接到这个选出的leader。不保证follower可以连接到这个选出的leader。例如，这个选出的leader可能crash了。一旦它连接了，follower和leader同步状态，只有在同步状态之后，follower才可以开始处理新的请求。

### 处于FOLLOWING或LEADING状态
如果一个server处于FOLLOWING或LEADING状态，说明此时已经有leader存在了。那么我们只需要告诉这个发起`leader election Notification`的server哪个server是leader就可以了。

先说明一下，一个节点处于LOOKING状态可能有两种原因：

1. 这个节点是新加入到集群中的.

2. 这个节点没有接受到Leader的心跳。可能是Leader crash了，或者网络问题。

如果接收到`Vote`的server处于FOLLOWING或LEADING状态的处理流程是这样的：

1. 如果vote 的`epoch`等于server的`epoch`，那么说明接收到vote的这个server之前可能也是出于LOOKING状态，然后选出leader,这个server才转变为FOLLOWING或LEADING状态。所以我们要找出谁是Leader。

> 因为Server发送`leader election Notification`时，epoch要自增1。如果是新加入集群的server，他们的epoch应该是初始值，不可能等于集群中任何一个server的epoch，所以发送Vote的server不可能是新加入集群的。那么就只能是因为server 没有接收到leader的心跳。因为Server发送`leader election Notification`时，epoch要自增1。所以发送Vote的server要比集群中的其他Follower的epoch大，如果vote 的`epoch`等于server的`epoch`，那么说明接收到vote的这个server之前也是处于LOOKING状态。

2. 如果vote 的`epoch`不等于server的`epoch`,那么说明发送`Vote`的这个server是新加入到集群中（`vote epoch < server epoch`）或者这个server出现了网络隔离导致其触发一次新的选主，但是系统中其他节点状态依然正常(`vote epoch > server epoch`）。这时我们只需要告诉它，谁是leader就可以了。

Leader选完之后，follower并且开始连接到这个选出的leader。不保证follower可以连接到这个选出的leader。例如，这个选出的leader可能crash了。一旦它连接了，follower和leader同步(SYNC)状态，只有在同步状态之后，follower才可以开始处理新的请求。因为在选举leader时，选择zxid最大的那个server,所以leader上的transactions是最全，最新的。当处于两个epoch的过渡期时，为了优化过程，Zookeeper使用两个不同的方式来更新followers。如果follower不是太落后于leader，leader简单的发送丢失的transactions。他们都是最近的transactions，因为followers以严格的顺序接收所有的transactions。这个更新被称为`DIFF`。如果follower落后太多，Zookeeper使用一个full snapshot 传输，称为`SNAP`。

## Zab:Broadcasting State Updates
如果follower接收到读请求，那么它将在本地执行而不用转发给leader。follower读取状态，返回给客户端。因为在本地处理读请求，Zookeeper在处理读请求时很快。我们可以增加更多的服务器来增加整体的吞吐能力。

一旦follower接收到一个写请求，follower将请求转发给leader。leader执行这个请求，然后将执行结果以transaction的形式广播，作为状态更新。

一个问题是，一个server如何决定一个transaction已经commit了。这引出了一个协议称为Zab:`the Zookeeper Atomic Broadcast Protocol`。假设有一个活跃的leader，并且它有a quorum of followers支持他的领导权，commit一个transaction是很简单的，类似与一个二阶段提交协议：

1. leader发送一个PROPOSAL 消息，我们称为p,到所有的followers.

2. 一旦接收到p,follower回复一个ACK给leader，通知它已经接受了这个提议。

3. 一旦接收到a quorum of followers的ACK,leader 发送一个消息(COMMIT)通知follower commit这个transaction。

在回复ACK之前，follower需要执行一些检查。follower需要检查这个提议确实是来自当前正在following的leader，并且以leader广播的顺序，ACK和commit transaction。

Zab保证两个重要的特性：

+ If the leader broadcasts T and Tʹ in that order, each server must commit T before committing Tʹ .

+ If any server commits transactions T and Tʹ in that order, all other servers must also commit T before Tʹ .

第一个特性保证transactions在多个服务器间以同样的顺序传递，而第二个特性保证服务器不会跳过transactions。考虑到transaction是state update，并且每一个state update取决于前一个state update,跳过transaction可能导致不一致。两阶段提交保证了transactions的顺序。Zab在a quorum of servers上记录一个transaction。a quorum 必须在leader提交之前,确认(ACK)一个transaction，并且a follower在磁盘上记录它已经ACK了这个transaction。

在Zookeeper的运行过程过程中，leader可能crash，或者disconnected。所以需要选出行的leader，保证系统可用。`epoch`表示领导权随着时间改变。一个`epoch`表示一个特定server执行领导权的时期。epoch number在每一个新的leader被选举出来时增加。同一个server可能是不同的epoch的leader，但是对于协议来说，一个server在不同的epoch执行领导权被当做不同的leader。

Zookeeper只能保证当leader crash时，已经commit了的transaction才能被之后新选出的leader看见。如果在leader crash时，一个PROPOSAL被大多数follower ACK了,那么这个PROPOSAL也能被新选出的leader看到，如果只被少数follower ACK了，那么它可能不会被新选出的leader看到(取决于发起leader选举的节点是否ACK了这个PROPOSAL)。

> 如果一个PROPOSAL被follower ACK了，那么它会被持久化到Storage。

实现一个广播协议最困难的部分和同时出现的leader有关，不仅仅限于split-brain的场景。多个同时出现的leader可能使得server无序的commit transactions或者跳过transactions，使得Server处于不一致的状态。为了get around这个问题，Zab保证：

+ 一个选举出来的leader已经commit了所有的transactions,并且在开始广播新的transactions之前，会commit来自前一个epoch的transactions。

+ 两个servers任何时候都不可能有a quorum of supporters。就是说任何时候都只有一个server有超过半数的支持者。

为了实现第一个要求，leader在它确保a quorum of servers agrees on the state it should start with for the new epoch之前都是不可用的（因为state是由transactions改变的，所以state就代表了已经commit的transactions，所以这句话的意思是确保a quorum of servers和leader已经commit的transactions一样）。一个epoch的初始状态必须包含他之前已经提交过得所有的transactions，可能包括之前已经接受的(ACK)但是还没提交的(Commit)。在leader开始提出任何新的提议之前（比如是epoch e），它已经提交了epoch e-1 commit的所有的提议。如果有一个来自e1 的提议没有被leader在e(e1 < e)提出第一个新的提议之前提交，那么旧的提议永远不会被提交。

通过上面这两个保证可以get around concurrent leader的问题。

## Observers
Observers和followers在一些方面有相同点。特别的，他们commit 来自leader的提议。不像followers，Observers不参与投票过程。他们只是简单的通过`INFORM`消息学习已经commit的提议。followers和Observers都被称为learners因为leader告诉他们关于状态的改变。

参与投票，决定哪个提议committed的servers被称为`PARTICIPANT` servers。一个`PARTICIPANT`Server要么是leader，要么是follower。Observers，相反的，被称为`OBSERVER` servers。

有observers的一个主要原因是扩展读请求。通过添加更多的observers，我们可以服务更多的读请求而不用牺牲写请求的吞吐量。注意，写的吞吐量由quorum 大小决定。如果我们添加更多的可以投票的servers，我们可以等到更大的quorums,这会减小写的吞吐。然而，添加observers，不是完全没有成本的；每一个新的observer在每次commit transition时引入一个额外的消息成本。这个成本比添加可以投票的server的成本小。

observers的另一个原因是跨多个数据中心部署。将participants跨数据中心部署可能会显著的让系统变慢，因为数据中心的连接的延迟(the latency of links connecting data centers)。有了observers，更新请求可以以高吞吐和低延迟的方式在一个数据中心执行，并且传送到其他数据中心所以位于其他地方的客户端可以消费他们。注意，使用observers并没有消除跨数据中心的网络消息，因为observers需要转发更新请求到leader和处理`INFORM`消息。It instead enables the messages necessary to commit updates to be exchanged in a single data center when all participants are set to run in the same data center.

## Local Storage
我们已经提到了transaction log和snapshots,已经`SyncRequestProcessor`是处理写提议的processor。

servers使用transaction log来持久化transactions。在accept一个提议之前，server(follower 或 leader)持久化提议中的transaction到transaction log，transaction log是一个本地磁盘上的文件，transaction顺序追加（append）到这个文件。Every now and then, the server rolls over the log by closing the current file and creating a new one.

持久化transaction到磁盘有一个很重要的警告。现代操作系统通常cache dirty pages，并且异步将它们写入磁盘。然而，我们需要保证在我们继续之前，transaction已经持久化了。结果，我们需要`flush`transaction到磁盘。flushing 仅仅意味着我们告诉操作系统将dirty pages写入磁盘并且在操作完成时返回。因为我们在`SyncRequestProcessor`里持久化transaction，这个processor负责flush。当到了flush一个transaction到磁盘时，我们实际上为所有的处于队列中的transaction实现了group commit优化。如果有一个transaction 处于队列中，processor仍然执行flush。processor不会等待更多的排队的transactions，因为这样可能会增加执行延迟。For a code reference, check SyncRequestProcessor.run().

`Padding`是预分配磁盘块给文件。这样，对于块分配的文件系统元数据(metadata)的更新不会显著的影响文件的顺序写。如果transaction以很高的速度被追加到log，并且如果块没有预分配给文件，当它写到文件的最后一个块时，文件系统需要分配一个新的块。这将引入至少两个磁盘寻道：一个是更新metadata,另一个是返回文件。

为了避免和其他的写干涉，我们强烈推荐你写transaction log到一个独立的设备。第二个设备可以用于操作系统文件和snapshots。

## Snapshots
Snapshot是Zookeeper data tree的拷贝。每个server经常抓取data tree的快照，通过序列化整个data tree并且将它写入一个文件。servers 不需要协调的抓取快照，也不需要停止处理requests。因为servers在抓取快照时，仍然执行requests,当快照抓取好时，data tree改变了。我们称这样的快照为`fuzzy`，因为他们不需要反映某个特别时刻Data tree的真实状态。

我们看一个例子。假设一个data tree 只有两个znode：`/z`和`/z1`。最初，`/z`和`/z1`的数据都是1。现在考虑如下顺序的步骤：

1. 开始抓取快照

2. Serialize 并且写入`/z = 1`到快照

3. 设置`/z`的数据为2（transaction T）

4. 设置`/z1`的数据为2（transaction T1）

5. Serialize 并且写入`/z1 =2 `到快照

这个快照包含`/z =1,/z1 =2`。但是在时间上没有任何一个点，两个znode的值是这样的。然而，这不是一个问题，因为server replay transactions。它以快照开始时已经提交的最后一个transaction（称它为TS）为每一个快照打标签(比如lastZxid)。如果server最终load 这个快照，他重放(replay)TS之后的所有的transactions。在上述情况下，有T和T1。在replay T和T1之后，server获得`/z =2,/z1=2`，它是一个有效的状态。

An important follow-up question to ask is whether there is any problem with applying Tʹ again because it had already been applied by the time the snapshot was taken. As we noted earlier, transactions are idempotent, so as long as we apply the same transactions in the same order, we will get the same result even if some of them have already been applied to the snapshot.

## Servers and Sessions
