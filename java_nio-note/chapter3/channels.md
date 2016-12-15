---
title: channels
date: 2016-09-28 16:48:07
categories:[技术]
tags:[java nio,channel]
---
Channel是`java.nio`的第二个主要革新。他们不是一个扩展或提升，而是一个新的，一流的Java I/O 范例。他们提供了与I/O服务的直接连接。`Channel`是在byte buffer和Channel另一端的实体（通常是文件或socket）之间有效运输数据的管道。

在多数情况下，channel和操作系统文件描述符或文件指针有一对一的关系。虽然channel比文件描述符更通用，多数channel的使用场景是连接到一个打开文件的描述符。channel类提供了必要的抽象来维护平台无关性，但仍然塑造了现代操作系统的native I/O能力。

channel是网关，通过channel，操作系统的native I/O服务可以以最小的开支访问。buffer是内部endpoint，被channel用来接收和发送数据。

<img src="./3-1.png"/>

正如你在图3-2中看到的，channel类的继承关系比buffer类要复杂一些。内部关系更加复杂，并且部分类依赖于`java.nio.channels.spi`中的类。

<img src="./3-2.png"/>

## 3.1 Channel Basics
我们首先看看`Channel`接口：

````
package java.nio.channels;






### 3.1.1 Opening Channels
Channel是I/O服务的管道。I/O主要有两大类：文件I/O和streamI/O。所以有两种channels：文件和socket。如果你参考图3-2，你会发现有一个`FileChannel`类和三种Socket channel类：SocketChannel，ServerSocketChannel和DatagramChannel。

Channel可以用几种方式创建。Socket Channel类有工厂方法直接创建新的socket channel。但是一个`FileChannel`对象只能在一个打开的`RandomAccessFile`,`FileInputStream`,`FileOutputStream`对象调用`getChannel()`获取。你不可以直接创建`FileChannel`对象。


````
SocketChannel sc = SocketChannel.open();














不像buffers，channel不可以重用。一个打开的channel代表和一个具体的I/O服务的连接，并且包含连接的状态。当一个channel关闭了，连接也丢失了，channel再也没有连接到任何东西。

调用一个Channels的close方法，可能会引起线程短暂的阻塞，当这个channel最后确定底层I/O服务的关闭，即使这个channel是非阻塞模式。当一个channel关闭时，阻塞行为取决于操作系统和文件系统。在一个通道上调用多次close方法是无害的，但是如果第一个线程已经阻塞在close方法，任何其他的线程调用close方法也会阻塞，直到第一个线程完成关闭channel。之后在一个已经关闭的channel上调用close不会做任何事，直接返回。

一个channel的打开状态可以用`isOpen()`测试。如果返回true，那么这个channel可以使用。如果返回false，那么这个channel再也不可以使用了。

Channels引入了一些和关闭和中断有关的新的行为。如果一个channel实现了`InterruptibleChannel`接口，那么它支持以下语义。如果一个线程阻塞在一个channel上，并且这个线程被中断了（被另一个线程调用这个阻塞线程的interrupt方法），那么这个channel将会关闭，并且这个阻塞线程将会抛出一个`ClosedByInterruptException`。

另外，如果一个线程的中断状态设置了，并且这个线程试图访问一个channel，那么这个channel将会立即被关闭，抛出`ClosedByInterruptException`。当线程的`interrupt()`方法被调用时，线程的中断状态被设置。线程当前的中断状态可以用`isInterrupted()`方法测试。当前线程的中断状态可以调用静态的`Thread.Interrupted()`方法清除。

Channels提供了一个重要的新的能力，scatter/gather。Scatter/gather是简单却强大的概念。它适用于对多个buffers执行一个I/O操作。对一个write操作，数据从几个buffers收集，然后发送到channel。buffers不需要有同样的capacity。就像所有的buffers中的数据在发送前串联到一个大的buffer中。对于read操作，从channel读取的数据**顺序**的分发到多个buffers，将每一个buffers填满（到limit），直到从channel读取的数据或所有的buffer空间耗尽。

现代操作系统支持native 向量I/O（vectored I/O）。当你在一个channel上请求scatter/gather操作，这个请求将会转化成合适的native 调用来直接填充或读取这个buffer。这是巨大的胜利，因为buffer copy 和系统调用减少或消除了。Scatter/gather应该和direct ByteBuffer一起使用，从nativeI/O获得最大的优势，特别是当这个buffer是long-lived。

````
public interface ScatteringByteChannel
}


int bytesRead = channel.read (buffers);
````

一旦从`read()`返回，bytesRead的值为48，header buffer包含10 bytes数据，body buffer包含剩下的38 bytes数据。channel自动的将数据顺序分发到这两个buffer。buffers已经填充了，需要flip使得他们准备读取。
图3-5说明了一个gather writer。数据从buffers数组中的每个buffer中收集，组装成bytes 流，然后发送到channel。

<img src="./3-5.png"/>

图3-6展示了一个scattering read。到达channel的数据分发给buffer list，顺序填充每个buffer，从position到limit。下面的position和limit是read操作之前的。

<img src="./3-6.png"/>

接收offset和length参数的`read()`和`write()`提供了一种方法可以利用buffers数组的子数组。这里的offset表示哪一个buffer开始使用。这个length参数表示使用的buffer的数量。例如，如果我们有一个五个元素的array叫fiveBuffers，并且已经初始化了五个buffers，那么以下的代码将会把第二，三，四个buffer中的数据写入。

````
int bytesRead = channel.write(fiveBytes,1,3);
````

Scatter/gather使用得当的话会非常强大。它允许你委托操作系统来完成辛苦的活：将读到的数据分开存储到多个存储桶或者将不同的数据区块合并成一个整体。这是一个巨大的成就，因为操作系统已经被高度优化来完成此类工作了。它节省了来回移动数据的工作，因此避免了buffer copy，减少了你需要书写的代码。

### 3.3 File Channels
之前我们都是讨论channels的通用之处。现在我们要讨论具体的了。这一节我们讨论file Channels。正如图3-7所示，`FileChannel`类可以实现常用的read和write操作，也可以scatter/gather。也提供了一些新的方法。

<img src="./3-7.png"/>

File channels总是阻塞的，不能置于非阻塞模式。现代操作系统有成熟的caching 和 prefetch 算法使得本地磁盘I/O低延迟。网络文件系统通常有高延迟但也会受益与同样的优化。面向流的非阻塞模式并不适合于面向文件的操作，因为文件I/O的本质不同。对于文件I/O，最好的方式是异步I/O，它允许一个进程向操作系统请求一或多个I/O操作，但是不用等待他们完成。在请求的I/O完成时，进程会得到通知。

正如之前说明的，`FileChannel`对象并不能直接创建。一个`FileChannel`实例只能通过一个打开的文件对象（RandomAccessFile，FileInputStream，FileOutputStream）的`getChannel()`获得。调用`getChannel()`方法，返回一个连接到相同文件的`FileChannel`对象，它和文件有相同的权限。

像大多数Channels一样，FileChannel会尽可能的使用nitive I/O。FileChannel类是抽象的，`getChannel()`对象返回的是子类。

FileChannel对象是线程安全的。多个线程可以并发的调用同一个实例上的方法，但是并不是所有的操作都是多线程的。影响channel的position或文件大小的方法是单线程的。如果其他线程已经执行一个影响channel position或文件大小的操作，那么任何其他尝试这些操作的线程将等待。

像大多数和I/O相关的类，`FileChannel`反映JVM外部具体对象的一个抽象。`FileChannel`类保证同一个JVM里的所有的实例对于一个给定的文件将会看到一致的视图。但是JVM并不能保证他不能控制的要素。通过`FileChannel`实例看到的文件的视图可能或可能不和外部的非java进程看到的文件视图一致。多进程的并发文件访问的语义，很大程度上依赖于操作系统或文件系统。一般来说，不同JVM上的`FileChannel`对象并发访问同一个文件，和非java进程并发访问是一致的。

### 3.3.1 Accessing Files
每一个`FileChannel`对象和文件描述符有一对一的关系。所以FileChannel的方法和符合POSIX操作系统上的常见文件I/O系统调用一致。名字可能不同，但是作用是一致的。你可能注意到了和RandomAccessFile的方法的相似之处。`RandomAccessFile`提供了大体上同样的抽象。在通道出现之前,底层的文件操作都是通过 RandomAccessFile 类的方法来实现的。FileChannel 模拟同样的 I/O 服务,因此它的API自然也是很相似的。

<img src="./table-3-1.png"/>

我们仔细看看基本的文件访问方法（记住这些方法中的每一个都会抛出`java.io.IOException`）

````
public abstract class FileChannel



当磁盘分配给一个文件的空间小于文件的大小时，文件洞就出现了。现代文件系统只在数据真正写入时分配磁盘空间。如果数据写入到文件是不连续的，那么这可能导致文件的区域逻辑上不包含任何数据（holes）。例如下面的代码将会产生文件洞。


````
public class FileHole
		File temp = File.createTempFile ("holy", null);
````

如果这个文件是顺序读取的，那么任何的file hole将会被0填充，但是并不占据磁盘空间。读取这个文件的进程将会看到5000021 bytes，其中的绝大多数是0。尝试运行`strings`命令在这个文件上看你会得到什么。

````
dengyunhuideMacBook-Pro:~ dengyunhui$ strings /var/folders/3n/d7rkc8x538j2pd8zydd9fyw40000gn/T/holy7521862031760369510.tmp
*<-- location 0
*<-- location 50000
*<-- location 5000000
````

试着将文件大小设置为50MB或100MB，看看你的磁盘空间消耗（应该不会变），扫描文件花费的时间（相应的改变）.

<img src="./3-8.png"/>

FileChannel position是底层文件描述符的反映，和FileChannel 引用获得的文件对象共享。这意味着一个对象（文件对象或FileChannel对象）改变position，另一个对象(文件对象或FileChannel对象)也会看见。

````
RandomAccessFile randomAccessFile = new RandomAccessFile ("filename", "r");
// Change the position using the RandomAccessFile object
````

和buffers的相关的`get()`和`put()`类似，当bytes 通过`read()`或`write()`传输时，file position自动更新。如果position达到了文件大小值，正如`size()`方法返回的，`read()`方法将会返回 end-of-file(-1)。但是，不像buffers，如果`write()`方法将position移动到大于文件大小值，那么文件会扩充来适应新的bytes 写入。

像buffers一样，channel有绝对形式的`read()`和`write()`，以position为参数。绝对版本返回时当前文件的position不会改变。绝对的reads和writes可能效率更高，因为channel的状态不需要更新；请求可以直接传递到native code。更好的是，多个线程可以并发的访问同一个文件而不会彼此干扰。因为每一个调用时原子的，并且不依赖与任何之前的状态。

如果绝对版本的read，传递的参数position大于文件的大小（如size方法所返回的），那么将会返回end-of-file(-1)。尝试绝对版本的`write()`，传递的参数position大于文件大小将会导致文件扩容来适应写入的新的bytes。在之前的end-of-file position和新增加的bytes位置之间的值，`FileChannel`并没有具体说明，但是大多数情况下取决于底层文件系统语义(大部分系统会以0填充，比如macos)。取决于操作系统和文件系统类型，这将导致 file hole。

当需要减少文件的大小，`truncate()`方法将砍掉你指定的size之外的任何数据。如果当前的size大于新的size，所有超过新的size的数据将会被丢弃。如果新的size大于或等于当前size，那么文件将不会更改。这两种情况下，`truncate()`方法的一个副作用是它会将file position设置为你传递的新的size。


````
public abstract class FileChannel
````

最后一个方法是force。这个方法告诉channel强制将所有的待定的修改持久化到磁盘。所有的现代操作系统cache data和推迟磁盘更新来提高性能。调用`force()`要求所有的对文件的待决的更改立即同步到磁盘。

如果文件存在于本地文件系统，那么一旦`force()`返回，它保证自从channel创建（或者最后一次调用force方法）以来，所有对文件的更改都写入磁盘。这对于关键性操作很重要，例如事务处理，确保数据完整和可靠的恢复。但是，如果文件存在于远程文件系统，如NFS，这种永久存储的同步保证并不能做出。对于其他文件系统可能同意正确，这依赖于实现。**如果你的系统面临系统崩溃时，必须维护数据完整性，核实操作系统和你所使用的文件系统。（如消息中间件，消息储存时要考虑此类问题）**

`force()`的布尔参数表明文件的元数据(meta data)在方法返回之前是否应该同步到磁盘。Metadata 代表诸如文件所有者，访问权限，最后一次修改时间等等。在大多数情况下，对于数据恢复，这些信息并不重要。传 false 给`force()`表明只有文件数据需要同步到磁盘，在返回之前。大多数情况下，同步metadata将至少需要操作系统一次额外的低层次I/O操作。一些大数量事务处理程序可能通过在每次调用 force()方法时不要求元数据更新来获取较高的性能提升,同时也不会牺牲数据完整性。

最后，这一部分可以和RandomAccessFile对比看。

### 3.3.2 File Locking
正如在第一章所讨论的，locks 可以共享或独占（互斥）。本部分讨论的file-locking 特性非常依赖于native 操作系统实现。 不是所有的操作系统和文件系统都支持共享文件锁。对于那些不支持的，请求共享锁将会静默的升级为请求独占锁。这保证了正确性但是可能会影响性能。确保你了解你所部署的操作系统和文件系统的file-locking 行为。

此外，不是所有的平台都用同样的方式实现了基本文件锁定。文件锁定语义可能因操作系统的不同而不同，也可能因同一操作系统的不同文件系统而不同。有些操作系统只提供建议锁，有些只提供独占锁，有些可能提供两者。最安全的方式是，你总是用建议锁管理文件。意识到底层操作系统是如何实现的一样很明智。例如，如果所有的锁都是强制的，如果你没有在规定的时间释放锁，你获得的锁可能影响同一系统上的其他应用。









}



如果锁定区间是有效的，但是它必须等待一个已经存在的锁释放，lock方法将阻塞。如果你的线程在这种情况下暂停了，它的中断语义和3.1.3节相似。如果这个channel被另一个线程关闭，那么这个暂停的线程会恢复并且收到`AsynchronousCloseException`。如果暂停线程被直接中断了（调用它的interrupt方法），那么它将唤醒并且收到`FileLockInterruptionException`。当执行`lock()`方法时，如果线程的中断状态已经设置，这个异常会立即被抛出。

在上面的API中，两个名字为`tryLock()`的方法是`lock()`的非阻塞版本。他们起到和lock一样的作用，但是当请求锁不能立即获取时返回null。

正如你看到的,`lock()`和`tryLock()`返回一个`FileLock`对象，下面是FileLock的完整API。

````
public abstract class FileLock
}






FileLock lock = fileChannel.lock(  )
新的`FileChannel`类提供了一个方法，`map()`，它可以建立一个打开的文件和一个特殊的ByteBuffer之间的虚拟内存映射（可以参考第一章）。在一个`FileChannel`上调用`map()`创建了一个由磁盘文件支持的虚拟内存映射并且将那块虚拟内存空间包装为一个`MappedByteBuffer`对象。

从`map()`方法返回的`MappedByteBuffer`对象在很多方面和基于内存的buffer类似，但是他的数据元素是存储在磁盘的文件上的。调用`get()`将从磁盘取数据，并且这些数据反映了文件的当前内容，即使在映射建立之后，文件已经被外部进程修改了。通过file mapping看到的文件内容和你常规的读取文件看到的内容是一致的。同样的，在一个mapped buffer上做`put()`操作，将会更新磁盘上的文件（假设你有写权限），并且你对文件的改变会被文件的其他读取者看到。

通过memory-mapping 机制访问一个文件比常规的read和write（即使是通过channel）效率高多了。不需要显式的系统调用，这可能是费时的。更重要的是，操作系统的虚拟内存系统会自动的cache 内存页（memory Pages）。这些页会使用系统内存cache，并不会消耗JVM的堆内存空间。

一旦一个内存页有效（从磁盘中获取到内存中），它可以以最大的硬件速度被再次获取而不用发起另外的系统调用来获取这些数据。大的，结构化的文件比如包含index或者其他的频繁引用或者更新的部分可以从memory mapping获益良多。**比如RocketMQ的消息持久化文件**。当结合文件锁来保护关键区域和控制事务的原子性，你可以看到memory mapped buffer的好处。

让我们看看如何使用memory mapping：

````
public abstract class FileChannel

正如你所看到的，只有一个`map()`方法可以建立一个文件映射。它接收一个mode，一个position和一个size为参数。这个position和size参数和`lock()`方法的含义一样。可以创建一个表示一个文件部分bytes的MappedByteBuffer。例如，为了映射299 bytes中的100 bytes，你可以这样做：

`buffer = fileChannel.map(FileChannel.MapMode.READ_ONLY,100,200);`。

要map整个文件，你需要这样做：

`buffer = fileChannel.map(FileChannel.MapMode.READ_ONLY,0,fileChannel.size());`。

不像文件锁的锁范围，mapped file范围不可以超过文件的真实大小。如果你要求一个映射大于文件大小，文件将会扩张来满足你所map的size。如果你传递`Integer.MAX_VALUE`给size参数，你的文件大小会剧增到大于2.1G。即使你请求一个read-only mapping，`map()`方法可能也会这样做，但是在大多数情况下，会抛出`IOException`因为底层的文件不可以被修改（因为你请求的权限是read-only）。这个行为和之前讨论的file hole的行为一致。

`FileChannel`类定义了表示mapping mode的常量和使用类型安全的枚举而不是数值来定义这些常量。这些常量是`FileChannel`内部定义的一个内部类(inner class)的静态字段,它们可以在 编译时被检查类型,不过您可以像使用一个数值型常量那样使用它们。



public static void mappedFile() throws IOException {
        RandomAccessFile raf = new RandomAccessFile("./mappedFile.txt","rw");
        java.nio.channels.FileChannel fc = raf.getChannel();
        MappedByteBuffer mappedByteBuffer = fc.map(java.nio.channels.FileChannel.MapMode.PRIVATE,0,fc.size());
    
        //通过filechannel改变文件的的内容
        ByteBuffer buffer = ByteBuffer.wrap("test".getBytes());
        fc.write(buffer);

        //读取mappedByteBuffer的内容
        while (mappedByteBuffer.hasRemaining()){
            System.out.println((char)mappedByteBuffer.get());
        }

        //System.out.println("--------------");
        mappedByteBuffer.flip();
        mappedByteBuffer.put("asdfg".getBytes());

        //对mappedByteBuffer进行put,再读取它的内容.
        mappedByteBuffer.flip();
        while (mappedByteBuffer.hasRemaining()){
            System.out.println((char)mappedByteBuffer.get());
        }

    }

输出结果：

````
testo--------------
asdfg
````






import java.net.URLConnection;

 			contentLength = msg.length(  );











文件数据从一个地方到另一个地方的的大块传输很普遍，所以有很多优化的方法加入到`FileChannel`类中，可以使它更加有效率。

````
implements ByteChannel, GatheringByteChannel, ScatteringByteChannel

public abstract long transferFrom (ReadableByteChannel src,
}
````

`transferTo()`和`transferFrom()`方法允许你从一个channel到另一个channel跨连接传递数据，消除了通过中间buffer传递数据的需要。这些方法只存在于`FileChannel`类，所以参与channel-to-channel传输的必须是`FileChannel`。你不可以在socket channels之间直接传输数据，但是socket channel实现了`WritableByteChannel`和`ReadableByteChannel`，所以文件的内容可以通过`transferTo()`传递给socket，socket中的数据可以通过`transferFrom()`直接读入文件。

直接channel 传输并不更新关联的`FileChannel`的position。要求的数据传输从position参数表示的地方开始，往后传输count个bytes。返回的是实际传输的bytes数，这可能比你要求的数要小。

对于`transferTo()`方法，如果传输的源是一个文件，如果position + count 大于文件大小，那么传输将在文件末尾停止。If the target is a socket in nonblocking mode, the transfer may stop when its send queue is filled, possibly sending nothing if the output queue is already full。同样的。对于`transferFrom()`方法：如果`src`是另一个`FileChannel`，并且到达文件末尾，那么传输会停止。如果`src`是一个非阻塞模式的socket，只有当前处于队列中的数据会被传输（可能没有数据）。在阻塞模式的socket可能部分传输，取决于操作系统，因为网络数据传输的不确定性。很多socket实现会传输当前处于队列中的数据，而不是等待你要求的所有数据都处于队列中。

记住,如果传输时有问题，会抛出`java.io.Exceptioin`。

Channel-to-channel 传输是可以极其快速的,特别是在底层操作系统提供本地支持的时候。某些 操作系统可以不必通过用户空间传递数据而进行直接的数据传输。对于大量的数据传输,这会是一个巨大的帮助。（见例3-6）


package com.dyh.tutorial.nio;

import java.io.FileInputStream;
import java.io.IOException;
import java.nio.channels.Channels;
import java.nio.channels.FileChannel;

/**
 * Created by dengyunhui on 16/10/5.
 */
public class ChannelTransfer {
    public static void main(String[] args) throws IOException {
        FileInputStream in = new FileInputStream("pom.xml");
        FileChannel fc = in.getChannel();
        fc.transferTo(0,fc.size(), Channels.newChannel(System.out));
        fc.close();
        in.close();
    }
}
````

## 3.5 Socket Channels
Socket Channels有不同于file channels的特性。

新的socket channels可以运行在非阻塞模式，并且是可以选择的(selectable)。这两个能力使得大型应用有极好的可扩展性和灵活性，例如网络服务器和中间件。正如我们将可能到的，不再需要一个线程对应一个socket连接（以及遭受管理大量线程的上下文切换的开销）。使用新的NIO类，一个或几个线程可以管理上百或上千个活跃的socket连接，而又很少或没有任何性能损失。

如图3-9所示，有三种socket channel类（`DatagramChannel`, `SocketChannel`, and `ServerSocketChannel`）继承自`AbstractSelectableChannel`。这意味着使用`Selector`对象执行socket channel的readiness selection是可能的。

<img src="./3-9.png"/>

注意到`DatagramChannel`和`SocketChannel`实现了定义了read和write能力的接口，但是`ServerSocketChannel`却没有。`ServerSocketChannel`监听传入的连接并且创建新的`SocketChannel`对象。他自己从不传输数据。

在讨论单独类型的socket channels之前，你应该理解sockets和socket channels之间的关系channel是一个连接I/O服务的管道并提供与该服务交互的方法。对于socket而言，它不会再次实现与之对应的 socket channels类中的 socket 协议 API,而 `java.net` 中已经存在的 socket channels都可以被大多数协议操作重复使用。

所有的socket channels（`DatagramChannel`, `SocketChannel`, and `ServerSocketChannel`）当它们实例化时都会创建一个对应的socket 对象。他们是来自`java.net`中的熟悉的类（`Socket`, `ServerSocket`, and `DatagramSocket`）。执行一个channel的`socket()`方法可以获取对应的socket。此外，`java.net`中的每一个类都有`getChannel()`方法。

尽管每一个socket channel（在`java.nio.channels`中）都有一个关联的`java.net`socket对象，并不是所有的socket 都有关联的channel。如果你用传统的方式创建一个`Socket`对象，即直接实例化，那么它没有一个关联的`SocketChannel`，并且它的`getChannel()`方法总是返回null。

### 3.5.1 Nonblocking Mode
Socket channels可以以非阻塞模式运行。这是一个简单的说明，但是却有深远的含义。传统Java sockets的阻塞特性是Java应用可扩展性的最重要的限制之一。Nonblocking I/O是许多高性能应用建立的基础。

为了将一个socket置为非阻塞模式，我们看看所有socket channel类的父类：`SelectableChannel`。下面的方法和一个channel的阻塞模式有关：


````
public abstract class SelectableChannel
````

Readiness selection是一种机制，通过它一个channel可以查询来确定它是否准备好来执行感兴趣的操作，比如read或write。非阻塞I/O和选择能力是密切关联的。

设置或者重新设置一个channel的阻塞模式是很容易的。调用`configureBlocking()`并传递true可以把它置为阻塞模式，传递false可以把它置为非阻塞模式。就是这么简单。通过执行`isBlocking()`你可以确定socket channel当前是哪种模式。


````
SocketChannel sc = SocketChannel.open();
sc.configureBlocking(false);//nonblocking
if(sc.isBlocking()){
	doSomething();
}
````

非阻塞sockets 通常认为在服务端使用，因为他们使得同时管理大量sockets更加容易。但是在客户端使用一个或几个非阻塞模式的sockets也是有好处的。例如，通过非阻塞模式的sockets，一个GUI应用可以专注于用户请求，同时维护与一个或多个服务器的会话。非阻塞模式对大多数应用是有用的。

偶尔的，我们需要防止改变一个socket channel的阻塞模式。API提供了`blockingLock()`方法，它返回一个不透明的对象应用。返回的对象是channel实现修改阻塞模式时内部使用的。只有线程持有这个对象上的锁时，才可以改变channel的阻塞模式（通过使用java关键词`synchronized`可以获取一个对象锁，这个3.3节的lock方法不同）。对于保证一个socket的阻塞模式不会在代码的临界区改变，或者暂时改变阻塞模式而不会影响其他的线程，这是灰常好用的。

````
Socket socket = null;

我们开始讨论socket channels里最简单的：`ServerSocketChannel`。下面是`ServerSocketChannel`的完整API：

````

 `connect()`方法还没有调用。那么抛出`NoConnectionPendingException`。
 连接建立正在进行但是还没完成。那么什么都不会发生，`finishConnect()`直接返回false。
 在非阻塞模式上调用`connect()`之后，`SocketChannel`又切换回了阻塞模式。那么执行线程阻塞直到连接建立。然后，`finishConnect()`返回true。
 自从初始调用`connect()`之后或最后调用`finishConnect()`之前连接已经建立。那么`SocketChannel`对象中的内部状态已经更新到连接完成的状态，并且`finishConnect()`返回true。`SocketChannel`对象可以用于传输数据。
 连接已经建立。那么什么都不会发生，`finishConnect()`返回true。

public class SocketChannelDemo {
    public static void main(String[] args) throws IOException {
        connect();
    }


    public static void connect() throws IOException {
        SocketChannel sc = SocketChannel.open();
        sc.configureBlocking(false);
        sc.connect(new InetSocketAddress("localhost",9090));
        while (!sc.finishConnect()){
            System.out.println("connection in progress...");
        }

        ByteBuffer buffer = ByteBuffer.allocate(100);
        while (sc.read(buffer) >= 0){
            buffer.flip();
            while (buffer.hasRemaining()){
                System.out.print((char)buffer.get());
            }
            /**
             * 为什么这里一定要clear呢?
             * 因为sc.read() 并不是第一次读取就能成功把数据从服务端读回来,
             * 如果第一次没有读到数据,buffer的position为0
             * 之后调用flip,limit = 0,position=0;
             * 这样buffer就再也不能读取数据了.这就陷入了死循环
             *
             * 所以每次都要clear
             */

            buffer.clear();
        }

        sc.close();
    }
}
 你的应用可以容忍丢失数据和数据乱序。
 你想发送数据包之后就不管了。你并不想知道这个数据包是否收到。
 吞吐量比可靠性更重要。
 你需要同时发送到多个接收者。
 数据包比喻(packet metaphor)比流比喻(stream metaphor)更加适合这个任务。