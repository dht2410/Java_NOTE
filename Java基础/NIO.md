## NIO

### IO模型

> **1、完全阻塞模型**
>
> ![preview](https://pic1.zhimg.com/v2-6cd259374d93454373ecaae707962aa2_r.jpg)
>
> 就好比，我叫个外卖，然后我就去大门口傻等着，外卖不送到，我也什么都不做，就坐在门口打盹，直到外卖小哥过来，把我叫醒，我才拿着外卖回家去吃。

> **2、非阻塞IO**
>
> 对非阻塞fd调用系统接口时，不需要等待事件发生而立即返回，事件没有发生，接口返回-1
>
> ![preview](https://pic2.zhimg.com/v2-2489dcdb735214fc1becf3e220228c22_r.jpg)
>
> 仍然以外卖举例，就相当于，我一边扫地，一边等外卖。我不再像原来一样，在门口傻等了，而是扫两下，就跑到门口看看外卖到了没有。一直这样循环，直到我取到外卖，才从这个循环中跳出来，进入吃的流程。
>
> ```c
> int flags = fcntl(sock_fd, F_GETFL, 0); 
> fcntl(sock_fd, F_SETFL,flags | O_NONBLOCK);
> 
> int total = 0;
> 
> while (total < 12) {
>     n = recv(sock_fd, buff, MAXLINE, 0); 
>     if (n >= 0) {
>         printf("I can do something else %d\n", n); 
>         total += n;
>     }   
> }
> ```
>
> 本质上，我们使用了一个自旋在这里不断地检查数据，直到数据到达以后，才会跳出这个循环。所以，这种写法仍然是服务端在等待客户端，仍然是一种同步模型。
>
> **同步一定阻塞吗？**
>
> **同步和阻塞并不是等价的。同步的意义只是说客户端发过来的数据到达之前，我干不了其他的事情。而阻塞强调的是调用一个函数，线程会不会休眠。在上面的非阻塞模型里，虽然调用recv不会再使线程休眠了，但程序并没有去执行什么有效的逻辑，所以这本质上仍然是一个同步模型。**

> **3、IO多路复用**
>
> 最常用的I/O事件通知机制就是I/O复用(I/O multiplexing)。Linux 环境中使用select/poll/epoll 实现I/O复用。
>
> ![preview](https://pic2.zhimg.com/v2-af19da1f5685cd8903e9c3c284fda3af_r.jpg)
>
> 还是外卖的例子，如果我们整栋楼的人，很多人叫了外卖，都有下楼来看外卖到没到的需求，于是物业就出了个招，让门卫小哥帮大家看着，整栋楼上的，不管是谁的外卖到了，先放到门卫小哥那里，然后门卫小哥再通知你下来拿自己的外卖。这样一来，我们就把本来多个人要跑去看自己的外卖到了这件事交给门卫小哥去做了。而我们解放出来，就可以继续看电视，打扫卫生，刷知乎了。由于我们可以继续做自己的事情，外卖小哥和门卫小哥在同时也在工作，互不干扰，所以这种工作方式就被称为**异步模型**

> **4、SIGIO**
>
> 与IO多路复用不同的是，在等待数据达到期间，I/O复用是会阻塞应用程序，而SIGIO方式是不会阻塞应用程序的。
>
> ![preview](https://pic2.zhimg.com/v2-465198aef74ca1753ed22f2f29b7e3e3_r.jpg)
>
> 很少使用。

> **5、async io**
>
> ![preview](https://pic2.zhimg.com/v2-70b311e60fc53f292af63108d145a669_r.jpg)
>
> 纯异步。回调。

### 阻塞和非阻塞

为什么阻塞式接口 + 多线程会遇到瓶颈：

一个线程对应一个连接。当连接多时，线程切换会严重影响性能。

解决方案就是IO多路复用，要搞清楚，Java的多路复用不过是操作系统相关调用的封装。比如 select / poll / epoll / kqueued 等等。

### IO多路复用

poll 就是我们引入的门卫小哥，可以帮助服务端把所有的客户端 socket 连接都管理起来。poll 可以同时观察许多 socket 的I/O事件， 如果所有的 socket 都空闲的时候，会把当前线程阻塞掉，当有一个或多个 socket 有I/O事件时，就从阻塞态中醒来，并且，它的返回值就是有IO事件的那一个或多个 socket。我们得到这些返回值以后，就可以逐个地处理这些socket上的IO事件了。这样，我们就把多个客户端，每个都要去做的轮询交给一个 poll 去实现了。

NIO一个重要的特点是：socket主要的读、写、注册和接收函数，在等待就绪阶段都是非阻塞的，真正的I/O操作是同步阻塞的（消耗CPU但性能非常高）。

### Selector

在Java中，Selector这个类是select/epoll/poll的外包类。这是NIO的核心。

channel是为了方便Selector管理socket来进行的一层封装。

![preview](https://pic1.zhimg.com/v2-80af6295fcb000339ce71a546d04e5cf_r.jpg)

所有的Channel都归Selector管理，这些channel中只要有至少一个有IO动作，就可以通过Selector.select方法检测到，并且使用selectedKeys得到这些有IO的channel，然后对它们调用相应的IO操作。

### Buffer

本质上，缓冲区就是一个数组。

4个属性：

> 1. 容量（Capacity） 缓冲区能够容纳的数据元素的最大数量。容量在缓冲区创建时被设定，并且永远不能被改变。
> 2. 上界（Limit） 缓冲区里的数据的总数，代表了当前缓冲区中一共有多少数据。
> 3. 位置（Position） 下一个要被读或写的元素的位置。Position会自动由相应的 get( )和 put( )函数更新。
> 4. 标记（Mark） 一个备忘位置。用于记录上一次读写的位置。一会儿，我会通过reset方法来说明这个属性的含义。

![preview](https://pic2.zhimg.com/v2-07952bbba1ff33a30b213a4941b4242e_r.jpg)

实现：

- 堆内存储数据	HeapByteBuffer
- 堆外存储数据    DirectByteBuffer    类似C语言直接内存管理
- 文件映射    MappedByteBuffer



### Channel

NIO中通过channel封装了对数据源的操作，通过 channel 我们可以操作数据源，但又不必关心数据源的具体物理结构。

这个数据源可能是多种的。比如，可以是文件，也可以是网络socket。在大多数应用中，channel 与文件描述符或者 socket 是一一对应的。



## epoll

redis	epoll     epoll_wait轮询   因为redis是单线程，除了epoll_wait还要干其他事。

6.x版本之前，单线程负责read/计算/write。之后，引入IO threads，epoll通知后由IO thread负责读写，单线程只负责进程。



nginx	epoll      epoll_wait阻塞    只等着socket

netty	epoll

用户程序不能直接访问硬件

Linux中 /proc/下都是进程。进入进程ID命名的目录后，task目录下是该进程的线程。

strace -ff -o ./xxxx java TestSocket // 可以看程序内核发生的调用



#### 1、每线程对应一个client模式（BIO）

![image-20200803135249575](C:\Users\dht24\AppData\Roaming\Typora\typora-user-images\image-20200803135249575.png)

accept和recvfrom都是阻塞的

fd5是监听socket的文件描述符，fd6及以后是连接socket的文件描述符

accept的其中一个参数是fd5

recvfrom的其中一个参数是fd6

抛出一个线程去执行，意味着accept只在主线程，recvfrom的每一个连接socket都在子线程中进行

**问题：**

**每个线程都通过系统调用克隆出来，同时线程消耗内存资源，栈是独立的**

**线程切换浪费时间**



#### 2、非阻塞

系统调用socket有不同的non-blocking和blocking两个选项

采用non-blocking时不阻塞，循环调用

**问题：但如果有1w个客户端连接，那么每循环要进行1w次系统调用。O(n)复杂度**

#### 3、select

```c
// ndfs 文件描述符个数
// *readfds 文件描述符
int select(int nfds, fd_set *readfds, fd_set *writefds,
                  fd_set *exceptfds, struct timeval *timeout);

// select()  and pselect() allow a program to monitor multiple file descriptors, waiting until one or more of the file descriptors become "ready" for some class of I/O operation (e.g., input possible).  A file descriptor is considered ready if it is possible  to  perform  the  corresponding  I/O operation (e.g., read(2)) without blocking.
```

![image-20200803155921259](C:\Users\dht24\AppData\Roaming\Typora\typora-user-images\image-20200803155921259.png)

问题：

每次程序要把文件描述符们传进内核，fds每次都要拷贝。

内核每次**主动遍历**一遍所有文件描述符。

#### 4、epoll

解决：

内核内开辟一个空间，每建立一个socket连接就把它扔进去。基于事件驱动，来了数据后，放到一个list里。程序只去这个list取就可以。

充分发挥硬件，尽量不浪费CPU。

> epoll_create: 创建一个epoll实例，返回这个实例的文件描述符。
>
> An epoll instance created by epoll_create(2), which returns a file descriptor referring to the epoll instance.   (The  more  recent  epoll_create1(2) extends the functionality of epoll_create(2).)

> epoll_ctl: 在这个epoll实例里创建对特定文件描述符的关注。
>
> ```c
> //epfd 是epoll_create返回的文件描述符
> //op是执行的操作，可以有add/delete/mod
> //fd是在这个epfd里注册的文件描述符
> int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
> //伪代码
> epoll_ctl(fd8,add,fd5,accept)
> ```
>
> Interest  in  particular  file  descriptors  is then registered via epoll_ctl(2).  The set of file descriptors currently registered on an epoll instance is sometimes called an epoll set.

> epoll_wait: 等待epoll的响应。同步。
>
> ```c
> int epoll_wait(int epfd, struct epoll_event *events,
>                       int maxevents, int timeout);
> ```
>
> 

![image-20200803162928909](C:\Users\dht24\AppData\Roaming\Typora\typora-user-images\image-20200803162928909.png)

1. epoll_create首先创建一个epoll实例
2. epoll_clt将fd5代表的那个监听secket注册进去这个epoll实例，event为accept。
3. epoll_wait(fd8)，等待这个epoll的响应。这时如果有一个连接过来了，那么会有一个硬中断产生。中断程序让epoll会把fd5移到list里面，并通知程序。返回一个连接socket文件描述符fd6。
4. epoll_clt(fd8,add,fd6,w/r)，把fd6加入到epoll实例中，也就是加到一个文件描述符表中。这个时候表里面有fd5和fd6。

**多路复用返回的是状态，读写还是要程序自己做系统调用来完成。这是同步的。**



## 零拷贝

kafka     epoll

基于jvm	持久    存磁盘

mmap	不进行系统调用	直接打通了用户空间和内核空间	

写的时候	segment	1G			mmap在内存中开1G内存	直接写到磁盘中，不进行系统调用



正常读的顺序：数据从磁盘——》内核——》用户空间——》内核——》发出去

零拷贝：数据从磁盘——》内核——》发出去

适合不需要用户空间加工的数据

零拷贝系统调用：sendfile

```c
ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
```

sendfile() copies data between one file descriptor and another.  Because **this copying is done within the kernel**, sendfile() is more efficient than the combination of read(2) and write(2), which would require transferring data to and from user space.





