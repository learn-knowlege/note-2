# I/O 多路复用

什么是多路复用？

> 一个进程监听多个文件描述符。在 Linux 中，一切皆文件。

I/O 多路复用的技术 解决了 C10k 问题。

> C10K 问题本质上是操作系统处理大并发请求的问题。对于 Web 时代的操作系统而言，对于客户端过来的大量的并发请求，
> 需要创建相应的服务进程或线程。这些进程或线程多了，导致数据拷贝频繁（缓存 I/O、内核将数据拷贝到用户进程空间、阻塞），
> 进程 / 线程上下文切换消耗大，从而导致资源被耗尽而崩溃。这就是 C10K 问题的本质。


## select

**缺点**

> 1024并发数限制
>
> 内存拷贝，效率低


## epoll

epoll是Linux内核的可扩展I/O事件通知机制。于 Linux 2.5.44首度登场，它设计目的旨在取代 select 与 poll 系统函数。
让需要大量操作文件描述符的程序得以发挥更优异的性能。在高并发场景，随着文件描述符的增长，有良好的可扩展性。

> select 与 poll 时间复杂度为 O(n);
> 
> epoll的时间复杂度 O(log n)。

epoll 实现的功能与 poll 类似，都是监听多个文件描述符上的事件。`nginx`、`redis`、`java NIO(Linux)` 都是用 `epoll` 模型进行实现的。

epoll 与 FreeBSD 的 kqueue 类似，底层都是由可配置的操作系统内核对象建构而成，并以文件描述符(file descriptor)的形式呈现于用户空间。epoll 通过使用红黑树(RB-tree)搜索被监控的文件描述符。

在 epoll 实例上注册事件时，epoll 会将该事件添加到 epoll 实例的红黑树上并注册一个回调函数，当事件发生时会将事件添加到就绪链表中。

在 Linux 系统下，可以查看到用户能注册到epoll实例中的最大文件描述符的数量限制。

```
[root@centos7 note]# cat /proc/sys/fs/epoll/max_user_watches
789913
```


**epoll_event 数据结构**

```
typedef union epoll_data
{
  void *ptr;
  int fd;               /* 建立连接的文件描述符 */
  uint32_t u32;
  uint64_t u64;
} epoll_data_t;

struct epoll_event
{
  uint32_t events;      /* Epoll events */ 事件宏
  epoll_data_t data;    /* User data variable */
};
```

**事件宏**

> EPOLLIN ： 表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
>
> EPOLLOUT： 表示对应的文件描述符可以写；
>
> EPOLLPRI： 表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
>
> EPOLLERR： 表示对应的文件描述符发生错误；
>
> EPOLLHUP： 表示对应的文件描述符被挂断；
>
> EPOLLET： 将 EPOLL设为边缘触发(Edge Triggered)模式（默认为水平触发），这是相对于水平触发(Level Triggered)来说的。
>
> EPOLLONESHOT： 只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里

**程序接口**

```
int epoll_create(int size);
```

在内核中创建epoll实例并返回一个epoll文件描述符。 在最初的实现中，调用者通过 size 参数告知内核需要监听的文件描述符数量。如果监听的文件描述符数量超过 size, 则内核会自动扩容。而现在 size 已经没有这种语义了，但是调用者调用时 size 依然必须大于 0，以保证后向兼容性。

```
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```

向 epfd 对应的内核epoll 实例添加、修改或删除对 fd 上事件 event 的监听。op 可以为 EPOLL_CTL_ADD, EPOLL_CTL_MOD, EPOLL_CTL_DEL 分别对应的是添加新的事件，修改文件描述符上监听的事件类型，从实例上删除一个事件。如果 event 的 events 属性设置了 EPOLLET flag，那么监听该事件的方式是边缘触发。

```
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```
当 timeout 为 0 时，epoll_wait 永远会立即返回。而 timeout 为 -1 时，epoll_wait 会一直阻塞直到任一已注册的事件变为就绪。当 timeout 为一正整数时，epoll 会阻塞直到计时 timeout 毫秒终了或已注册的事件变为就绪。因为内核调度延迟，阻塞的时间可能会略微超过 timeout 毫秒。


**实现**

```
#define MAX_EVENTS 10
struct epoll_event ev, events[MAX_EVENTS];
int listen_sock, conn_sock, nfds, epollfd;

/* Code to set up listening socket, 'listen_sock',
  (socket(), bind(), listen()) omitted */

epollfd = epoll_create(1);          /* 建立红黑树，自动扩展 */
if (epollfd == -1) { 
   perror("epoll_create1");
   exit(EXIT_FAILURE);
}

ev.events = EPOLLIN;
ev.data.fd = listen_sock;          /* server fd，接收连接开启、关闭 */
if (epoll_ctl(epollfd, EPOLL_CTL_ADD, listen_sock, &ev) == -1) {  
   perror("epoll_ctl: listen_sock");
   exit(EXIT_FAILURE);
}

for (;;) {
   nfds = epoll_wait(epollfd, events, MAX_EVENTS, -1);   /* 有消息的事件个数 */
   if (nfds == -1) {
       perror("epoll_wait");
       exit(EXIT_FAILURE);
   }

   for (n = 0; n < nfds; ++n) {
       if (events[n].data.fd == listen_sock) {
           conn_sock = accept(listen_sock,       /* conn fd，接收数据 */
                              (struct sockaddr *) &addr, &addrlen);
           if (conn_sock == -1) {
               perror("accept");
               exit(EXIT_FAILURE);
           }
           setnonblocking(conn_sock);
           ev.events = EPOLLIN | EPOLLET;
           ev.data.fd = conn_sock;
           if (epoll_ctl(epollfd, EPOLL_CTL_ADD, conn_sock, 
                       &ev) == -1) {
               perror("epoll_ctl: conn_sock");
               exit(EXIT_FAILURE);
           }
       } else if(event[n].events & EPOLLIN) {
           do_use_fd(events[n].data.fd);
       }
   }
}
```

红黑树、就绪链表

mmap



**Q:水平触发与边缘触发区别？**

> libevent 采用水平触发， nginx 采用边沿触发
>
>
> 直观感受，缓存区有数据，只通知一次；水平触发，缓存区数据没有取完，会一直触发。






**参考**

[Linux网络高并发技术之epoll](https://www.bilibili.com/video/BV1Yt4y1i7hf?t=35)

[深入理解 Epoll](https://zhuanlan.zhihu.com/p/93609693)

[源码剖析](https://github.com/Liu-YT/IO-Multiplexing)

[IO多路复用select/poll/epoll介绍](https://www.bilibili.com/video/BV1qJ411w7du/?spm_id_from=333.788.videocard.2)





