Leslie Lamport的[Paxos Made Simple](http://lamport.azurewebsites.net/pubs/paxos-simple.pdf)的中文翻译。

## 1.简介
用于实现一个分布式容错系统的Paxos算法，对很多读者来说，被认为难以理解，可能是因为原来的表述是希腊式的。实际上，它是最简单，很明显的分布式算法。它的核心是一个一致性算法-教会会议算法。下一节展示了这个一致性算法满足了我们想让他满足的绝大部分性质。最后一部分解释了完整的Paxos算法，which is obtained by the straightforward application of consensus to the state machine approach for building a distributed system-an approach that should be well-known,since it is the subject of what is probably the most often-cited article on the theory of distributed systems.

## 2.一致性算法
### 2.1 The Problem
假设一组进程可以提出 values。一个一致性算法保证被提议的值中只有一个被选中。如果没有被提出，那么没有值被选中。如果一个值已经被选中，那么这些进程可以learn这个被选中的值。一致性的安全要求是：

+ 只有被提出（propose）的值才能被选中
+ 只有一个值被选中，并且
+ 一个进程不会知道一个值被选中，除非他真的被选中。

我们不想指定精确的活性（liveness）要求。然而，目标是保证一些提议的值最终被选中，并且，如果一个值已经被选中，那么，一个进程可以最终learn这个值。

我们让一致性算法中的三种角色用三类代理表示：proposers（提议者）,acceptors（接收者），learners（学习者）。在一个实现中，单个进程可能身兼数职。

假设，代理可以通过发送消息来彼此通信。我们使用惯常的异步的，非拜占庭模型，在这个模型中：

+ 代理的速度是任意的，可能stop而失败，也可能重启。因为所有的代理可能在一个值被选中后失败，然后重启，除非失败并且重启的代理可以记住一些信息，否则没有解决方案。

+ 消息可能花费任意长的时间发送，可能重复，可能丢失，但是不能被破坏。

### 2.2 Choosing a Value
选中一个值的最简单的方式是只有一个acceptor 代理。一个提议者(proposer)发送一个提议(proposal)到这个acceptor,acceptor选中它接收的第一个提议值。虽然简单，这个解决方案是不令人满意的，因为acceptor的失败将导致任何接下来的流程都是不可能的。

所以，我们尝试其他的方式来选中一个值。不是一个acceptor,我们使用多个acceptor。一个proposer 发送一个proposed 的值到一组acceptors。一个acceptor可能接受这个提议的值。这个值被选中，当足够多的acceptors接受了它。怎么才算足够多呢？为了保证只有一个值被选中，我们认为只有包括了agent中的大多数才算足够多。因为，任何两个的大多数至少有一个acceptor是相同的，这是有效的如果一个acceptor只能接受最多一个value。

没有失败或者消息丢失，我们想一个值被选中，即使只有一个值被一个proposer提议。这表明了如下的要求：

**P1** 一个acceptor必须接受它接收的第一个提议。

但是这个要求引发了一个问题。几个值可能同时被不同的proposer提议，导致这种情况：每一个acceptor已经接受了一个值，但是没有哪个值被其中的大多数接受。即使只有两个提议的值， 如果每一个被acceptors中的一半接受，一个acceptor失败了，将使得不能知道哪一个值被选中。

P1 和 要求一个值只有在被大多数acceptors接受才被选中 意味着一个acceptor必须允许接受不止一个提议。我们通过分配一个整数给每一个提议来记住一个acceptor可能接受的不同的提议，所以一个提议由一个number和一个值(value)组成。为了防止混淆，我们要求不同的提议有一个number。如何做到取决于实现，目前我们只是假设这样。当一个提议被大多数acceptors接受时，这个提议的值被选中。在这种情况下，我们说，这个提议（和它的值）被选中了。

我们可以允许多个提议被选中，但是我们必须保证所有的选中的提议有相同的值。在提议的number上，用归纳法，它满足如下保证：

**P2** 如果一个提议（值为v）被选中，那么每一个被选中的number更大的提议的值为v

因为number总体是 **有序的**，条件 P2 保证了至关重要的安全性-只有一个值被选中。

为了被选中，一个提议必须至少被一个acceptor接受。那么，我们可以满足如下条件从而满足 P2:

**P2a** 如果一个提议（值为v）被选中,那么每一个被任一acceptor接受的number更大的提议的值为v。

我们仍然维持 P1 来保证一些提议被选中。因为交流是异步的，一个提议可以被选中，但是某些特定的acceptor c 可能没有接受到任何提议。假设一个新的提议者 “醒来”，发起一个number更大，但是值不同的提议。P1 要求 c 接受这个提议，但是这样违背了 P2a。同时维持P1 和 P2a 要求加强 P2a 到：

**P2b** 如果一个提议（值为v)被选中，那么每一个被任一提议者(proposer)发起的number 更大的提议的值为v。

因为一个提议必须被提议者发起才能被acceptor接受，P2b意味着P2a,反过来意味着P2。

为了发现如何满足P2b,我们考虑如何证明它。我们假设一些提议（值为v，number为m）被选中，那么任何发起的提议如果number n > m，它的值也为v。我们可以很容易证明，通过在n上使用归纳法，所以我们可以证明number 为n的提议的值为v, 所以我们可以假设每一个发起的number 为m到n-1 的提议的值也为v。因为number为m的提议被选中，所以必须有一个集合C 包含了大多数的acceptor,每一个在集合C中的acceptor接受它（提议）。结合这个和之前的归纳假设，假设number为m的提议被选中，意味着：

> 集合C中的每一个acceptor都接受了一个number在m到n-1中的提议，并且每一个number 在m到n-1的被任一acceptor接受的提议的值都为v

因为任何包含大多数acceptor的集合S 都至少包含了 集合C的一个元素，我们可以得出结论，一个number为n的提议的值为v,只需要保证下面的不变式满足：

**P2c** 对于任何的v 和 n,如果一个值为v,number为n的提议被发起，那么存在一个包含大多数acceptor的集合S，要么

+ (a)集合S中没有任何一个acceptor接受了任一number小于n的提议，或者
+ (b)v是被S中的acceptors接受的number最大，但是number小于n的提议的值

因此，我们可以通过维持不变式P2c来满足P2b.

为了维持不变式P2c,一个提议者想发起一个number为n的提议，必须学习（learn）到number最大，但是number小于n的提议，如果有的话，它已经或者将会被acceptors中的大多数接受。学习到已经被接受的提议是很容易的；预测未来的接受是很难的。不去预测未来，相反的，提议者设法保证这样的一个提议不会被接受。用其他的话来说，这个提议者要求acceptors不接受任何number小于n的提议。这引申出发起提议的如下算法：

1. 一个提议者选中一个number为n的新提议，发送request到某个acceptors集合中的每一个成员，要求他做出如下答复：

    (a)不接受number比n小的提议，并且

    (b)如果他已经接受了任何的提议，那么他们的最大的number要比n小

 我将这样的一个request称为一个 number 为n的prepare request

2. 如果这个提议者从大多数acceptors接收到request的答复（就是上面的1），那么它可以发起一个number为n,值为v的提议，v是所有答复中number最大的提议的值，或者是这个提议者选中的任意值（如果接收的答复中没有提议）.

一个提议者发起一个提议，通过发送给某个集合的acceptors一个request--这个提议要被接受（这个集合不必和最初request的那个集合是同一个）。我们称这为一个 accept request。

以上描述了一个提议者的算法。那么acceptor呢？它可以从提议者接受两种request： prepare request和accept request。一个acceptor可以忽略任何request而不会减少安全性。所以我们需要说，只有当它被允许答复一个request。它可以总是答复prepare request. 它也可以答复一个accept request,接受提议。用另一句话来说：

**P1a** 一个acceptor可以接收一个number为n的提议，如果它没有答复number比n大的 prepare request.

我们注意到，P1a可以归入P1.

我们现在有一个完整的算法来选中一个值满足要求的安全性--假设唯一的提议number。最终的算法可以通过做出一点优化来获得。

假设一个acceptor接收到了一个number为n的 prepare request,但是它已经答复了一个number比n大的prepare request,因此承诺不再接收任何number为n的新提议。因此这个acceptor没有理由答复这个新的prepare request,所以它将不会接受那个提议者发起的number为n的提议。所以我们让这个acceptor忽略了这个prepare request。我们也让它忽略了它已经接受的一个提议的prepare request.

使用这个优化，一个acceptor只需要记住他曾经接受的number最大的提议，以及它已经答复的number最大的prepare request。因为P2c必须无论失败与否保持不变，一个acceptor必须记住这些信息，即使当它失败然后重启了。记住，提议者总是可以丢弃一个提议，然后忘记它--只要它不在发起一个number一样的提议。

将acceptor 和 proposer的行为放在一起，我们可以看到这个算法可以分为如下两步：

**阶段1.**

+ (a) 一个提议者选中一个number为n的提议，然后发送一个number 为n 的prepare request到acceptors的大多数。

+ (b) 如果一个acceptor接收到一个number为n的prepare request，number n大于它已经答复的任何prepare request,那么它答复这个prepare request并且承诺不再接受任何number小于n的提议，n是它已经接受的number最大的提议（如果有的话）.

**阶段2.**

+ (a) 如果这个提议者接收到大多数acceptors对它的prepare request(number为n)的答复，那么它为number为n,值为v的提议发送一个accept request到这些acceptors中的每一个，其中v是答复中number最大的提议的值，或者是任何值（如果答复中没有任何的提议）

+ (b) 如果一个acceptor接收到一个number为n的提议的accept request，它将接受这个提议，除非它已经答复了一个number大于n的prepare request。

一个提议者可以发起多个提议，只要他的每一个提议都符合这个算法。它可以在这个协议的任何时候丢弃一个提议。如果一些提议者已经开始尝试发起一个number更大的提议，丢弃一个提议（number更小）是一个好主意。因此，如果一个acceptor因为它已经接受了一个number更大的prepare request而忽略了一个prepare 或accept request，那么它应该通知提议者，提议者然后可以丢弃这个提议。这是一个优化，不影响正确性。

### 2.3 Learning a Chosen Value
为了learn到已经选中的值，一个learner必须知道已经被大多数acceptors接受的提议。一个简单的算法是，让每一个acceptor,无论它什么时候接受到一个提议，答复给所有的learners,将提议发送给他们。这允许learners可以尽快的知道一个选中的值，但是它要求每一个acceptor答复每一个learner--答复的数量等于acceptors的数量乘以learners的数量。

假设没有拜占庭错误使得一个learner很容易的从另一个learner找出已经接受的值。我们可以让acceptors答复他们的接受值到不同的learner,当一个值已经选中时，这个learner反过来通知其他的learner。这种方法要求一个额外的round来使所有的learners发现选中的值。这可能不太可靠，因为learner可能失败。但是他要求恢复的数量只等于acceptors和learners的数量的和。

更普遍的是，acceptors可以回复他们的接受值到某个集合的不同的learner，当一个值被选中时这个集合中的每一learner然后可以通知所有的learner。使用一个大集合的不同的learners提供了更高的可靠性，但是耗费更大的沟通。

因为消息丢失，一个值可能被选中了但是没有learner知道。learner可以询问acceptors他们接受的提议，但是一个acceptor的失败可能使得知道大多数acceptors是否接受了一个提议变得不可能。在这种情况下，learners只有在一个新提议被选中时才知道什么值被选中了。如果一个learner需要知道一个值是否被选中，它可以让一个提议者发起一个提议，使用上面描述的算法。

### 2.4 Progress
构建一个两个提议者一致发起一系列的number增加的提议，但是没有任何一个被选中的场景是很容易的。提议者P 使用一个number 为n1的提议完成步骤1。另一个提议者Q 然后使用一个number n2(n2 > n1)的提议完成步骤1.提议者P 的步骤2 accept request被忽略因为接受者已经承诺不再接受number小于n2的任何新的提议。所以，提议者P 然后开始一个新的number n3（n3 > n2）的提议,完成步骤1,导致提议者Q 的步骤2被忽略。

为了保证进展，一个不同的提议者必须被选中出来作为唯一的发起提议。如果这个不同的提议者可以和大多数acceptors沟通成功，并且它使用了一个number比之前大的提议，那么它将成功的发起一个可以接受的提议。如果它知道有number更大的request，它可以丢弃提议重新尝试，最终，这个不同的提议者可以选中一个number足够大的提议。

如果系统中的足够部分（proposer,acceptors,网络）工作正常，liveness可以通过选中一个不同的提议者来实现。The famous result of Fischer, Lynch, and Pat- terson [1] implies that a reliable algorithm for electing a proposer must use either randomness or real time—for example, by using timeouts. However, safety is ensured regardless of the success or failure of the election.

## 2.5 The Implementation
The Paxos algorithm [5] assumes a network of processes.在它的一致性算法里，每一个进程扮演proposer,acceptor和learner的角色。算法选中一个leader，它扮演这个不同的提议者和不同的learner的角色。Paxos 一致性算法正如上面描述的，它的request和response作为普通信息发送。回复(response)信息以对应的提议number标记来阻止混淆。稳定存储，在失败时保存，被用于维护acceptor必须记住的信息。一个acceptor在真正发送回复之前将它将要回复的信息记录在稳定存储中。

剩下的就是描述确保没有两个提议以相同的number被提出。不同的提议者从number的不相交集中选中number，所以两个不同的提议者永远不会发起相同number的提议。每一个提议者记录（在稳定存储中）它已经发起的最大的number,以比已经使用的number更大的number发起过程1.

## 3. Implementing a State Machine
实现一个分布式系统的一种简单方式是一系列客户端发起命令道一个中央服务器。这个服务器可以形容为一个确定性的状态机，它以某个顺序执行客户端的命令。这个状态机有一个当前状态；它的每一步，以一个命令作为输入，产生一个输出和一个新的状态。举例来说，分布式银行系统的客户端可能是自动取款机，状态机的状态可能由所有用户的账户余额组成。一次取款可能通过执行一个减少账户余额的状态机命令来执行，当且仅当余额大于取款金额，输出旧的和新的余额。

使用单个中央服务器的实现，如果这个服务器失败了，这个实现也失败了。因此我们使用一系列的服务器，每一个独立实现状态机。因为状态机是确定性的，如果他们以同样的顺序执行命令，所有的服务器将产生同样顺序的状态和输出。一个客户端发起一个命令然后使用任一服务器产生的输出。

为了保证所有的服务器以同样的顺序执行状态机命令，我们实现一系列独立的Paxos一致性算法实例，被第i个实例选中的值作为序列中第i个状态机命令。每一个服务器扮演所有的角色(proposer,acceptor,learner)in each instance of the algorithm.目前我假设Servers数目是固定的，所以一致性算法的所有实例使用同样集合的代理。

通常来说，一个server被选举为leader，作为一致性算法所有实例中不同的提议者（唯一一个会发起提议的）。客户端发送命令到leader，leader决定序列中的每一个命令应该出现在哪儿。如果leader决定某个客户端命令应该是第135个命令，它会试着让这个命令被选中为第135个的值。这通常都会成功。这可能会因为failure而失败，或者因为另一个server也相信自己是leader,认为第135个命令有不同的值。但是一致性算法保证最多有一个命令被选中为第135个。

这个方法效率高的关键是，在Paxos一致性算法中，直到步骤2提议的值才被选中。回想一下，在完成提议者算法的步骤1之后，要么被提议的值还没有确定，要么提议者可以提议任何值。

我现在将描述Paxos状态机实现在通常步骤是如何工作的。晚一点，我会讨论如何发生错误。我考虑前一个leader刚好失败，一个新的leader已经选举的时候（系统启动时是一种特殊的情况，这时候没有任何的命令被提议）。

这个新的leader，在一致性算法的所有实例中，成为一个learner，应该知道已经被选中的绝大多数命令。假设，它知道命令1-134,138和139--它们是一致性算法实例中1-134,138,139的值。（回顾一下，每个命令是一个提议的值）。然后它执行实例135-137的步骤1，并且所有实例都大于139。（我在下面描述了这是如何做到的）。假设这些执行步骤的结果决定实例135和140被提议的值，但是使得其他实例被提议的值不受约束(unconstrained)。leader然后为实例135和140执行步骤2,因此选中命令135和140.

这个leader，和其他已经学习到这个leader已经知道的所有命令的服务器一样，现在可以执行命令1-135。然而，它不可以执行命令138-140（他也知道命令138-140），因为命令136和137还没有被选中。这个leader本可以让客户端请求的接下来的两个命令为命令136和137.相反，我们通过提议一个使状态保持不变的“无操作”命令，正如命令136和137来立即填满空隙。（通过执行实例136和137的步骤2）。一旦这些无操作指令被选中，命令138-140可以被执行。

命令1-140现在都被选中了。leader已经为所有大于140的实例完成了步骤1，并且它可以在步骤2提议任意值。它将命令141分配为客户端请求的下一个命令，提议它作为实例141步骤2的值。它提议它接收到的下一个客户端命令为命令142，以此类推。

leader可以在知道他提议的命令141已经被选中之前提议命令142.有可能它发送的提议命令141的消息都丢失了，也有可能在其他服务器知道leader提议的命令141之前，命令142已经被选中了。当leader无法接受到实例141在步骤2中的回复时，它会重新发送这些消息。如果一切顺利，它提议的命令将会被选中。然而，它可能第一个失败，在选中的命令的序列中留下空隙。通常来说，假设leader可以预先获得 a 个命令--就是说，命令1到i被选中后，它可以提议命令 i+1 到i+a。A gap of up to α−1 commands could then arise.

一个新选择的leader为一致性算法的很多的实例执行步骤1--在上面的例子中，就是135-137和所有大于139的实例。通过发送一个合理的短的信息到其他的服务器，可以为所有的实例使用同样的number。在步骤1中，只有当它已经从某个提议者接收到一个步骤2消息，这个acceptor才可以回复一个more than a simple OK.（在这个场景中，这种情况只适合实例135和140）。因此，一个服务器（作为acceptor）可以回复一个简单的合理的短的信息给所有的实例。Executing these infinitely many instances of phase 1 therefore poses no problem.

因为leader的failure和选举一个新的leader应该是不常有的事件，执行一个状态机命令的效率花费是在命令（提议值）上达成一致---只是执行步骤2的花费。可以看出，Paxos一致性算法的步骤2具有在存在故障时达到一致的任何算法的最小可能成本。因此，Paxos算法本质上是最优的。

系统的普通操作的讨论假设总是只有一个leader，除了当前leader failure和新的leader选举的这一小段时期。在反常的状况下，leader选举可能fail。如果没有server作为leader，那么没有新的命令被提议。如果多个servers认为它们是leaders,那么他们可能对同一实例提议不同的值，这可能会阻止任何的值被选中。然而，安全性得以保持--两个不同的Servers将永远不会就第i个状态机命令的值达成一致。选举一个leader只需要确保进展。

如果服务器集合改变了，那么必须有某种方式检测哪些服务器用了哪些一致性算法。最简单的方式是通过状态机本身来做。当前的服务器集可以成为状态的一部分，并且可以使用普通状态机命令来改变。We can allow a leader to get α commands ahead by letting the set of servers that execute instance i + α of the con- sensus algorithm be specified by the state after execution of the ith state machine command. This permits a simple implementation of an arbitrarily sophisticated reconfiguration algorithm.
