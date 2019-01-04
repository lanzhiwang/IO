## IO 多路复用

### select

```c
int select(int maxfdp1, fd_set *readset, fd_set *writeset, fd_set *exceptset, const struct timeval *timeout)

```

select的几大缺点：

  1、每次调用select，都需要把fd集合从用户态拷贝到内核态，这个开销在fd很多时会很大

  2、同时每次调用select都需要在内核遍历传递进来的所有fd，这个开销在fd很多时也很大

  3、select支持的文件描述符数量太小了，默认是1024

### poll

```c
int poll ( struct pollfd * fds, unsigned int nfds, int timeout);

struct pollfd {
  int fd;  // 文件描述符
  short events;  // 等待的事件
  short revents;  // 实际发生了的事件
};

```

poll的机制与select类似，与select在本质上没有多大差别，管理多个描述符也是进行轮询，根据描述符的状态进行处理，`但是poll没有最大文件描述符数量的限制`。

poll和select同样存在一个缺点就是，包含大量文件描述符的数组被整体复制于用户态和内核的地址空间之间，而不论这些文件描述符是否就绪，它的开销随着文件描述符数量的增加而线性增大。

### epoll

```c
int epoll_create(int size);
创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大

int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
epoll的事件注册函数，它不同与select()是在监听事件时告诉内核要监听什么类型的事件，
而是在这里先注册要监听的事件类型。

int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
等待事件的产生，类似于select()调用
```
epoll既然是对select和poll的改进，就应该能避免上述的三个缺点。那epoll都是怎么解决的呢？在此之前，我们先看一下epoll和select和poll的调用接口上的不同，select和poll都只提供了一个函数——select或者poll函数。而epoll提供了三个函数，epoll_create,epoll_ctl和epoll_wait，epoll_create是创建一个epoll句柄；epoll_ctl是注册要监听的事件类型；epoll_wait则是等待事件的产生。

对于第一个缺点，epoll的解决方案在epoll_ctl函数中。每次注册新的事件到epoll句柄中时（在epoll_ctl中指定EPOLL_CTL_ADD），会把所有的fd拷贝进内核，而不是在epoll_wait的时候重复拷贝。epoll保证了每个fd在整个过程中只会拷贝一次。

对于第二个缺点，epoll的解决方案不像select或poll一样每次都把current轮流加入fd对应的设备等待队列中，而只在epoll_ctl时把current挂一遍（这一遍必不可少）并为每个fd指定一个回调函数，当设备就绪，唤醒等待队列上的等待者时，就会调用这个回调函数，而这个回调函数会把就绪的fd加入一个就绪链表）。epoll_wait的工作实际上就是在这个就绪链表中查看有没有就绪的fd（利用schedule_timeout()实现睡一会，判断一会的效果，和select实现中的第7步是类似的）。

对于第三个缺点，epoll没有这个限制，它所支持的FD上限是最大可以打开文件的数目，这个数字一般远大于2048,举个例子,在1GB内存的机器上大约是10万左右，具体数目可以cat /proc/sys/fs/file-max察看,一般来说这个数目和系统内存关系很大。

#### 参考

* [select、poll、epoll 比较](http://www.cnblogs.com/Anker/p/3265058.html)
* [select](http://www.cnblogs.com/Anker/archive/2013/08/14/3258674.html)
* [poll](http://www.cnblogs.com/Anker/archive/2013/08/15/3261006.html)
* [epoll](http://www.cnblogs.com/Anker/archive/2013/08/17/3263780.html)
* [epoll 详解](http://www.cnblogs.com/my_life/articles/3968782.html)