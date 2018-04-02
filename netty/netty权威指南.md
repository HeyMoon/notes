java 同步I/O 的弊端：

    /**
     * Reads some number of bytes from the input stream and stores them into
     * the buffer array <code>b</code>. The number of bytes actually read is
     * returned as an integer.  This method blocks until input data is
     * available, end of file is detected, or an exception is thrown.
     *
     * <p> If the length of <code>b</code> is zero, then no bytes are read and
     * <code>0</code> is returned; otherwise, there is an attempt to read at
     * least one byte. If no byte is available because the stream is at the
     * end of the file, the value <code>-1</code> is returned; otherwise, at
     * least one byte is read and stored into <code>b</code>.
     *
     * <p> The first byte read is stored into element <code>b[0]</code>, the
     * next one into <code>b[1]</code>, and so on. The number of bytes read is,
     * at most, equal to the length of <code>b</code>. Let <i>k</i> be the
     * number of bytes actually read; these bytes will be stored in elements
     * <code>b[0]</code> through <code>b[</code><i>k</i><code>-1]</code>,
     * leaving elements <code>b[</code><i>k</i><code>]</code> through
     * <code>b[b.length-1]</code> unaffected.
     *
     * <p> The <code>read(b)</code> method for class <code>InputStream</code>
     * has the same effect as: <pre><code> read(b, 0, b.length) </code></pre>
     *
     * @param      b   the buffer into which the data is read.
     * @return     the total number of bytes read into the buffer, or
     *             <code>-1</code> if there is no more data because the end of
     *             the stream has been reached.
     * @exception  IOException  If the first byte cannot be read for any reason
     * other than the end of the file, if the input stream has been closed, or
     * if some other I/O error occurs.
     * @exception  NullPointerException  if <code>b</code> is <code>null</code>.
     * @see        java.io.InputStream#read(byte[], int, int)
     */
    public int read(byte b[]) throws IOException {
        return read(b, 0, b.length);
    }

注意这句话：
    This method blocks until input data is
    available, end of file is detected, or an exception is thrown.

当对Socket的输入流进行读取操作时，它会一直阻塞下去，直到发生如下三种事件。
+ 有数据可读；
+ 可用数据读取完毕
+ 发生空指针或者I/O异常

这意味着当对方发送请求或者应答消息比较缓慢、或者网络传输较慢时，读取输入流一方的通信线程将被长时间阻塞，
如果对方要60s才能够将数据发送完成，读取一方的I/O线程也会被同步阻塞60s，在此期间，其他接入的消息只能在消息队列中排队。

再来看看输出流。

    public void write(byte b[]) throws IOException
    Writes an array to bytes. This method will block until the bytes are actually written.

当调用OutputStream的write方法写输出流时，它将会阻塞，直到所有要发送的字节全部写入完毕，或者发生异常。
学习过TCP/IP相关知识的人都知道，当消息的接收方处理缓慢的时候，将不能及时地从TCP缓冲区读取数据，这将会导致发送方的TCP Window size不断减小，直到为0，双方处于keep-alive状态，消息发送发将不能再向TCP缓冲区写入消息，这时如果采用的是同步阻塞I/O，write操作将会被无限期阻塞，直到TCP window size大于0或者发生I/O异常。

读和写的操作都是同步阻塞的，阻塞的时间取决于对方I/O线程的处理速度和网络I/O的传输速度。

伪异步I/O实际上仅仅只是对之前的线程模型的一个简单优化，他无法从根本上解决同步I/O导致的通信线程阻塞问题。
我们简单分析下如果通信对方返回应答时间过长，会引起的级联故障。

1.  服务端处理缓慢，返回应答消息耗费60s，平时只需要10ms。
2.  采用伪异步I/O的线程正在读取故障服务节点的响应，由于读取输入流是阻塞的，因此，它将会被同步阻塞60s。
3.  假如所有可用线程都被故障服务阻塞，那后续所有的I/O消息都将在队列中排队。
4.  由于线程池采用阻塞队列实现，当队列积满之后，或许入队列的操作将被阻塞。
5.  由于前段是由一个accept线程接收客户端接入，它被阻塞在线程池的同步阻塞队列之后，新的客户端请求消息将被拒绝，客户端发生大量的连接超时。
6. 由于几乎所有的连接都超时，调用者会认为系统已经崩溃，无法接收新的请求消息。

NIO编程

## 1. 缓冲区Buffer
首先介绍缓冲区（Buffer）的概念，Buffer是一个对象，它包含一些要写入或者要读出的数据。在NIO类库中加入buffer对象，体现了NIO与原I/O的一个重要区别。在面向流的I/O中，可以将数据直接写入或者将数据直接读到Stream对象中。

在NIO库中，所有的操作都是用缓冲区处理的。在读取数据时，它是直接读到缓冲区中的；在写入数据时，写入到缓冲区中。任何时候访问NIO中的数据，都是通过缓冲区进行操作的。

缓冲区实质上是一个数组。通常它是一个字节数组（ByteBuffer）。也可以使用其他种类的数组。但是一个缓冲区不仅仅是一个数组，缓冲区提供了对数据的结构化访问以及维护读写位置等信息。
