Thread.sleep和Object.wait的区别：
Thread.sleep不会放弃监控器的所有权，Object.wait会放弃对象的监控器。


Hashtable是线程安全的，Hashtable通过在方法上加上`synchronized`关键字获得线程安全性。

Vector也是。

但是Hashtable和Vector上的复合操作不是线程安全的。

容器的toString,hashcode,equals方法会间接的执行迭代操作。

## ConcurrentHashMap:
ConcurrentHashMap使用粒度更细的加锁机制来实现更大程度的共享，这种机制称为分段锁。在这种机制中，任意数量的读取线程可以并发的访问Map,执行读取操作的线程和执行写入操作的线程可以并发的访问map，并且一定数量的写入线程可以并发的修改Map。ConcurrentHashMap带来的好处是在并发访问是可以是线程更高的吞吐量，而在单线程环境中只损失非常小的性能。


volatile：在synchronized和volatile提供的可见性保证中可能会使用一些特殊的指令，即内存栅栏。内存栅栏可以刷新缓存，是缓存无效，刷新硬件的写缓冲，以及停止执行管道。在内存栅栏中，大多数操作都是不能重排序的。

synchronized的局限性：

1. 无法中断一个正在等待获取锁的线程

2. 无法实现非阻塞的加锁规则

Lock

1. 可轮询，可中断，可以非阻塞，可设置获取锁等待超时时间

2. 公平性

只有在内置锁无法满足需求的情况下，ReentrantLock可以作为一种高级工具。当需要一些高级功能时才应该使用ReentrantLock，这些功能包括：可定时的，可轮询的与可中断的锁获取操作，公平队列，以及非块结构的锁。否则还是优先使用synchronized。


````
acquire lock on object state
while (precondition does not hold){
  release lock
  wait until precondition hold
  optional fail if interrupted or timeout expire
  reacquire lock
}
  preform action
  release Lock
````

条件等待的标准形式：

````
synchronized(lock){
  while(!conditionPredicate){
    lock.wait();
  }
  //现在对象处于合适的状态
   preform action
}
````

调用条件队列的方法时(wait,notify,notifyAll)必须获取条件队列对象的锁。

优先使用notifyAll而不是notify,因为条件队列上可能有多个条件谓词。

AQS使用一个双向链表来表示等待的线程集（Node）。 CLH lock


“重入”意味着获取锁的操作的粒度是“线程”而不是调用。重入的一种实现方法是，为每个锁关联一个获取计数值和一个所有者线程。

synchronized关键字的实现里，锁是存在对象头里的，对象头的"Mark word”里会存储获取了锁的的线程id.
