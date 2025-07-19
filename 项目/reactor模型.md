# reactor模型

![Edge Image Viewer (camo.githubusercontent.com)](https://camo.githubusercontent.com/c70f82b85fe5c52f6662f70ccd01281dfa8f111051817d9bc57bcf8364a4afb2/68747470733a2f2f63646e2e6e6c61726b2e636f6d2f79757175652f302f323032322f706e672f32363735323037382f313636333332343935353132362d33613830373866652d663237312d346131622d383263372d6237356564666633636461382e706e6723617665726167654875653d25323366346565643126636c69656e7449643d7536656566386535332d383839322d342663726f703d302663726f703d302663726f703d312663726f703d312666726f6d3d70617374652669643d753430613635343462266d617267696e3d2535426f626a6563742532304f626a656374253544266e616d653d696d6167652e706e67266f726967696e4865696768743d343335266f726967696e57696474683d373230266f726967696e616c547970653d75726c26726174696f3d3126726f746174696f6e3d302673686f775469746c653d66616c73652673697a653d333039393838267374617475733d646f6e65267374796c653d6e6f6e65267461736b49643d7563663735376166312d396534322d346335662d396566632d6165613532626138633533267469746c653d)

---

- **mainReactor**

  负责初始化操作，回调函数的注册，维护所有连接

- **acceptor**

  由accepttor负责注册服务器sockfd的读事件，监听新的连接，以及新连接的分派，对应mainLoop，以及一个epollor

- **subReactor**

  每一个subReactor维护一个epollor，每当有事件发生，由subReactor负责执行事件的回调函数

---

~~~cpp
TcpServer::TcpServer(EventLoop *loop,
                     const InetAddress &listenAddr,
                     const std::string &nameArg,
                     Option option)
    : loop_(CheckLoopNotNull(loop)),
    ipPort_(listenAddr.toIpPort()),
    name_(nameArg),
    acceptor_(new Acceptor(loop, listenAddr, option == kReusePort)),
    threadPool_(new EventLoopThreadPool(loop, name_)),
    connectionCallback_(),
    messageCallback_(),
    writeCompleteCallback_(),
    threadInitCallback_(),
    started_(0),
    nextConnId_(1)    
{
    // 当有新用户连接时，Acceptor类中绑定的acceptChannel_会有读事件发生执行handleRead()调用TcpServer::newConnection回调
    acceptor_->setNewConnectionCallback(
        std::bind(&TcpServer::newConnection, this, std::placeholders::_1, std::placeholders::_2));
}
~~~




---

- **TcpServer类-**--------是我们启动服务的入口
  - 维护mainLoop
  
  - 服务器IP地址和端口号
  - 初始化acceptor，并注册有新的连接的时候的回调函数
  - 初始化线程池
  - 维护各种回调函数

---
- 执行start函数，开启线程池，Acceptor开启监听

---




~~~cpp
// 开启服务器监听
void TcpServer::start()
{
    if (started_++ == 0)
    {
        // 启动底层的lopp线程池
        threadPool_->start(threadInitCallback_);
        // acceptor_.get()绑定时候需要地址
        loop_->runInLoop(std::bind(&Acceptor::listen, acceptor_.get()));
    }
}
~~~
---

- **Acceptor类**--------监听连接，执行新连接到来时的回调函数
  
  - 初始化服务器的socketfd，设置服务器的socket地址复用，端口复用，绑定ip和port
  
  - 将socketfd封装到channel，并注册到mainLoop对应的epollor中
  
  - listenning_初试化为false，等待tcpServer执行start函数
  
    

---

~~~cpp
Acceptor::Acceptor(EventLoop *loop, const InetAddress &ListenAddr, bool reuseport) 
    : loop_(loop),
    acceptSocket_(createNonblocking()),
    acceptChannel_(loop, acceptSocket_.fd()),
    listenning_(false)
{
    LOG_DEBUG << "Acceptor create nonblocking socket, [fd = " << acceptChannel_.fd() << "]";
    acceptSocket_.setReuseAddr(reuseport);
    acceptSocket_.setReusePort(true);
    acceptSocket_.bindAddress(ListenAddr);

    /**
     * TcpServer::start() => Acceptor.listen
     * 有新用户的连接，需要执行一个回调函数
     * 因此向封装了acceptSocket_的channel注册回调函数
     */
    acceptChannel_.setReadCallback(
            std::bind(&Acceptor::handleRead, this));   
}
~~~

---

- **监听到有新的连接**

  - 执行handleRead函数，实际是执行socket的accept函数，返回值是客户端socketfd

  - 确认确实是新的连接，执行TcpServer注册的连接回调函数

- 负责新连接的回调函数
	- 通过轮询选择出负责此连接的subLoop
	- new一个TcpConnection对象，负责维护连接的信息，然后把这个对象加入到TcpServer中维护
	- TcpConnection对象是对客户端的fd负责生成channel，并注册感兴趣的事件，也就是加入到subLoop对应的epollor中

---

~~~cpp
// 有一个新用户连接，acceptor会执行这个回调操作，负责将mainLoop接收到的请求连接(acceptChannel_会有读事件发生)通过回调轮询分发给subLoop去处理
void TcpServer::newConnection(int sockfd, const InetAddress &peerAddr)
{
    // 轮询算法 选择一个subLoop 来管理connfd对应的channel
    EventLoop *ioLoop = threadPool_->getNextLoop();
    // 提示信息
    char buf[64] = {0};
    snprintf(buf, sizeof buf, "-%s#%d", ipPort_.c_str(), nextConnId_);
    // 这里没有设置为原子类是因为其只在mainloop中执行 不涉及线程安全问题
    ++nextConnId_;  
    // 新连接名字
    std::string connName = name_ + buf;

    LOG_INFO << "TcpServer::newConnection [" << name_.c_str() << "] - new connection [" << connName.c_str() << "] from " << peerAddr.toIpPort().c_str();
    
    // 通过sockfd获取其绑定的本机的ip地址和端口信息
    sockaddr_in local;
    ::memset(&local, 0, sizeof(local));
    socklen_t addrlen = sizeof(local);
    if(::getsockname(sockfd, (sockaddr *)&local, &addrlen) < 0)
    {
        LOG_ERROR << "sockets::getLocalAddr() failed";
    }

    InetAddress localAddr(local);
    TcpConnectionPtr conn(new TcpConnection(ioLoop,
                                            connName,
                                            sockfd,
                                            localAddr,
                                            peerAddr));
    connections_[connName] = conn;
    // 下面的回调都是用户设置给TcpServer => TcpConnection的，至于Channel绑定的则是TcpConnection设置的四个，handleRead,handleWrite... 这下面的回调用于handlexxx函数中
    conn->setConnectionCallback(connectionCallback_);
    conn->setMessageCallback(messageCallback_);
    conn->setWriteCompleteCallback(writeCompleteCallback_);

    // 设置了如何关闭连接的回调
    conn->setCloseCallback(
        std::bind(&TcpServer::removeConnection, this, std::placeholders::_1));

    ioLoop->runInLoop(
        std::bind(&TcpConnection::connectEstablished, conn));
}
~~~

---

- **channel类**
  - 主要作用是作为事件传入epoll，当事件发生的时候执行对应的回调函数
  - 维护所属subLoop信息，客户端fd，感兴趣的事件，epollor返回的事件类型，是否已经加入epollor 

---

~~~
Channel::Channel(EventLoop *loop, int fd)
    :   loop_(loop),
        fd_(fd),
        events_(0),
        revents_(0),
        index_(-1),
        tied_(false)
{
}
~~~

---

- **InetAddress类**
  - 封装ip和port
  - 维护一个sockaddr_in addr_;

---

---

- **Socket类**
  - 封装服务器的fd，以及对serverfd的一些列操作
  - listen，accept操作，（前面的操作放到Acceptor中了）
  - 是否开启地址复用、端口复用、长连接等

---

---

- **epollor类**
  - 由所属loop引用
  - 初始化epollfd
  - epoll_create1，epoll_wait，epoll_ctl
  - 负责监听事件，并将事件传递给loop

---

~~~cpp
// TODO:epoll_create1(EPOLL_CLOEXEC)
EPollPoller::EPollPoller(EventLoop *loop) :
        Poller(loop), // 传给基类
        epollfd_(::epoll_create1(EPOLL_CLOEXEC)),
        events_(kInitEventListSize)
{
    if (epollfd_ < 0)
    {
        LOG_FATAL << "epoll_create() error:" << errno;
    }
}
~~~

---

- **eventLoop类**
  - 主要作用是处理对应的epollor收集到的发生的事件
  - 也会收到Acceptor发来的新的连接
  - eventfd可以用来通知事件到来唤醒线程，主要是通知mainloop往subLoop中添加事件后需要通知subLoop中执行
  - 同时也用来实现线程安全
  - timerQueue_用来存放定时器

---

![Edge Image Viewer (camo.githubusercontent.com)](https://camo.githubusercontent.com/f79b4031473a08e78ee2e13f8796e9b5aa875361a7cc38b5f89315a84f8e4055/68747470733a2f2f63646e2e6e6c61726b2e636f6d2f79757175652f302f323032322f706e672f32363735323037382f313635383536323332303138372d39346333316331612d376462382d343264662d386536622d6438383031376362616632322e706e6723617665726167654875653d25323338633835373026636c69656e7449643d7563636666333565312d306334652d342663726f703d302663726f703d302663726f703d312663726f703d312666726f6d3d7061737465266865696768743d3430362669643d4432755346266d617267696e3d2535426f626a6563742532304f626a656374253544266e616d653d31363538353632333038253238312532392e706e67266f726967696e4865696768743d353038266f726967696e57696474683d31343730266f726967696e616c547970653d62696e61727926726174696f3d3126726f746174696f6e3d302673686f775469746c653d66616c73652673697a653d3630313530267374617475733d646f6e65267374796c653d6e6f6e65267461736b49643d7561306662346239352d386334622d343237332d386161662d3062653334323934646666267469746c653d2677696474683d31313736)

~~~cpp
EventLoop::EventLoop() : 
    looping_(false),
    quit_(false),
    callingPendingFunctors_(false),
    threadId_(CurrentThread::tid()),
    poller_(Poller::newDefaultPoller(this)),
    timerQueue_(new TimerQueue(this)),
    wakeupFd_(createEventfd()),
    wakeupChannel_(new Channel(this, wakeupFd_)),
    currentActiveChannel_(nullptr)
{
    LOG_DEBUG << "EventLoop created " << this << " the index is " << threadId_;
    LOG_DEBUG << "EventLoop created wakeupFd " << wakeupChannel_->fd();
    if (t_loopInThisThread)
    {
        LOG_FATAL << "Another EventLoop" << t_loopInThisThread << " exists in this thread " << threadId_;
    }
    else
    {
        t_loopInThisThread = this;
    }

    // 设置wakeupfd的事件类型以及发生事件的回调函数
    wakeupChannel_->setReadCallback(std::bind(&EventLoop::handleRead, this));
    // 每一个EventLoop都将监听wakeupChannel的EPOLLIN事件
    wakeupChannel_->enableReading();
}
~~~

---

- **buffer类**
  - 每个TcpConnection维护两个buffer
  - 输入buffer负责读取socketfd，如果一次能读完，直接读入buffer，否则，申请一个临时缓冲区，使用readv同时将数据读到buffer和临时缓冲区中，然后将临时缓冲区的内容copy到buffer中
  - 输出buffer负责输出到socketfd，需要注意能否一次将数据输出，如果能，输出后将写事件注销，否则，因为写事件一直在，会再次调用写事件的回调函数处理剩余数据

---

![Edge Image Viewer (camo.githubusercontent.com)](https://camo.githubusercontent.com/53de8bd0b65e9399a5260148fb698762e7093b85fd3e7aa7b99ae3bab5998b63/68747470733a2f2f63646e2e6e6c61726b2e636f6d2f79757175652f302f323032322f706e672f32363735323037382f313636333439313934303636352d38313037643265632d616663342d346132342d626163632d6338653633666462333263652e706e6723617665726167654875653d25323339346264636526636c69656e7449643d7536663731383133342d363634362d342663726f703d302663726f703d302663726f703d312663726f703d312666726f6d3d7061737465266865696768743d3539302669643d756332386132363631266d617267696e3d2535426f626a6563742532304f626a656374253544266e616d653d31363633343931393337253238312532392e706e67266f726967696e4865696768743d373337266f726967696e57696474683d31313934266f726967696e616c547970653d62696e61727926726174696f3d3126726f746174696f6e3d302673686f775469746c653d66616c73652673697a653d3634353837267374617475733d646f6e65267374796c653d6e6f6e65267461736b49643d7564373338613630312d326634332d343161382d613566372d3161353234656331663433267469746c653d2677696474683d3935352e32)

![Edge Image Viewer (camo.githubusercontent.com)](https://camo.githubusercontent.com/0f89643e4b225432ff71e85d5b4597e359c4be91a7d93b6e7ff71094496900f3/68747470733a2f2f63646e2e6e6c61726b2e636f6d2f79757175652f302f323032322f706e672f32363735323037382f313636333439333631313137352d32643739663265652d313238322d343766332d613739642d6363623239346161383431332e706e6723617665726167654875653d25323363616533666426636c69656e7449643d7536663731383133342d363634362d342663726f703d302663726f703d302663726f703d312663726f703d312666726f6d3d7061737465266865696768743d3235352669643d5463484630266d617267696e3d2535426f626a6563742532304f626a656374253544266e616d653d343230353662613538656366663766633062393231663735316137376462662e706e67266f726967696e4865696768743d333139266f726967696e57696474683d393231266f726967696e616c547970653d62696e61727926726174696f3d3126726f746174696f6e3d302673686f775469746c653d66616c73652673697a653d3137323030267374617475733d646f6e65267374796c653d6e6f6e65267461736b49643d7532323366633961392d313332632d346431332d623766612d3134336435623834333337267469746c653d2677696474683d3733362e38)

![Edge Image Viewer (camo.githubusercontent.com)](https://camo.githubusercontent.com/aa33e043e75f0094dd58a89fa6d787cbe99cf664cedf74168b26ede9ad5e9413/68747470733a2f2f63646e2e6e6c61726b2e636f6d2f79757175652f302f323032322f706e672f32363735323037382f313636333439333631353639382d66633033356361352d623535612d343933302d626334372d3965383839303937393863612e706e6723617665726167654875653d25323363616534666526636c69656e7449643d7536663731383133342d363634362d342663726f703d302663726f703d302663726f703d312663726f703d312666726f6d3d7061737465266865696768743d3236342669643d7672557775266d617267696e3d2535426f626a6563742532304f626a656374253544266e616d653d623238623535393930393561326166303635646633346332353738343738362e706e67266f726967696e4865696768743d333330266f726967696e57696474683d393935266f726967696e616c547970653d62696e61727926726174696f3d3126726f746174696f6e3d302673686f775469746c653d66616c73652673697a653d3137393633267374617475733d646f6e65267374796c653d6e6f6e65267461736b49643d7565633339323262322d643962392d343961352d383865662d3661353765346166316363267469746c653d2677696474683d373936)

