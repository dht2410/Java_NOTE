## epoll

### 1、epoll_create

系统调用为

```c
//返回eventpoll结构对应的文件描述符
int epoll_create(int size);
```

该系统调用首先创建一个eventpoll数据结构，并用一个文件file表示其。file的private域指向eventpoll，返回的文件描述符为该file。

```c
//eventpoll数据结构
struct eventpoll
{
    spin_lock_t       lock;         // 对本数据结构的访问
    struct mutex      mtx;          // 防止使用时被删除
    /*
     * 等待队列可以看作保存进程的容器，在阻塞进程时，将进程放入等待队列；
     * 当唤醒进程时，从等待队列中取出进程
     */ 
    wait_queue_head_t   wq;         // sys_epoll_wait() 使用的等待队列
    wait_queue_head_t   poll_wait;  // file->poll()使用的等待队列
    struct list_head    rdllist;    // 就绪链表
    struct rb_root      rbr;        // 用于管理所有fd的红黑树（树根）
    struct epitem      *ovflist;    // 将事件到达的fd进行链接起来发送至用户空间
}
```

```c
//每个被关注的fd对应的epitem数据结构
struct epitem 
{
    struct rb_node  rbn;        // 用于主结构管理的红黑树
    struct list_head  rdllink;  // 事件就绪链表
    struct epitem  *next;       // 用于主结构体中的链表
 	struct epoll_filefd  ffd;   // 这个结构体对应的被监听的文件描述符信息
 	int  nwait;                 // poll操作中事件的个数
    struct list_head  pwqlist;  // 双向链表，保存着被监视文件的等待队列
    struct eventpoll  *ep;      // 该项属于哪个主结构体（多个epitm从属于一个eventpoll）
    struct list_head  fllink;   // 双向链表，用来链接被监视的文件描述符对应的struct
    struct epoll_event  event;  // 注册的感兴趣的事件，也就是用户空间的epoll_event
}
```

### 2、create_ctl

系统调用为

```c
//成功返回0，失败返回-1
//epfd为eventpoll对应的文件描述符
//op代表向eventpoll中增加、修改或删除epitem
//fd为关注的文件，例如socket的文件描述符
//event为关注的事件
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```

struct epoll_event定义为

```c
typedef union epoll_data {
    void        *ptr;
    int          fd;
    __uint32_t   u32;
    __uint64_t   u64;
} epoll_data_t;

struct epoll_event {
    __uint32_t   events;      /* Epoll events */
    epoll_data_t data;        /* User data variable */
};
```

以添加关注的fd为例，该系统调用首先检查ep的红黑树中是否已存在这个fd，如果没有，则将该fd对应的epitem插入红黑树。调用的为ep_insert方法。该方法中，调用ep_ptable_queue_proc方法将回调函数ep_poll_callback注册到epitem中。

因此当该fd所对应的socket发生变化时，中断程序调用ep_poll_callback函数。

- 首先判断产生的事件是否包含在epitem中设置的关注事件中
- 如果是，则将该epitem放到ep的就绪列表中
- 唤醒等待队列中等待的进程或线程（调用event_wait阻塞在这里的）

### 3、epoll_wait

系统调用为

```c
//返回值为关注的fd中有事件到来的fd数量
//events负责承接内核返回给用户空间的数据
//maxevents为返回的最多的events数量
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

该调用调用**ep_poll**，当**rdlist**为**空**（无就绪fd）时**挂起当前进程**，直到**rdlist不空时进程才被唤醒**。然后就将就绪的events和data发送到用户空间（**ep_send_events()**），如果ep_send_events()返回的事件数为0，并且还有超时时间剩余(jtimeout)，那么我们**retry**，期待不要空手而归。

**ep_send_events**函数，它扫描txlist中的每个epitem，调用其关联的fd对应的的**poll**方法，取得fd上较新的events（防止之前events被更新）即**revents**，之后将**revents**和相应的**data**拷贝（**__put_user()**）到用户空间。如果这个epitem对应的fd是**LT模式**监听且取得的events是用户所关心的，**则将其重新加入回rdlist**，如果是**ET模式**则不再加入**rdlist**。

