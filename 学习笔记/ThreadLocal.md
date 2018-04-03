当使用ThreadLocal维护变量时，ThreadLocal为每个使用该变量的线程提供独立的变量副本，所以每一个线程都可以独立地改变自己的副本，而不会影响其它线程所对应的副本。

我们看看ThreadLocal的实现。

每一个Thread都有一个`ThreadLocal.ThreadLocalMap threadLocals`实例变量，这个map存放该Thread所有的ThreadLocal变量和维护的变量。

````
static class ThreadLocalMap {

    /**
     * The entries in this hash map extend WeakReference, using
     * its main ref field as the key (which is always a
     * ThreadLocal object).  Note that null keys (i.e. entry.get()
     * == null) mean that the key is no longer referenced, so the
     * entry can be expunged from table.  Such entries are referred to
     * as "stale entries" in the code that follows.
     */
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }

    /**
     * The initial capacity -- MUST be a power of two.
     */
    private static final int INITIAL_CAPACITY = 16;

    /**
     * The table, resized as necessary.
     * table.length MUST always be a power of two.
     */
    private Entry[] table;

    /**
     * The number of entries in the table.
     */
    private int size = 0;

    /**
     * The next size value at which to resize.
     */
    private int threshold; // Default to 0

    /**
     * Set the resize threshold to maintain at worst a 2/3 load factor.
     */
    private void setThreshold(int len) {
        threshold = len * 2 / 3;
    }

    /**
     * Construct a new map initially containing (firstKey, firstValue).
     * ThreadLocalMaps are constructed lazily, so we only create
     * one when we have at least one entry to put in it.
     */
    ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
        table = new Entry[INITIAL_CAPACITY];
        int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
        table[i] = new Entry(firstKey, firstValue);
        size = 1;
        setThreshold(INITIAL_CAPACITY);
    }
````

ThreadLocalMap实际上是一个`Entry`数组，Entry里存放ThreadLocal对象和它维护的变量。我们知道一个线程可以有多个ThreadLocal对象，那么如何确定每一个ThreadLocal对象位于`Entry[]`中的位置呢？`int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);`这样就确定了ThreadLocal对象在`Entry[]`中的位置。

所以ThreadLocal对象本身并不存储值，所有的值都是存储在`Entry[]`中，ThreadLocal对象只是提供一个threadLocalHashCode来确定在Entry中的位置。

### ThreadLocal内存泄漏
最根本的原因是有线程池 才会内存泄露 如果线程用完即销毁 是不存在内存泄露的这个问题
