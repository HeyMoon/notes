Raft和Multi-Paxos的主要区别是

1. Raft中的log entry是线性Append的，而Multi-Paxos可以并发的。

2. Raft中的强leader模式。Multi-Paxos中leader仅仅作为一种性能优化模式，而Raft中leader是一致性算法的一部分。并且在Raft中只有有最新，最全的log的节点才能成为leader，Multi-Paxos则没有这个限制。
