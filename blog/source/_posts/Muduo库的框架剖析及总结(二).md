---
title: Muduo库的框架剖析及总结(二)
date: 2016-08-06 13:41:46
tags:
- Muduo
---

我觉得可能由于我可能要分析完这几个大的模块可能并非易事，要仔细的分析也是绝壁很困难，我写的文章带有很多的口水话，一方面我是希望用通俗的语言去解释这些东西大家能更加好的去理解，另一面可以让我自己更加方便在日后看的时候不至于蒙了

<!--more-->

上次我们已经介绍完了TcpServer这个大类，诶这个大类啊还真是一个非常犀利的一个类呢，小小一个类把服务器改做的事情的接口全部都封装得完美无缺，那么现在我觉得应该介绍一下TcpConnection和Acceptor这两个大类了

我们先从TcpConnection这个大类开始说吧，我们前面说过，我们在Acceptor中设置了一个回调函数调用了TcpServer中的newconnection这个函数，不知道大家还有印象没

```c++
{
  acceptor_->setNewConnectionCallback(
      boost::bind(&TcpServer::newConnection, this, _1, _2));
}
```

当然这里也新建了一个新的**Tcpconnection**了，那我们就进**Tcpconnection**中去看一下
首先还是看它内部的成员变量：

```
  EventLoop* loop_;//事件处理器指针
  const string name_;
  StateE state_;  // FIXME: use atomic variable
  // we don't expose those classes to client.
  boost::scoped_ptr<Socket> socket_;
  boost::scoped_ptr<Channel> channel_;//封装了一个事件处理器
  const InetAddress localAddr_;//端口号什么的
  const InetAddress peerAddr_;
  ConnectionCallback connectionCallback_;//连接回调
  MessageCallback messageCallback_;//信息回调
  WriteCompleteCallback writeCompleteCallback_;
  HighWaterMarkCallback highWaterMarkCallback_;
  CloseCallback closeCallback_;
  size_t highWaterMark_;
  Buffer inputBuffer_;
  Buffer outputBuffer_; // FIXME: use list<Buffer> as output buffer.
  boost::any context_;
```

说句实在话，我觉得这里除了**EventLoop**、**Channel**、**ConnectionCallback**、**MessageCallback** 可以讨论一下，其它信息都跟主框架没有什么直接关系，哦，对了，还有个**Buffer**是起到缓冲区的作用

前面我们有过说在**TcpServer::newConnection**中调用了一个**TcpConnection::connectEstablished**函数，其实现在我们可以先来看一下这个函数：

```
//有连接来的时候创建(顺带调用一下回调函数)
void TcpConnection::connectEstablished()
{
  loop_->assertInLoopThread();
  assert(state_ == kConnecting);
  setState(kConnected);
  channel_->tie(shared_from_this());
  channel_->enableReading();

  connectionCallback_(shared_from_this());
}
```

我们可以看到它中间调用了一个比较重要的函数**channel_->enableReading()**，当然这个函数我们先记着，我暂时先告诉你们它的作用是注册时间，第二个它调用了**connectionCallback_(shared_from_this())**，这个我们也说过，在**TcpServer::newConnection**中它将connectionCallback和MessageCallback都注册了：

```c++
  conn->setConnectionCallback(connectionCallback_);
  conn->setMessageCallback(messageCallback_);
```

当然它又用同样的方式注册Channel事件中的回调函数，当然这个注册的过程在TcpConnection中的构造函数上咯：

```c++
TcpConnection::TcpConnection(EventLoop* loop,
                             const string& nameArg,
                             int sockfd,
                             const InetAddress& localAddr,
                             const InetAddress& peerAddr)
  : loop_(CHECK_NOTNULL(loop)),
    name_(nameArg),
    state_(kConnecting),
    socket_(new Socket(sockfd)),
    channel_(new Channel(loop, sockfd)),
    localAddr_(localAddr),
    peerAddr_(peerAddr),
    highWaterMark_(64*1024*1024)
{
  channel_->setReadCallback(
      boost::bind(&TcpConnection::handleRead, this, _1));
  channel_->setWriteCallback(
      boost::bind(&TcpConnection::handleWrite, this));
  channel_->setCloseCallback(
      boost::bind(&TcpConnection::handleClose, this));
  channel_->setErrorCallback(
      boost::bind(&TcpConnection::handleError, this));
  LOG_DEBUG << "TcpConnection::ctor[" <<  name_ << "] at " << this
            << " fd=" << sockfd;
  socket_->setKeepAlive(true);
}
```

当然我们还知道Tcpconnection中还有好几个send()函数，当然，你要知道它们实际上是重载了，这些send()是回复给客户端用的了，在这里我就不想再具体介绍了

下一个TcpConnection中比较重要的函数就是**TcpConnection::handleRead**了：

```c++
void TcpConnection::handleRead(Timestamp receiveTime)
{
  loop_->assertInLoopThread();
  int savedErrno = 0;
  //此处调用了Buffer回调，这里会直接将readcv中得到的数据放入自己的Buffer中
  ssize_t n = inputBuffer_.readFd(channel_->fd(), &savedErrno);
  if (n > 0)
  {
  //此处调用了信息回调
    messageCallback_(shared_from_this(), &inputBuffer_, receiveTime);
  }
  else if (n == 0)
  {
    handleClose();
  }
  else
  {
    errno = savedErrno;
    LOG_SYSERR << "TcpConnection::handleRead";
    handleError();
  }
}
```

我们前面也看见了在构造函数中，它把**TcpConnection::handleRead**注册到了Channel中的ReadCallback中，至于怎么做哦，后期我们说到了Channel再探讨这个话题。

当然这个函数比较重要的两个地方还是：
1、**ssize_t n = inputBuffer_.readFd(channel_->fd(), &savedErrno);**读取信息
2、**messageCallback_(shared_from_this(), &inputBuffer_, receiveTime);**//这里的这个messageCallback当然是你可以自己编写内容的，就相当于逻辑处理由你自己处理，这个函数自己写，最后写成一个函数指针并用bind回调返回去调用这个你自己写的函数就好了。

------------------------------------------------

好了，对上面的地方我们就不要太过于去纠结了，接下来我们继续往下走，嗯接下来就说说Acceptor吧

呃，我认为还是应该先看看它的成员变量

```c++
  EventLoop* loop_;  //需要指向一个主线程事件分发器
  Socket acceptSocket_;  //这个主要指的是监听socket
  Channel acceptChannel_;  //这个主要指的是将监听事件放入到eventloop中
  NewConnectionCallback newConnectionCallback_;  //创建一个新的连接，就此建立那么一个回调
  bool listenning_;
  int idleFd_;
```

一个Eventloop指针，一个Socket对象，一个Channel事件对象，然后我们前面在TcpServer模块中说过newConnectionCallback_实际上就是TcpServer中的**TcpServer::newConnection**

好吧，我们这个时候再来看一下它的构造函数：

```c++
Acceptor::Acceptor(EventLoop* loop, const InetAddress& listenAddr, bool reuseport)
  : loop_(loop),
    acceptSocket_(sockets::createNonblockingOrDie()),
    acceptChannel_(loop, acceptSocket_.fd()),
    listenning_(false),
    idleFd_(::open("/dev/null", O_RDONLY | O_CLOEXEC))
{
  assert(idleFd_ >= 0);
  acceptSocket_.setReuseAddr(true);
  acceptSocket_.setReusePort(reuseport);//端口重用，多个tcp可以绑定相同的端口
  acceptSocket_.bindAddress(listenAddr);
  acceptChannel_.setReadCallback(
      boost::bind(&Acceptor::handleRead, this));//在有可读的fd的时候调用ReadCallback
}
```

我们可以看到它调用了这么一个函数**acceptSocket_(sockets::createNonblockingOrDie())**，这个**sockets::createNonblockingOrDie()**到底是干嘛用的呢？待会我们介绍socket的时候我们会介绍到，这个还是比较重要的

**acceptChannel_(loop, acceptSocket_.fd())**这个地方我们可以很明显的看出它启用了一个Channel对象，而这个Channel我在这里也可以说是跟监听socket相关

**acceptChannel_.setReadCallback(boost::bind(&Acceptor::handleRead, this));**我们在这里也可以看到handleRead封装进了Acceptor中的Channel中，跟TcpConnection的作法是一样的，这里后期我们再讨论它的重要性

前面我们说过调用newConnectionCallback_实际上就是调用了TcpServer中的**TcpServer::newConnection**，而我们什么时候调用它呢？那么就在我们刚刚所讨论到的HandleRead中

```c++
//对读事件的处理
void Acceptor::handleRead()
{
  loop_->assertInLoopThread();
  InetAddress peerAddr;
  //FIXME loop until no more
  int connfd = acceptSocket_.accept(&peerAddr);
  if (connfd >= 0)
  {
    // string hostport = peerAddr.toIpPort();
    // LOG_TRACE << "Accepts of " << hostport;
    if (newConnectionCallback_)
    {
      //我们可以看到在这里它就调用了创建新的TcpConnection
      newConnectionCallback_(connfd, peerAddr);
    }
    else
    {
      sockets::close(connfd);
    }
  }
  //...中间我省略一些不要的东西
 }
```

这里我们说除了看到它调用newConnectionCallback_外其实还有一个更重要的东西那就是**acceptSocket_.accept(&peerAddr)**，我们可以现在在这里大致的说一下它的底层实际是调用了accept()函数，这个当然是当我们分析到Socket这个类的时候我会说的。

Acceptor还有一个比较重要的函数就是**void Acceptor::listen()**

```c++
//开始调用listen进行监听，并且enableReading注册事件
void Acceptor::listen()
{
  loop_->assertInLoopThread();
  listenning_ = true;
  //底层调用了listen()函数
  acceptSocket_.listen();
  //注册事件到epoll上
  acceptChannel_.enableReading();
}
```

其实我想说Accepor这个类到这里就算分析完了！！！

------------------------------------------------
 
我们接下来继续往下走，这个才是Acceptor的核心，Socket类
其实在前面我们能看到Acceptor中的构造函数调用了这么一个Socket类中的函数：
 **acceptSocket_(sockets::createNonblockingOrDie())**
 
```c++
int sockets::createNonblockingOrDie()
{
#if VALGRIND
	//创建一个监听socket
  int sockfd = ::socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
  if (sockfd < 0)
  {
    LOG_SYSFATAL << "sockets::createNonblockingOrDie";
  }
	//把socket设置成非阻塞模式
  setNonBlockAndCloseOnExec(sockfd);
#else
  int sockfd = ::socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK | SOCK_CLOEXEC, IPPROTO_TCP);
  if (sockfd < 0)
  {
    LOG_SYSFATAL << "sockets::createNonblockingOrDie";
  }
#endif
  return sockfd;
}
```
在这里我们把设置监听socket找到了
还调用了那么一个函数：
 **acceptSocket_.bindAddress(listenAddr)**
 
```c++
void Socket::bindAddress(const InetAddress& addr)
{
  sockets::bindOrDie(sockfd_, addr.getSockAddrInet());
}
```

接着继续往下走

```c++
void sockets::bindOrDie(int sockfd, const struct sockaddr_in& addr)
{
  int ret = ::bind(sockfd, sockaddr_cast(&addr), static_cast<socklen_t>(sizeof addr));
  if (ret < 0)
  {
    LOG_SYSFATAL << "sockets::bindOrDie";
  }
}
```

这里我们把bind()找到了

**Acceptor::listen()**中的listen调用了**acceptSocket_.listen()**

```c++
void Socket::listen()
{
  sockets::listenOrDie(sockfd_);
}
```

继续往下走

```c++
void sockets::listenOrDie(int sockfd)
{
  int ret = ::listen(sockfd, SOMAXCONN);
  if (ret < 0)
  {
    LOG_SYSFATAL << "sockets::listenOrDie";
  }
}
```

这里我们把listen()找到了

从**Acceptor::handleRead()**中我们可以找到**acceptSocket_.accept(&peerAddr)**，而这个函数的底层则是accpet()

```c++
int Socket::accept(InetAddress* peeraddr)
{
  struct sockaddr_in addr;
  bzero(&addr, sizeof addr);
  //找到accept()
  int connfd = sockets::accept(sockfd_, &addr);
  if (connfd >= 0)
  {
    peeraddr->setSockAddrInet(addr);
  }
  return connfd;
}
```
是吧accept()找到了吧！！！
OK，我想说服务器最重要的几个函数我们都找到了
socket()、bind()、listen()、accept()，那么这一块也都算是分析完了吧！！

