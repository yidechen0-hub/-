# socketfd、eventfd、timerfd



## 1. 什么是fd

[Linux](https://so.csdn.net/so/search?q=Linux&spm=1001.2101.3001.7020) 系统中，把一切都看做是文件，当进程打开现有文件或创建新文件时，内核向进程返回一个文件描述符，文件描述符就是内核为了高效管理已被打开的文件所创建的索引，用来指向被打开的文件，所有执行I/O操作的系统调用都会通过文件描述符。

![image](D:/c++八股文/fd.png)

## 2. socketfd

在操作系统内核空间里，实现网络传输功能的结构是sock，基于不同的协议和应用场景，会被泛化为各种类型的xx_sock，它们结合硬件，共同实现了网络传输功能。为了将这部分功能暴露给用户空间的应用程序使用，于是引入了**socket层**，同时将sock嵌入到文件系统的框架里，sock就变成了一个特殊的文件，用户就可以在用户空间使用文件句柄，**也就是socket_fd来操作内核sock的网络传输能力**。

这个`socket_fd`是一个**int类型的数字**。现在回去看`socket`的中文翻译，**套接字**，将它理解为一**套**用于连**接**的数**字**。

## 3. eventfd

### **为什么需要eventfd？**

muduo可以通过设置线程数量来决定开几个EventLoop，一个EventLoop有多个channel。当channel上有事件发生时，就应该在他所属的EventLoop上执行。
比如在一个mainReactor一个subReactor的情况下，mainReactor监听到一个新连接，要把这个新连接派发给subReactor。subReactor可能大部分的时间都在loop(epoll_wait)阻塞的状态，mainReactor不能强把这个新连接塞给subReactor的Poller，这不合理也不安全，这就需要唤醒阻塞的subReactor，并且让它自己来把这个新连接添加到它的Poller上。
**如何确定这个新连接是在它所属的EventLoop上处理的呢？**
one loop per thread，我们可以通过系统中唯一的线程ID来确认他们所属的loop。

### eventfd在muduo中是怎么唤醒阻塞线程的

**eventfd**用于创建一个专门唤醒事件的一个文件描述符。
当有新连接到来时，如果subReactor还在阻塞，不能一直等到subReactor阻塞返回，应该把它唤醒，立刻处理新连接。
**我们可以让subReactor监听一个eventfd，在需要唤醒时往eventfd里写一个字节，这样subReactor就可以从loop(epoll_wait)阻塞状态返回了。**
用传统的pipe也可以实现，但eventfd可以更高效的唤醒，因为他不必管理缓冲区。

### 常用API

~~~cpp
int eventfd(unsigned int initval, int flags);
initval：初始化计数器的值。
flags：可以是以下之一：
	EFD_SEMAPHORE：将 eventfd 当作信号量使用。
	EFD_CLOEXEC：在 exec() 调用后关闭 eventfd。
	EFD_NONBLOCK：非阻塞模式。
~~~

~~~cpp
ssize_t read(int fd, void *buf, size_t count);
ssize_t write(int fd, const void *buf, size_t count);
int close(int fd);
int fcntl(int fd, int cmd, ... /* arg */ );
	cmd：命令，如 F_SETFL。
	arg：文件描述符的标志，如 O_NONBLOCK。
    
~~~



## 4.timerfd

`timerfd` 是 Linux 为用户程序提供的一个定时器接口。这个接口基于文件描述符，通过文件描述符的可读事件进行超时通知，它可以用来实现高精度的定时器，以及与 `I/O` [多路复用](https://so.csdn.net/so/search?q=多路复用&spm=1001.2101.3001.7020)机制（如`select`、`poll`、`epoll`等）进行配合，实现异步事件驱动的程序。

### 优点

1. 高精度：timerfd使用内核提供的高精度计时器，精度通常可以达到纳秒级别，相对于使用系统时间的方式，误差更小，定时器的精度更高。
2. 资源占用低：timerfd使用内核提供的异步I/O机制，可以将定时器超时事件通知给应用程序，而不需要应用程序不断地[轮询](https://so.csdn.net/so/search?q=轮询&spm=1001.2101.3001.7020)，从而避免了大量的CPU资源占用，提高了系统性能。
3. 灵活性强：timerfd可以设置不同的定时器超时时间，并且可以同时管理多个定时器，可以根据应用程序的需要进行灵活配置。
4. 可移植性好：timerfd是Linux内核提供的标准接口，可以在不同的Linux系统上使用，具有很好的可移植性。

### 常用API

~~~cpp
#include <sys/timerfd.h>

int timerfd_create(int clockid, int flags);

int timerfd_settime(int fd, int flags, const struct itimerspec *new_value, struct itimerspec *old_value);

int timerfd_gettime(int fd, struct itimerspec *curr_value);

~~~

***

~~~cpp
timerfd_create(CLOCK_REALTIME, 0); // 使用系统实时时间作为计时基准，阻塞模式。

clockid:
	CLOCK_REALTIME：使用系统实时时钟，可以被修改；
	CLOCK_MONOTONIC：使用系统单调时钟，不能被修改；
flags:
	TFD_NONBLOCK：非阻塞模式，read操作不会阻塞；
	TFD_CLOEXEC：创建的文件描述符将在执行exec时关闭。

~~~

~~~cpp
timerfd_settime(fd, TFD_TIMER_ABSTIME, &new_value, NULL); 
// 计时器文件描述符fd，初始状态new_value，并且采用绝对时间。（0：相对，1：绝对）

fd：   		 timerfd_create函数创建的定时器文件描述符
flags：		 可以设置为0或者TFD_TIMER_ABSTIME，用于指定定时器的绝对时间或者相对时间
new_value：	 是一个指向itimerspec结构体的指针，用于指定新的定时器参数；
old_value：  是一个指向itimerspec结构体的指针，用于获取旧的定时器参数。

~~~

~~~cpp
// timerfd_gettime是一个用于获取timerfd定时器的当前超时时间的系统调用。
// 它可以用于查询定时器的当前状态，以便在需要时进行调整。

curr_value是指向itimerspec结构体的指针，用于存储当前定时器的超时时间和间隔。

~~~

~~~cpp
struct timespec {
    time_t tv_sec;   // seconds
    long   tv_nsec;  // nanoseconds
};

struct itimerspec {
    struct timespec it_interval;  // Interval for periodic timer
    struct timespec it_value;     // Initial expiration
};

~~~

