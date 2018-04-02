NIO 提供了多路I/O，NIO.2 提供了异步I/O，它让客户端代码开始一个I/O操作，随后当I/O操作完成时通知客户端。

## Asynchronous I/O Overview
`java.nio.channels.AsynchronousChannel`接口声明了一个异步channel，它支持异步操作（read，write）。通过调用一个方法开始一个I/O操作，然后返回一个future或需要一个completion handler 参数。

+ `Future<V> operation(...)`:调用操作然后返回一个`java.util.concurrent.Future<V>`接口对象，V 是操作的结果类型。`Future`方法可能被调用来检查I/O操作是否完成，等待操作完成，获取操作结果。
+ `void operation(... A attachment, CompletionHandler<V,? super A>handler)`:调用操作，以attachment和handler为参数。

`CompletionHandler`声明下列方法来处理操作成功的结果，以及操作失败时采取合适的操作。

+ `void completed(V result, A attachment)`:当操作成功时调用。操作的结果被定义为`result`，开始时附属的对象定义为`attachment`.
+ `void failed(Throwable t, A attachment)`:当操作失败时调用。

调用之后，这个方法立即返回。然后你调用`Future`或者在`CompletionHandler`中实现对操作结果的处理。

`Future`声明了`boolean cancel(boolean mayInterruptIfRunning)`方法来取消执行。这个方法导致所有等待这个I/O操作结果的线程抛出`java.util.concurrent.CancellationException`。底层I/O 是否可以取消很大程度上取决于实现，因此没有具体说明。如果取消使channel或它连接的实体处于不一致的状态，那么channel处于error状态以阻止开始I/O操作的进一步尝试。

如果调用`cancel()`时设置`mayInterruptIfRunning`为true，那么这个I/O操作可能因为关闭channel而中断。在这种情况下，所有等待这个I/O操作结果的线程抛出`CancellationException`，channel上的其他未完成的I/O操作以抛出`java.nio.channels.AsynchronousCloseException`而结束。

当执行`cancel()`以取消read或write，建议I/O操作中使用的所有的buffer都丢弃。

`AsynchronousChannel`继承`java.nio.channels.Channel`接口，继承了它的`isOpen()`和`close()`方法。`close()`方法适用于下面的情况：在这个channel上任何未完成的异步操作都会抛出`AsynchronousCloseException`而完成。

> Asynchronous Channels可以在多个并发线程中安全使用。一些channel实现可能支持并发读和写。

`java.nio.channels.AsynchronousByteChannel`接口继承`AsynchronousChannel`，他提供了下面四个方法：

+ `Future<Integer> read(ByteBuffer dst)`:从这个channel中读取一系列bytes到ByteBuffer中。返回一个Future，当可用时以访问这些bytes。
+ `<A> void read(ByteBuffer dst, A attachment,CompletionHandler<Integer,? super A> handler)`：从这个channel中读取一系列bytes到ByteBuffer中，在CompletionHandler中访问这些bytes。
+ `Future<Integer> write(ByteBuffer src)`：从ByteBuffer中写入一系列bytes到这个channel。返回一个Future以访问写入数。
+ `<A> void write(ByteBuffer src, A attachment,CompletionHandler<Integer,? super A> handler)`：从

The read() methods throw `java.nio.channels.ReadPendingException` when the channel doesn’t allow more than one read to be outstanding and a previous read has not completed. The write() methods throw `java.nio. channels.WritePendingException` when the channel doesn’t allow more than one write to be outstanding and a previous write has not completed.

> `java.nio.ByteBuffer`对象在多线程并发线程中是不安全的。当开始了一个read或write，要保证在操作完成之前，buffer不能被访问。

### Asynchronous File Channels

抽象的`java.nio.channels.AsynchronousFileChannel`类描述了一个异步channel，read，write，和操作文件。这个channel可以这样打开:

`AsynchronousFileChannel ch;
ch = AsynchronousFileChannel.open(Paths.get("somefile"));`

通过调用`AsynchronousFileChannel`的`read()`和`write`方法可以读和写文件。

一个异步文件channel没有文件的当前position。然而，文件position作为参数传递给每一个`read`和`write()`方法，开始异步操作。

> `read()`和`write()`方法必须提供一个文件中的绝对的position（相对于零），从这个position开始读写。文件有一个关联的position并且读和写相对于这个position没有任何意义。因为读和写可能在前一个操作完成之前就开始了，并且他们发生的顺序不能保证。因为同样的原因，`AsynchronousFileChannel`中没有方法设置和查询file position。

为了支持read 和 write，`AsynchronousFileChannel`定义了如下操作：

+ 对文件的更新可以强制刷入底层存储设备，保证数据不会因为系统宕机而丢失。调用`force(boolean metaData)`来完成这个任务。
+ 文件中的一个区域可能锁住防止其他程序访问。调用`lock()`和`trylock()`完成这个任务。

下面代码展示了使用一个`AsynchronousFileChannel`打开一个文件，读取前1024个字节到Future.

````
public static void main(String[] args) throws IOException, ExecutionException, InterruptedException {
    Path path = Paths.get(".").resolve(Paths.get("src/main/java/com/dyh/tutorial/nio2/AFCDemo.java"));
    AsynchronousFileChannel afc = AsynchronousFileChannel.open(path);
    ByteBuffer buffer = ByteBuffer.allocate(1024);
    Future result = afc.read(buffer,0);
    while (!result.isDone())
    {
        System.out.println("Sleeping...");
        Thread.sleep(500);
    }
    System.out.println("Finished = " + result.isDone());
    System.out.println("Bytes read = " + result.get());
    afc.close();
}
````

下面的代码展示了用`AsynchronousFileChannel`打开一个文件，读取前1024字节到CompletionHandler。

````
public static void completionHandler() throws IOException {
        Path path = Paths.get(".","src/main/java/com/dyh/tutorial/nio2/AFCDemo.java");
        AsynchronousFileChannel afc = AsynchronousFileChannel.open(path);
        ByteBuffer buffer = ByteBuffer.allocate(2089);
        final Thread mainThread = Thread.currentThread();
        afc.read(buffer, 0, null, new CompletionHandler<Integer, Object>() {
            public void completed(Integer result, Object attachment) {
                System.out.println("Bytes read = " + result);
                mainThread.interrupt();
            }

            public void failed(Throwable exc, Object attachment) {
                System.out.println(exc);
                mainThread.interrupt();
            }
        });

        //waiting for completion
        try {
            mainThread.join();
        } catch (InterruptedException e) {
            mainThread.interrupt();
            e.printStackTrace();
        }

        afc.close();

    }
````

无论操作成功或失败，每个方法都调用main线程的`interrupt()`方法。这样做因为，main线程调用`join()`方法，这将使得main线程无限期阻塞。线程只有被抛出的`InterruptedException`唤醒。

### Asynchronous Socket Channels
`AsynchronousServerSocketChannel`描述了一个异步channel以监听面向流的socket。
`AsynchronousSocketChannel`。

#### AsynchronousServerSocketChannel
为了获取一个`AsynchronousServerSocketChannel`对象，执行`AsynchronousServerSocketChannel`的`open()`方法：

`AsynchronousServerSocketChannel ch = AsynchronousServerSocketChannel.open();`

根据`AsynchronousServerSocketChannel`的文档，这个方法返回一个和默认群组（default group）绑定的Asynchronous server socket channel。另一个`AsynchronousServerSocketChannel open(AsynchronousChannelGroup group)`
返回一个和指定group 绑定的Asynchronous server socket channel。

你可以通过执行`<T> AsynchronousServerSocketChannel setOption(SocketOption<T> name, T value)`配置。支持的options有：SO_RCVBUF 和 SO_REUSEADDR.

一个新创建的，可能配置了的Asynchronous server socket channel已经打开，但是还没有绑定到一个本地地址。通过执行`AsynchronousServerSocketChannel`的`bind()`可以绑定到本地地址，监听连接。一旦绑定，调用任何一个`accept()`方法开始接受连接。

> Asynchronous server socket channel在多线程中是安全的。如果一个线程在前一个accept操作结束之前开始一个accept操作，`java.nio.channels.AcceptPendingException`抛出。


如下代码说明了`AsynchronousServerSocketChannel`。

````
public class Server {

    private final static int PORT = 9090;
    private final static String HOST = "localhost";


    public static void main(String[] args) {
        AsynchronousServerSocketChannel channelServer;
        try {
            channelServer = AsynchronousServerSocketChannel.open();
            channelServer.bind(new InetSocketAddress(HOST, PORT));
            System.out.printf("Server listening at %s%n",
                    channelServer.getLocalAddress());
        } catch (IOException ioe) {
            System.err.println("Unable to open or bind server socket channel");
            return;
        }
        Attachment att = new Attachment();
        att.channelServer = channelServer;
        channelServer.accept(att, new ConnectionHandler());
        try {
            Thread.currentThread().join();
        } catch (InterruptedException ie) {
            System.out.println("Server terminating");
        }
    }

}
````

上述代码需要注意的是，main线程通过调用`join()`阻塞。唯一的使线程不阻塞的方法是从另一个线程interruptmain线程。

Attachment 类如下图所示：

````
public class Attachment {
    public AsynchronousServerSocketChannel channelServer;
    public AsynchronousSocketChannel channelClient;
    public boolean isReadMode;
    public ByteBuffer buffer;
    public SocketAddress clientAddr;
}
````

为了不写get和set，我直接把他们声明为public。

`channelServer` 字段存储Server类main方法创建的server socket channel的引用。它被ConnectionHandler用于调用`AsynchronousServerSocketChannel`的`accept()`方法。

`channelClient`字段存储传递到ConnectionHandler的`completed()`方法响应`accept()`方法的socket channel引用。它也用于执行开始的`read()`调用，并且执行`ReadWriteHandler`上的read()/write()。

`isReadMode`字段表明是read还是write操作，在调用`ReadWriteHandler`的`completed()`方法之前。

`buffer`字段定义在Connection Handler中创建的byte buffer并且用于server和client交互。每一个client 都有它自己的byte buffer。

最后，`clientAddr`字段存储远程客户端的`jav.net.SocketAddress`。它存储客户端的socket address并且作为客户端的特殊message而输出。

我们用`ConnectionHandler`类来管理来自客户端的Connection。

````
public class ConnectionHandler implements CompletionHandler<AsynchronousSocketChannel, Attachment>{

    /**
     * Invoked when an operation has completed.
     *
     */
    public void completed(AsynchronousSocketChannel channelClient, Attachment att) {
        try
        {
            SocketAddress address = channelClient.getRemoteAddress();
            System.out.printf("Accepted connection from %s%n",address);
            att.channelServer.accept(att,this);
            Attachment newAtt = new Attachment();
            newAtt.clientAddr = address;
            newAtt.channelClient = channelClient;
            newAtt.channelServer = att.channelServer;
            newAtt.isReadMode = true;
            newAtt.buffer = ByteBuffer.allocate(2048);
            ReadWriteHandler rwh = new ReadWriteHandler();
            channelClient.read(newAtt.buffer, newAtt, rwh);

        }
        catch (IOException ioe)
        {
            ioe.printStackTrace();
        }
    }

    /**
     * Invoked when an operation fails.
     *
     * @param exc        The exception to indicate why the I/O operation failed
     * @param attachment
     */
    public void failed(Throwable exc, Attachment attachment) {
        System.out.println("Failed to accept connection");
    }
}
````

`ConnectionHandler`是一个`CompletionHandler`，它响应到来的连接。它的`completed()`方法在一个连接成功时被调用。否则，它的`failed()`方法被调用。

`completed()`方法的第一个任务是获取和输出client的socket address以识别。我指定`channelClient.getRemoteAddress()`而不是`channelClient.getLocalAddress()`来返回client的socket address因为`channelClient`的对应的socket是在服务端创建，它和客户端的socket交互。调用`channelClient`的`getLocalAddress()`返回服务端的SocketAddress。调用`channelClient`的`getRemoteAddress()`返回客户端的SocketAddress。

接下来，`completed()`方法调用server socket channel的`accept()`方法，以传递的`Attachment`对象和当前completion handler的引用为参数。这个调用允许服务端响应接下来到来的connection.

调用传递给`completed()`方法的`Attachment`对象中的server socket channel引用的`accept()`方法。这个对象也作为参数传给`accept()`方法。这个`Attachment`对象是用来识别server socket channel。

在调用`accept()`之后，第二个`Attachment`对象被创建和初始化以准备从client读取。次此外，一个`ReadWriteHandler`对象被创建以响应读操作。新的`Attachment`对象的byte buffer,新的`Attachment`对象，和`ReadWriteHandler`对象作为参数传递给client socket channel的`read()`方法。

`ReadWriteHandler`用来管理客户端的读和写：

````
public class ReadWriteHandler implements CompletionHandler<Integer,Attachment>{

    private final static Charset CSUTF8 = Charset.forName("UTF-8");

    /**
     * Invoked when an operation has completed.
     *
     * @param result     The result of the I/O operation.
     * @param attachment
     */
    public void completed(Integer result, Attachment attachment) {
        if (result == -1){
            try {
                attachment.channelClient.close();
                System.out.printf("Stopped listening to client %s%n",
                        attachment.clientAddr);
            } catch (IOException e) {
                e.printStackTrace();
            }
            return;
        }

        if (attachment.isReadMode){
            attachment.buffer.flip();
            int limit = attachment.buffer.limit();
            byte[] bytes = new byte[limit];
            attachment.buffer.get(bytes,0,limit);
            System.out.printf("Client at %s sends message: %s%n",
                    attachment.clientAddr,
                    new String(bytes, CSUTF8));

            attachment.isReadMode = false;
            attachment.buffer.rewind();
            attachment.channelClient.write(attachment.buffer,attachment,this);

        }else {
            attachment.isReadMode = true;
            attachment.buffer.clear();
            attachment.channelClient.read(attachment.buffer,attachment,this);
        }

    }

    /**
     * Invoked when an operation fails.
     *
     * @param exc        The exception to indicate why the I/O operation failed
     * @param attachment
     */
    public void failed(Throwable exc, Attachment attachment) {
        System.out.println("Connection with client broken");
    }
}
````

`ReadWriteHandler`是一个CompletionHandler，它响应`read()`和`write()`方法。它的`completed()`方法被调用来响应一个成功的`read()`或`write()`，包括发起一个对应的`write`和`read()`。它的`failed()`方法被调用当客户端断开和服务端的连接。

`completed()`方法的第一个任务是响应一个返回`-1`的调用，它表示没有bytes可以读，因为end-of-stream。它关闭client socket channel,输出一个信息，return。


#### AsynchronousSocketChannel
可以调用`AsynchronousSocketChannel`对象的`open()`方法获取`AsynchronousSocketChannel`对象：

`AsynchronousSocketChannel ch = AsynchronousSocketChannel.open();`

根据`AsynchronousSocketChannel`的文档，这个方法返回一个异步socket channel，绑定到default group。

可以通过执行`<T> AsynchronousSocketChannel setOption(SocketOption<T> name, T value)`方法配置，支持的配置有：`SO_RCVBUF, SO_SNDBUF, SO_KEEPALIVE, SO_REUSEADDR, and TCP_NODELAY`.

新创建和可能配置的异步socket channel已经打开但是还没有连接。

新创建的socket channel 通过调用`connect()`方法连接。一个`connect()`方法返回Future,另一个需要CompletionHandler参数。

一旦连接，socket channel保持连接直到关闭。一个socket chan是否连接也许可以通过调用它的`SocketAddress getRemoteAddress()`确定，返回远程socket的地址（null，如果没有连接）。

````
public class Client {

    private final static Charset CSUTF8 = Charset.forName("UTF-8");
    private final static int PORT = 9090;
    private final static String HOST = "localhost";

    public static void main(String[] args) {
        try {
            AsynchronousSocketChannel socketChannel = AsynchronousSocketChannel.open();
            socketChannel.connect(new InetSocketAddress(HOST, PORT)).get();

            Attachment att = new Attachment();
            att.channel = socketChannel;
            att.isReadMode = false;
            att.buffer = ByteBuffer.allocate(2048);
            att.mainThd = Thread.currentThread();
            byte[] data = "Hello".getBytes(CSUTF8);
            att.buffer.put(data);
            att.buffer.flip();
            socketChannel.write(att.buffer, att, new ReadWriteHandler());
            try {
                att.mainThd.join();
            } catch (InterruptedException ie) {
                System.out.println("Client terminating");
            }

        } catch (IOException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }

}
````

````
public class Attachment {
    public AsynchronousSocketChannel channel;
    public boolean isReadMode;
    public ByteBuffer buffer;
    public Thread mainThd;
}
````

````
public class ReadWriteHandler implements CompletionHandler<Integer,
        Attachment> {


    private final static Charset CSUTF8 = Charset.forName("UTF-8");
    private BufferedReader conReader =
            new BufferedReader(new InputStreamReader(System.in));


    public void completed(Integer result, Attachment att)
    {
        if (att.isReadMode)
        {
            att.buffer.flip();
            int limit = att.buffer.limit();
            byte[] bytes = new byte[limit];
            att.buffer.get(bytes, 0, limit);
            String msg = new String(bytes, CSUTF8);
            System.out.printf("Server responded: %s%n", msg);
            try {
                msg = "";
                while (msg.length() == 0)
                {
                    System.out.print("Enter message (\"end\" to quit): ");
                    msg = conReader.readLine();
                }
                if (msg.equalsIgnoreCase("end"))
                {
                    att.mainThd.interrupt();
                    return; }
            }
            catch (IOException ioe)
            {
                System.err.println("Unable to read from console");
            }
            att.isReadMode = false;
            att.buffer.clear();
            byte[] data = msg.getBytes(CSUTF8);
            att.buffer.put(data);
            att.buffer.flip();
            att.channel.write(att.buffer, att, this);
        } else {
            att.isReadMode = true;
            att.buffer.clear();
            att.channel.read(att.buffer, att, this);
        }
    }


    public void failed(Throwable t, Attachment att)
    {
        System.err.println("Server not responding");
        System.exit(1);
    }
}
````

### Asynchronous Channel Groups
抽象的`java.nio.channels.AsynchronousChannelGroup`描述了一组异步channel以资源共享为目的。一个`group`有关联的线程池处理提交的任务，处理I/O事件和分发异步操作结果到completion handler。

> 关联的线程池属于group，group终止导致关联的线程池关闭。

`AsynchronousSocketChannel`和`AsynchronousServerSocketChannel`属于group。当通过无参的`open()`创建`AsynchronousSocketChannel`或`AsynchronousServerSocketChannel`时，channel被绑定到default group,它是自动创建的系统内channel group，由JVM维护。default group 有一个关联的thread pool,根据需要创建新的线程。你可以配置default group通过初始化下列的系统属性（System properties）在JVM启动时。

+ `java.nio.channels.DefaultThreadPool.threadFactory`:这个属性的值是一个具体的`java.util.concurrent.ThreadFactory`类。这个类是使用system classloader加载和初始化的。这个工厂的`newThread(Runnable r)`方法在线程池创建每个线程时执行。如果加载和初始化这个属性值失败，在构造default group的过程中会抛出一个异常。如果default group的`ThreadFactory`没有配置，那么default group的池内的线程都是守护线程。

+ `java.nio.channels.DefaultThreadPool.initialSize`：`initialSize`的值指定了default group的线程池的初始大小。如果一个字符串不能解析为整数，在构造default group的过程中会抛出一个异常。

你也许像定义自己的channel group因为它让你对线程池的线程有更多的控制。更重要的是，它提供了关闭线程和等待终止的机制。你可以创建你自己的Asynchronous channel group通过调用下面方法中的一个：

+ `AsynchronousChannelGroupwithCachedThreadPool (ExecutorService executor, int initialSize)`:使用给定的线程池（java.util.concurrent.ExecutorService）创建，它根据需要创建新线程。
+ `AsynchronousChannelGroup withFixedThreadPool(int nThreads, ThreadFactory threadFactory)`:创建一个有固定线程池的Asynchronous channel group
+ `AsynchronousChannelGroup withThreadPool (ExecutorService executor)`:使用给定线程池创建.

下列代码创建一个新的channel group，它有固定的20个线程。
`AsynchronousChannelGroup group = AsynchronousChannelGroup. withFixedThreadPool(20,Executors.defaultThreadFactory());`

在创建一个group之后，你可以绑定一个Asynchronous server socket channel到这个group,通过调用有参数的`open()`方法。

>在写completion handler时，避免会阻塞线程的操作很重要。当所有的线程都阻塞了，整个应用可能阻塞。对于一个自定义或cached thread pool，队列可能会增长的很大，最终导致out-of-memory。

`void shutdown()`方法开始有序的shutdown 一个group。一旦shutdown，当所有的异步channel关闭，所有活跃的执行completion handler已经完成，group使用的resource都被释放，group也终止。正在执行的completion handler不会被停止或中断。

````
// Initiate an I/O operation that isn't satisfied.
channel.accept(null, completionHandler);
// After the operation has begun, the channel group is used to control
// the shutdown
if (!group.isShutdown())
{
   // After the group is shut down, no more channels can be bound to it.
    group.shutdown();
}
if (!group.isTerminated())
{
    // Forcibly shut down the group. The channel is closed and the
    // accept operation aborts.
    group.shutdownNow();
}
// The group should be able to terminate; wait for 10 seconds maximum.
group.awaitTermination(10, TimeUnit.SECONDS);
````

#### What About AsynchronousFileChannel?
`AsynchronousFileChannel`不属于groups.但是他们也有关联的线程池来接收提交的任务，处理I/O事件，分发I/O操作结果到completion handler。

When an asynchronous file channel is created without specifying a thread pool, the channel is associated with a system-dependent default thread pool that may be shared with other channels. The default thread pool is configured by the system properties defined by `AsynchronousChannelGroup`.

An asynchronous file channel created by `AsynchronousFileChannel`’s A`synchronousFileChannel open(Path file, OpenOption... options)` class method is associated with the default thread pool. You can associate a file channel with another thread pool by calling the `AsynchronousFileChannel open(Path file, Set<? extends OpenOption> options, ExecutorService executor, FileAttribute<?>... attrs)` class method. Here, executor identifies the desired thread pool.
