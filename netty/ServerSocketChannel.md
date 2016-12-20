SelectableChannel的这个方法：

````
SelectionKey register(Selector sel, int ops)
````

如果`ops`等于0。0也是一个合法的值，这表示你注册一个Channel，但是没有对任何的操作感兴趣。所以当你调用`select()`方法时，没有任何的操作会被selected,这可能导致`select()`方法永远阻塞，取决于你调用的`select()`方法。

参考：

https://bugs.openjdk.java.net/browse/JDK-6443887
