ConcurrentHashMap提供了完全并发的检索和高并发的更新。这个类遵守和`HashTable`一样的方法规范，对于`HashTable`中的每个方法都提供了对应的版本。但是虽然所有的操作都是线程安全的，检索（获取）操作不需要锁定（索取锁），也没有任何支持锁定整个table从而阻止其他访问的方法。

获取操作（get）通常不需要阻塞，所以可能和更新操作（包括put和remove）有重叠的地方。获取反映了最近完成的更新操作的结果，从他们hold开始。更加正式的是，对于一个给定key的更新操作和任何非null的获取操作服从happens-before关系，获取时会反映更新了的值。对于聚合操作，比如`putAll`和`clear`，并发获取可能反映只有少数entries的insertion或removal。相似的，iterators，spliterators和enmuerations返回的元素反映了hash table在某个时刻的状态，或者自从iterator/enumeration创建时的状态。他们不会抛出`java.util.ConcurrentModificationException`.然而，iterators被设计为在同一时间只能被一个线程使用。牢记这一点，总体状态方法包括`size`和`isEmpty`和`containsValue`的结果，通常只当一个Map没有被其他线程并发更新时有用。否则这些方法的结果反映的短暂的状态可能只适用于监控或者估计的目的，而不能用于程序控制。

当有太多的碰撞时（例如，keys有不同的hash code但是却落在同一个槽里），table会自动扩容，以预期平均效果维持大概两个bins一个映射（对应于用于调整大小的0.75的负载因子阈值）.当mapping添加或移除时，这种平均依然有很多的差异，但是整体上，这维持了一个通常可以接收的时间/空间折衷。然而，调整ConcurrentHashMap或其他种类的哈希表的大小，可能是一个相对慢的操作。当可能时，提供一个估计的大小作为`initialCapacity`可选的构造参数.一个另外的可选的`loadFactor`构造参数提供了自定义初始table容量的更加有用的参数，通过指定表的用于计算可分配的数量的密度。

Overview:

ConcurrentHashMap设计的主要目的是维持并发的读性能(主要是get()方法，但也包括iterators和相关方法)，最小化update竞争。第二个目的是，比`java.util.HashMap`有相同的或更好的空间消耗，也支持许多线程在空的table上的高的初始插入率。

这个map一般作为桶（bin）的hash table.每一个key-value mapping被保存在一个Node中。大多数nodes是基本的Node类的实例，有hash,key,value,和next fields。然而，有许多子类：TreeNode,被安排在平衡树，而不是lists.TreeBins 持有TreeNodes的根节点的集合。在调整大小时，ForwardingNodes 被放在桶的头部。ReservationNodes 在computeIfAbsent和相关方法确定值时被用作placeholds。TreeBin,ForwardingNode和ReservationNode并不持有正常的用户keys,values,或者hashs，在搜索时可以轻而易举的区别，因为他们有负的hash值，null的key和value。（z这些特殊的nodes要么是不寻常的要么是transient的，所以有这些没有使用的field是没有影响的）


直到第一次插入时，table被延迟初始化为2的N次方的大小。table中的每一个bin(桶)通常包含一列Nodes。table的访问需要volatile/atomic read，writes,和cas。

我们使用Node hash域的高位（有符号）来控制--这是可行的因为寻址限制。有负的hash 域的Nodes会被特殊处理，或者忽略。

在一个空桶的第一个node插入(通过put或者他的变体)是通过在这个桶(bin)上执行cas操作完成。这是目前为止最普遍的put操作under most key/hash distributions。其他的更新操作（insert，delete，和replace）需要获取锁。我们不想浪费空间来讲每一个桶(bin)和每一个不同的锁(lock)对象关联起来，所以使用每个bin list中的第一个node作为一个锁。在这些lock上的锁定依赖于內建的`synchronized`监视器。

使用一个list中的第一个node作为锁对象并不是很足够的：当一个node被锁定了，任何的update必须首先验证在锁定之后，仍然是第一个node被锁定，如果不是的话，就要重试。因为新的node总是被追加到list，一旦一个node是bin中的第一个，他总是第一个，直到被删除或者这个bin变得无效（一旦调整大小）。

以每一bin作为锁定单位的主要的缺点是，在这个bin list里的被同一个锁保护的其他node的更新操作可能暂缓。例如当用户的`equals`或mapping 方法花费很长时间。然而，从统计上来说，以随机的hash code,这不是一个普遍的问题。理想情况下，bin中的node的频率服从平均参数为0.5的泊松分布，假设调整大小阈值为0.75，尽管由于调整粒度，方差仍然较大。忽略方差，以k为大小的list，期望出现的概率是 (exp(-0.5) * pow(0.5, k) / factorial(k))。第一个值是：

````
 0:    0.60653066
 1:    0.30326533
 2:    0.07581633
 3:    0.01263606
 4:    0.00157952
 5:    0.00015795
 6:    0.00001316
 7:    0.00000094
 8:    0.00000006
 more : 小于千万分之一
````

在random hash 下，两个线程访问不同元素发生锁竞争的可能性大概为 1 / (8 * #elements)

在实践中遇到的实际哈希码分布有时会偏离均匀随机性。这包括了当 N > (1<<30)的情况，所以一些key必须抵触。 类似的，对于多个key被设计为有相同hash code或者仅在被屏蔽的高位上不同的愚蠢的或有敌意的用法。所以，当bin中node的数量超过一个阈值时，我们使用第二种策略。这些TreeBins使用一个平衡树（红黑树的一种具体形式）来保存node,边界搜索时间为O（log n）。在TreeBin中的每一个搜索步骤至少比普通list慢两倍，但是考虑到N cannot exceed (1<<64) (before running out of addresses) this bounds search steps, lock hold times,等等。TreeBin nodes（TreeNode）也和普通node一样维护了相同的下一个横向指针，所以可以以同样的方式在iterator中横向移动。

当占用率超过一个概率阈值（通常为0.75）时，table调整大小。

延迟初始化到第一次使用，也可以避免当第一次操作是从一个putAll,有map参数的构造器，或者反序列化时的调整大小。这些情况下，会试着覆盖initialCapacity设置，但是在竞争的情况下，会无害的失败。

 The element count is maintained using a specialization of
 LongAdder. We need to incorporate a specialization rather than
 just use a LongAdder in order to access implicit
 contention-sensing that leads to creation of multiple
 CounterCells.  The counter mechanics avoid contention on
 updates but can encounter cache thrashing if read too
 frequently during concurrent access. To avoid reading so often,
 resizing under contention is attempted only upon adding to a
 bin already holding two or more nodes. Under uniform hash
 distributions, the probability of this occurring at threshold
 is around 13%, meaning that only about 1 in 8 puts check
 threshold (and after resizing, many fewer do so).

TreeBins使用一种特殊形式的比较来搜索以及相关的操作（这也是为什么我们不能用已经存在的集合例如TreeMap）。TreeBin 包含了可以比较的元素，但是也可能包含其他的，比如可以比较但是对于同一的T不需要比较的元素，所以我们不能在他们上执行`compareTo`方法。为了处理这种情况，Tree首先使用hash 值排序，如果可行的话再使用`Comparable.compareTo`排序。在节点上查找时，如果元素不能比较或者比较等于0，那么左边和右边的子节点可能都需要以绑定的hash值查询。一旦插入，为了在调整平衡时保持整体的顺序，we compare classes and identityHashCodes as tie-breakers.The red-black balancing code is updated from pre-jdk-collections(http://gee.cs.oswego.edu/dl/classes/collections/RBCell.java)

TreeBins 也需要一个额外的锁定机制。尽管list的横向移动即使在update时也可以读取，tree 的横向移动就不可以了，只要是因为树旋转可能改变根节点和（或）他的连接。TreeBins在主要的bin锁定策略上寄生了一个简单的read-write lock 机制：与插入和移除相关的结构调整已经在bin上锁定了（所以不能和其他的写入者冲突），但是必须等待进行中的读取者完成。因为可能只有一个这样的等待者，我们使用一个简单的结构，使用单个`waiter` field来阻塞写入者。但是读取者需要永不阻塞。如果root 锁被持有了，他们沿着缓慢的横向路径前进（通过next-pointer）直到锁变得可用，或者list穷尽了，无论哪个先发生。这种情况不会很快，但是最大化总预期吞吐量。

维护API和与之前版本的一致性引出了几个古怪的事。主要是：我们去掉了没有使用的构造参数`concurrencyLevel`。我们接收一个`loadFactor`作为构造参数，但是只是在初始化table容量时使用它。我们也声明了一个不用的`Segment`类，只有当序列化时以最小的形式初始化。
