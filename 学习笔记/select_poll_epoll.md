I/O多路复用允许应用在多个文件描述符上同时阻塞，并在其中某个可以读写时收到通知。这时I/O多路复用就成了应用的关键所在，一般来讲I/O多路复用的设计遵循以下原则：

1. I/O多路复用：当任何文件描述符准备好I/O时告诉我

2. 在一个或多个文件描述符就绪前始终处于睡眠状态

3. 唤醒：哪个准备好了？

4. 在不阻塞的情况下处理所有I/O就绪的文件描述符

5. 返回第一步，重新开始

Linux提供了三种I/O多路复用方案：select,poll和epoll。

### select
select系统调用提供了一种实现同步I/O 多路复用的机制：

````
int select (int n,
                 fd_set *readfds,
                 fd_set *writefds,
                 fd_set *exceptfds,
                 struct timeval *timeout);

FD_CLR(int fd, fd_set *set);
FD_ISSET(int fd, fd_set *set);
FD_SET(int fd, fd_set *set);
FD_ZERO(fd_set *set);
````

在指定的文件描述符准备好I/O之前或者超过一定的时间限制，select()系统调用都会阻塞。

监控的文件描述符分为3类，分别等待不同的事件。

1. readfds.监控readfds集合中的文件描述符，确认其中是否有可读数据（也就是说读操作可以无阻塞的完成）

2. writefds.监控writefds集合中的文件描述符，确认其中是否有一个写操作可以不阻塞地完成。

3. exceptfds.监控exceptfds中的文件描述符，确认其中是否有出现异常或出现out of band数据。

指定的集合可以为空（NULL），相应的，select()则不对此类事件进行监控。

成功返回时，每个集合中只包含对应类型的I/O就绪了的文件描述符。

**第一个参数n,等于集合中所有文件描述符的最大值加1.这样，select的调用者需要找到最大的文件描述符值，并将其加1后传给第一个参数。**（所以可能需要遍历）

timeout参数是一个指向timeval结构体的指针，定义如下：

````
#include <sys/time.h>

struct timeval {
           long tv_sec; /* seconds */
           long tv_usec; /* microseconds */
};
````

如果这个值不是NULL，即使此时没有文件描述符处于I/O就绪状态，select调用也将在tv_sec秒tv_usec微妙后返回。返回时，这个结构体的状态在大多数Unix系统中都是未定义的。这样的话，每次调用前都必须重新初始化（还有集合中的文件描述符）。较新版本的Linux会自动将该值改为剩余的时间。这样如果时限时5秒，在某个文件描述符准备好时过去了3秒，tv.tv_sec在返回时就是2.对于这一种情况，等待也会被信号中断。

如果时限中两个值都是零，调用会立即返回，加入描述符集的描述符都会被测试，并且返回满足要求的描述符的个数。这种方法通过轮询，无阻塞的获得了多个文件描述符的状态。

如果这个值是NULL，无限等待。等待可以被一个信号中断。当有一个描述符做好准备或者是捕获到一个信号时函数返回。如果捕获到一个信号，select函数返回 -1，并将变量erro设为EINTR。


文件描述符集合保存在fd_set类型中。fd_set一般实现为bit 数组。我们可以认为数组的每一位（bit）代表了一个文件描述符。

集合中的文件描述符并不直接操作，而是通过辅助宏来进行管理。

````
FD_CLR(int fd, fd_set *set);
FD_ISSET(int fd, fd_set *set);
FD_SET(int fd, fd_set *set);
FD_ZERO(fd_set *set);
````

FD_ZERO,从指定集合中移除所有文件描述符。` FD_ZERO(&writefds);`

FD_SET，向指定集合中添加一个文件描述符。`FD_SET(fd, &writefds);`

FD_CLR，从指定集合中移除一个文件描述符。`FD_CLR(fd, &writefds);`

FD_ISSET,测试一个文件描述符在不在给定集合中。如果在，则返回一个非零值；如果不在，返回0。一般在select调用返回后，使用FD_ISSET来检查一个文件描述符是否就绪。`if (FD_ISSET(fd, &readfds))`

由于文件描述符集合是静态建立的，所以对文件描述符数量的上限和文件描述符的最大值均有限制，两者都由FD_SETSIZE设定。在Linux上，这个值是1024.

#### select 模型的特点：

1. 可监控的文件描述符的个数取决于sizeof(fd_set)的值。因为fd_set中的每一bit都可以表示一个文件描述符，则支持的最大文件描述符个数为: `8 * sizeof(fd_set)`。这个值可以由FD_SETSIZE设定。

2. 将fd加入select监控集的同时，还要再使用一个数据结构array保存放到select监控集中的fd，一是用于再select返回后，array作为源数据和fd_set进行FD_ISSET判断。二是select返回后会把以前加入的但并无事件发生的fd清空，则每次开始 select前都要重新从array取得fd逐一加入（FD_ZERO最先），遍历array的同时取得fd最大值maxfd，用于select的第一个参数。

3. 可见select模型必须在select前遍历array（加fd，取maxfd），select返回后遍历array（FD_ISSET判断是否有时间发生）。**所以,当array较大时，遍历数组也要花费较多时间***。

### poll
poll系统调用是System V的I/O多路复用解决方案。他解决了一些select的不足。

````
#include <sys/poll.h>

int poll (struct pollfd *fds, unsigned int nfds,int timeout);
````
与select使用的三个基于位掩码的文件描述符集合不同，poll使用一个简单的nfds个pollfd结构体构成的数组，fds指向该数组。结构体定义如下：

````
#include <sys/poll.h>

struct pollfd {
    int fd; /* file descriptor */
    short events; /* requested events to watch */
    short revents; /* returned events witnessed */
};
````

每个pollfd结构体指定监视单一的文件描述符。可以传递多个结构体，使得poll监视多个文件描述符。每个结构体的events字段是要监视的文件描述符事件的一组bit 码，用户设置这个字段。revents字段则是发生在该文件描述符上的事件（也就是已经准备好的事件）的bit码，内核在返回时设置这个字段。所有的events字段请求的事件都可能在revents字段中返回。下面是合法的事件：

| 事件 | 含义 |
| :------------- | :------------- |
|   POLLIN     |  没有数据可读     |
|        POLLRDNORM |   有正常数据可读   |
|    POLLRDBAND    |   有优先数据可读    |
|    POLLPRI    |  有高优先级数据可读     |
|   POLLOUT     |   写操作不会阻塞    |
|   POLLWRNORM     |   写正常数据不会阻塞    |
|   POLLBAND     |     写优先数据不会阻塞  |
|   POLLMSG     |    有一个SIGPOLL 消息可用   |

另外，如下事件可能在revents中返回：

+ POLLER,给定的文件描述符上有错误
+ POLLHUP,文件描述符上有挂起事件
+ POLLNVAL,给定的文件描述符非法

timeout参数指定在任何I/O就绪前需要等待时间的长度，以毫秒计。负值表示永远等待。零表示调用立即返回，列出所有未准备好的I/O，但不等待其他事件。在这种情况下，poll 就如同其名，轮询一次后立即返回。

返回值和错误码

成功时，poll 返回具有非零revents字段的文件描述符个数。超时前没有任何事件发生则返回零。失败返回 -1,errno被设置为下列值之一：

+ EBADF,一个或多个结构体有非法的文件描述符。
+ EFAULT,指向fds的指针超出了进程地址空间
+ EINTR,在请求事件发生前收到了一个信号，可以重新调用
+ EINVAL,nfds参数超过了RLIMIT_NOFILE值
+ ENOMEM,没有足够的内存完成请求

#### poll与select
尽管他们完成的工作一样，但poll系统调用仍然由于select：

+ poll 无需使用者计算文件描述符的最大值加1，和传递该参数。

+ poll 在应对较大值的文件描述符时更有效率。想象一下用select监视900个文件描述符---内核需要检查每个集合中的每一个bit。

+ select的文件描述符是静态大小的，所以需要作出权衡：要么集合很小，限制了select可以监视的文件描述符的最大值，要么较大，但是效率不高。尤其是当不能确定集合的组成是否稀疏时，对较大位掩码的操作效率不高。使用poll则可以创建合适大小的数组。只需要监视一项或仅仅传递一个结构体。

+ 若用select,文件描述符集合会在返回返回时重新创建，这样的话，之后每个调用都必须重新初始化它们。poll 系统调用分离了输入(events)和输出(revents)，数组无需改变即可重用。

但是select系统调用有几个不错的地方：

+ poll 由于某些Unix系统不支持，所以select 的可移植性更好
+ select 提供了更好的超时方案：微妙级。

### epoll
poll 和 select 每次调用时都需要所有被监视的文件描述符。内核必须遍历所有被监视的文件描述符。当这个表变得很大时---包含上百，甚至上千个文件描述符---每次调用时的遍历就成了明显的瓶颈。

epoll 把监听注册从实际监听中分离出来，从而解决了这个问题。一个系统调用初始化一个epoll上下文，另一个从上下文中加入或删除需要监视的文件描述符，第三个执行真正的事件等待。

#### 1.创建一个新的epoll实例
使用epoll_create创建一个epoll上下文：

````
#include <sys/epoll.h>
int epoll_create (int size)
````

调用成功，epoll_create创建一个epoll实例，返回与该实例关联的文件描述符。这个文件描述符和真正的文件没有关系，仅仅是为了后续调用使用epoll而创建。size 参数告诉内核需要监听的文件描述符数目，但不是最大值。传递一个适当的近似值会带来性能的提升，但不需要给出确切的数字。

#### 2.控制epoll
epoll_ctl可以向指定的epoll上下文中加入或删除文件描述符：

````
#include <sys/epoll.h>
int epoll_ctl (int epfd, int op, int fd, struct
        epoll_event *event);
````

epoll event 结构体：

````
struct epoll_event {
       __u32 events; /* events */
       union {
         void *ptr;
         int fd;
         __u32 u32;
         __u64 u64;
} data; };
````

epoll_ctl成功调用将关联epoll实例和epfd.参数op 指定对fd要进行的操作。event 参数描述epoll更具体的行为。

以下是op 的有效值：

+ EPOLL_CTL_ADD:把fd指定的文件添加到epfd指定的epoll实例监听集中，监听event中定义的事件。
+ EPOLL_CTL_DEL:把fd指定的文件从epfd指定的epoll监听集中删除
+ EPOLL_CTL_MOD:使用event改变在已有fd上的监听行为

epoll_event中的data字段由用户使用。确认监听事件后，data会被返回给用户。通常将event.data.fd设定为fd,这样就知道哪个文件描述符触发事件。

#### 3.等待epoll事件
epoll_wait等待给定epoll实例关联的文件描述符上的事件：

````
#include <sys/epoll.h>

int epoll_wait (int epfd, struct epoll_event *events, int maxevents, int timeout);
````

对epoll_wait的调用等待epoll实例epfd中的文件fd上的事件，时限timeout毫秒。成功返回时，events指向包含epoll_event结构体（该结构体描述了每个事件）的内存，且最多可以有maxevents个事件。返回值是事件数，出错返回-1，并设置errno值。

如果timeout为0，即使没有事件发生，调用也立即返回，此时调用返回0.如果timeout为-1,调用将一直等待到有事件发生。

当调用返回，epoll_event结构体中的events字段描述了发生的事件。data 字段包含了所有用户在调用epoll_ctl前的设置。

#### 边沿触发和水平触发
如果epoll_ctl的参数event中的events项设置为EPOLLE，fd上的监听称为边沿触发，相反的称为水平触发。考虑下面的生产者和消费者在通过unix管道通信时的情况。

1. 生产者向管道写入1kb数据。
2. 消费者在管道上调用epoll_wait，等待pipe出现数据，从而可读。

对于水平触发的监听，在步骤2里对epoll_wait的调用将立即返回，以表明pipe可读。对于边沿触发的监听，这个调用直到步骤1发生后才会返回。也就是说即使调用epoll_wait时管道已经可读，调用仍然会等待直到有数据可写，之后返回。

水平触发是默认的行为。也就是poll和select的行为。

边沿触发需要一个不同的方式来写程序，通常利用非阻塞I/O，并需要仔细检查EAGAIN。

水平触发在一个状态发生时触发；边沿触发只有在状态改变时才会产生。水平触发在只关心状态时有用。边沿触发则在关心事件本身时有用。


水平触发，只要可读或可写，事件会一直触发

边缘触发，只有从不可读变为可读、从不可写变成可写，事件才会触发

基于此，水平触发，socket不需要设置为non-block，因为只要触发，就可以读或写。

而对于边缘触发，事件一旦触发，需要用户一直读，直到缓冲区数据全部读完为止才可以，如果设置为block，在没有数据可读时，就block了，需要用eagain错误来告知用户，数据读完了，不用再读了。

ET 模式是一种边沿触发模型，在它检测到有 I/O 事件时，通过 epoll_wait 调用会得到有事件通知的文件描述符，每于每一个被通知的文件描述符，如可读，则必须将该文件描述符一直读到空，让 errno 返回 EAGAIN 为止，否则下次的 epoll_wait 不会返

select是顺序扫描（遍历）fd是否就绪，而且支持的fd数量有限。epoll是等待事件发生，使用事件驱动方式代替扫描，因此性能更高。当有fd就绪时，立即回调函数。使用select，有O(n)的无差别轮询复杂度，同时处理的流越多，每一次无差别轮询时间就越长(和fd数量有关)。epoll可以理解为event poll，不同于忙轮询和无差别轮询，epoll之会把哪个流发生了怎样的I/O事件通知我们。此时我们对这些流的操作都是有意义的。（复杂度降低到了O(1)）。nfds = epoll_wait,epoll_wait返回的是已经ready的fd。所以epoll只轮询已经ready的文件描述符。

上文提到了epoll可以将哪个流发生了什么事件主动通知，而不需要轮询。那么到底是如何实现通知的呢？

这就和操作系统的原理相关，在内核的最底层是中断，类似系统回调的机制。网卡设备对应一个中断号, 当网卡收到网络端的消息的时候会向CPU发起中断请求, 然后CPU处理该请求. 通过驱动程序 进而操作系统得到通知, 系统然后通知epoll, epoll通知用户代码。它能显著提高程序在大量并发连接中只有少量活跃的情况下的系统CPU利用率，因为它会复用文件描述符集合来传递结果而不用迫使开发者每次等待事件之前都必须重新准备要被侦听的文件描述符集合，另一点原因就是获取事件的时候，它无须遍历整个被侦听的描述符集，只要遍历那些被内核IO事件异步唤醒而加入Ready队列的描述符集合就行了。


http://wiki.xiaohansong.com/linux/epoll.html
https://www.freebsd.org/cgi/man.cgi?query=epoll&apropos=0&sektion=0&manpath=SuSE
