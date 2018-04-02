文件可以随机存取，这种随机存取是读操作和写操作的混合，直到文件被关闭。Java提供了`java.io.RandomAccessFile`来支持这种随机存取。

`RandomAccessFile`有如下的构造方法：
+ `RandomAccessFile (File file,String mode)`：
+ `RandomAccessFile (String path,String mode)`

每个构造方法的mode参数必须是"r","rw","rws","rwd"中的一个。mode参数的含义如下所示：

+ “r” 告诉构造器以只读方式打开一个已经存在的文件。对这个文件的任何写操作都会抛出异常。
+ "rw" 告诉构造器以读写方式创建和打开一个新的文件（当这个文件不存在时），或者以读写方式打开一个已经存在的文件。
+ "rwd" 告诉构造器以读写方式创建和打开一个新的文件（当这个文件不存在时），或者以读写方式打开一个已经存在的文件。而且，每一个对文件内容的更新必须同步的更新到底层的存储设备上。
+ "rws" 告诉构造器以读写方式创建和打开一个新的文件（当这个文件不存在时），或者以读写方式打开一个已经存在的文件。而且，每一个对文件内容或者元数据（metadata）的更新必须同步的更新到底层的存储设备上。

“rwd”和"rws" 模式保证了每一个对位于本地存储设备上的文件的写操作都同步的写入至底层存储设备，保证了当操作系统崩溃时，重要的数据不会丢失。 当文件不存在与本地时，并不会做出任何的保证。

> 以“rwd”和"rws"模式打开的文件的操作要比以"rw"模式打开的文件上的同样的操作要慢。

### File Pointer

一个随机存取的文件与一个文件指针关联，文件指针是一个游标，表明下一个要读/写的byte的位置。当打开一个文件时，文件指针被设置为0。当一个文件刚被创建时，文件指针也被设置为0.

读/写操作从文件指针开始，随后文件指针移动到读/写的byte数。

### FileDescriptor
当一个文件被打开，底层操作系统会创建一个与操作系统相关的结构来表示这个文件。这个结构的句柄被存储在`java.io.FileDescriptor`的实例中。`RandomAccessFile.getFD()`返回的就是这个`FileDescriptor`。

> FileDescriptor是当需要底层操作系统执行文件操作时，Java传递给底层操作系统的，用于标识一个具体的打开的文件的标识符。

FileDescriptor是一个小的类，定义了三个FileDescriptor常量，`in`,`out`,`err`.这三个常量让`System.in`,`System.out`,`System.err`提供了对标准输出，标准输入和标准错误流的访问。

FileDescriptor声明了一对方法：
+ `void sync()` 告诉底层操作系统将打开的文件的output buffer中的内容flush到磁盘。当所有的modified data和文件元数据同步到相应的设备时，`sync()`方法返回。当buffer不能flush或者操作系统不能保证所有的buffers已经同步到物理介质中时，抛出`SyncFailedException`。

+ `boolean valid()` 决定这个FileDescriptor是否有效。

写入到一个打开的文件中的数据被存储到底层操作系统的output buffers中。当buffers填满了，操作系统再把buffer中的data刷入硬盘中。
buffer提高了性能，因为对硬盘的访问要比对内存的访问慢得多。

但是当以"rwd"或者"rws"模式打开文件时，每一个写操作的数据直接被写入了硬盘，所以写操作要比以"rw"模式打开的要慢得多。

假如你有需要结合通过buffer写入数据和直接写入数据到磁盘的情况，你可以使用如下的方式：

````
RandomAccessFile file = new RandomAccessFile("","rw");
FileDescriptor fd = file.getFD();

// Perform a critical write operation.
file.write(...);

// Synchronize with the underlying disk by flushing the operating system
// output buffers to the disk.
fd.sync();

// Perform a non-critical write operation where synchronization isn't necessary
file.write(...);

// Do other work here.

// Close the file, emptying output buffers to the disk.
file.close();
````

## Using RandomAccessFile
RandomAccessFile用来建立平坦的文件数据库，一旦文件组织成records和fields很有用。
