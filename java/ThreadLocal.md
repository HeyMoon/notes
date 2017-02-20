ThreadLocal类提供了一种线程隔离的方案。Threadlocal类提供了thread-local的变量。这些变量和普通的变量不同之处在于每一个线程只可以通过`set`或`get`方法访问他们（线程）自己的变量。ThreadLocal实例一般是类中的`private static`field。

那么ThreadLocal的实现原理是什么呢？结合ThreadLocal的源码，我们看看。

我们先看看ThreadLocal的一般用法：

````
public class ThreadLocalTest {

    private static final AtomicInteger nextId = new AtomicInteger(0);

    private static final ThreadLocal<Integer> threadId = new ThreadLocal<Integer>(){
        @Override
        protected Integer initialValue() {
            return nextId.getAndIncrement();
        }
    };

    public static int get() {
          return threadId.get();
    }

}
````
