ThreadLocal类提供了一种线程隔离的方案。Threadlocal类提供了thread-local的变量。这些变量和普通的变量不同之处在于每一个线程只可以通过`set`或`get`方法访问他们（线程）自己的变量。ThreadLocal实例一般是类中的`private static`field。

那么ThreadLocal的实现原理是什么呢？结合ThreadLocal的源码，我们看看。

每一个Thread都有一个`ThreadLocal.ThreadLocalMap threadLocals` field，这个map由`ThreadLocal`类维护。

ThreadLocalMap内部有一个Entry[]数组，这个Entry[]数组是最终存线程隔离变量的地方。

那么ThreadLocalMap如何与ThreadLocal类对应起来呢？或者说，每一个ThreadLocal类的变量如何确定在Entry[]中的index呢？

原来每一个ThreadLocal类有一个threadLocalHashCode属性。

The ThreadLocal objects act as keys, searched via threadLocalHashCode.  This is a custom hash code (useful only within ThreadLocalMaps) that eliminates collisions in the common case where consecutively constructed ThreadLocals are used by the same threads, while remaining well-behaved in less common cases.

ThreadLocal对象作为ThreadLocalMap的key，通过threadLocalHashCode进行查找。它是一个自定义的hash code,只用于ThreadLocalMap,消除了大部分情况下的碰撞。

总结一下：
每一个线程都有一个ThreadLocalMap变量，这个ThreadLocalMap以ThreadLocal对象为key。那么如何确定ThreadLocal对象在ThreadLocalMap中的位置呢？通过ThreadLocal对象的threadLocalHashCode属性。`int i = key.threadLocalHashCode & (table.length - 1);` 这里的table是Entry[]数组。
