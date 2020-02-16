---
title: Muduo库的框架剖析及总结(一)
date: 2016-08-06 10:58:53
tags:
- Muduo
---

话说回来呢，前段时间又重新看了一遍Muduo库，嗯哼，把原来的框架又大概走了一遍，嗯对，仅仅是框架，还有说到最精华的部分，我想说以前我是发过一篇对Muduo库的线程池的剖析（虽然说剖析的一般吧       =   =！！！，但是毕竟是自己的心血啊）。

<!--more-->

Muduo库的内存池也是Muduo库精华的一部分，当然，还有一个精华的部分我认为是Muduo库的Buffer缓冲区，它运用了环形缓冲区以及vector制动拓长数组的这一个性质，这里觉得它运用的很巧妙有木有，WHY？我想说它这里的环形缓冲区既保持了解决黏包问题缓冲区不够的问题，同时vector又解决了缓冲区自动增长的问题，我想说陈硕在这方面考虑问题简直就是天才！！！

当然还有什么bind,function,shareptr,scoreptr,weakptr这些boost库的东西在Muduo库中也是巧妙的运用了，诶，陈硕简直天才，妈蛋啊，把C++简直运用得炉火纯青，淋漓尽致，呃，感觉说了好多屁话吧，接下来开始进入重点！！！

![muduo](Muduo库的框架剖析及总结(一)/muduo.jpeg)


好了好了，我们可以很明显的从图上看到我分了几个部分对吧

```
TcpServer  
代表的是一个总的服务器的类，我们并不知道服务器内部大概要做什么，我们只需要调用这个TcpServer就可以了，我们可以理解成它是一个服务器入口吧。

TcpConnection
这里是一个连接处理器，当accept接受了一条连接的话就会在这里面进行处理

Acceptor
这里面是对Socket和一个Channel的一个一个封装，我想理解它为一个中间的处理器，当然我们必须把Socket算进来，因为毕竟Acceptor封装的是Socket

Socket 
Socket本身是对socket、listen、bing、accept等服务器函数进行的一个封装

Channel
一个时间处理器，会把相关的对可读可写的事件处理函数回调放入这个类中形成一个对象，就是相当于对事件处理方法封装一下，假如epoll需要处理事件，会从Channel事件处理器中找到相应的方法对事件进行处理

Eventloop  
可以理解成为事件分流处理器，把相关的事件处理器Channel注册金epoll中，而假如epoll有反应则会调用Channle中相应的回调函数，Eventloop底层调用的是poll或者epoll

Poll 
只是一个抽象类，作为对调用的Epoll或者Poll的基类

EPollPoller
Poll的子类，是调用的epoll的一个封装

```

我们得把Muduo库分成四大块，分别是以```Acceptor```为主的服务器块，```TcpConnection```的连接块，```Channel```的事件块以及以```Eventloop```为主的epoll监听块。

我们先从Tcpserver来分析一下吧
看下TcpServer中的成员

```
  EventLoop* loop_;  // the acceptor loop
  const string hostport_;  //端口
  const string name_;  //服务器名字吧
  boost::scoped_ptr<Acceptor> acceptor_; // avoid revealing Acceptor  一个Acceptor对象
  boost::shared_ptr<EventLoopThreadPool> threadPool_; //线程池
  ConnectionCallback connectionCallback_;  //连接回调函数
  MessageCallback messageCallback_;  //连接到信息的回调函数
  WriteCompleteCallback writeCompleteCallback_;
  ThreadInitCallback threadInitCallback_;
  AtomicInt32 started_;
```

当然重要的成员对象我都写在上面了，有了注释。
我们再来看看**TcpServer**的构造函数

```
TcpServer::TcpServer(EventLoop* loop,
                     const InetAddress& listenAddr,
                     const string& nameArg,
                     Option option)
  : loop_(CHECK_NOTNULL(loop)),
    hostport_(listenAddr.toIpPort()),
    name_(nameArg),
    acceptor_(new Acceptor(loop, listenAddr, option == kReusePort)),
    threadPool_(new EventLoopThreadPool(loop)),
    connectionCallback_(defaultConnectionCallback),
    messageCallback_(defaultMessageCallback),
    nextConnId_(1)
{
  acceptor_->setNewConnectionCallback(
      boost::bind(&TcpServer::newConnection, this, _1, _2));
}
```

我们可以看到传进来几个参数，分别是**EventLoop、InetAddress、string、Option**
重点还是**EventLoop**和**InetAddress**，一个是事件分流器，而这个**InetAddress**呢则是对套接字地址**sockaddr_in**的一个封装，我们把端口号和要绑定的IP地址传进去即可。

**EventLoopThreadPool**这家伙是个线程池，至于内部的结构希望看我对**EventLoopThreadPool**分析的那篇文章比较好吧。

**TcpServer**内部封装了一个**Accepror**对象**acceptor_**，当然我们知道它是用**scoped_ptr**对它进行封装的了。

**ConnectionCallback connectionCallback_**的作用是当accept接受到一个新的连接产生一个新的文件描述符而创造了一个通信套接字的时候，调用一下这个**connectionCallback_**回调函数，当然这个函数内容可以你自己设计，可以让你提取回调的**TcpConnection** 对象。

**MessageCallback messageCallback_**的作用是当通信套接字接受到一个新的信息的时候进行回调，当然，这个回调函数的内容也可以你自己去设计。

**ConnectionCallback**和**MessageCallback** 的回调函数的参数都有固定格式，在这里不是重点对象就不说了。

我们可以看到在构造函数中**setNewConnectionCallback**这个回调函数调用了**TcpServer::newConnection**，嗯哼，这是一个什么样的函数呢？我们来看一下

```
void TcpServer::newConnection(int sockfd, const InetAddress& peerAddr)
{
  loop_->assertInLoopThread();
  //从io线程池中提取一条工作线程出来对accept的通信socket进行处理
  EventLoop* ioLoop = threadPool_->getNextLoop();
  char buf[32];
  snprintf(buf, sizeof buf, ":%s#%d", hostport_.c_str(), nextConnId_);
  ++nextConnId_;
  string connName = name_ + buf;

  LOG_INFO << "TcpServer::newConnection [" << name_
           << "] - new connection [" << connName
           << "] from " << peerAddr.toIpPort();
  InetAddress localAddr(sockets::getLocalAddr(sockfd));
  // FIXME poll with zero timeout to double confirm the new connection
  // FIXME use make_shared if necessary
  //我们能从这里看到它实际上是创建了一个TcpConnection，当accept接受到一个新的文件描述符的时候，创建一个TcpConnection对它进行封装
   TcpConnectionPtr conn(new TcpConnection(ioLoop,
                                          connName,
                                          sockfd,
                                          localAddr,
                                          peerAddr));
  //我们可以从这里看出来是对新建立的TcpConnection的一些初始化
  connections_[connName] = conn;
  conn->setConnectionCallback(connectionCallback_);
  conn->setMessageCallback(messageCallback_);
  conn->setWriteCompleteCallback(writeCompleteCallback_);
  conn->setCloseCallback(
      boost::bind(&TcpServer::removeConnection, this, _1)); // FIXME: unsafe
  ioLoop->runInLoop(boost::bind(&TcpConnection::connectEstablished, conn));
}
```

当然，你可以从这里看到了啊，这个函数实际上是对accpet新接受到一个通信套接字以后把它放入到一个新创立的**TcpConnection**对这个通信套接字做一个封装，并将函数指针什么的初始化，诶对咯，它在最后还调用了一个**TcpConnection::connectEstablished**这个函数，后面介绍到**TcpConnection**的时候我们会来介绍一下这个函数的作用，在这里先大概介绍一下这个函数的作用就是将接受到的socket的文件描述符注册到epoll事件列表当中。

当然TcpServer中还有第二个比较重要的函数，这个函数代表了服务器的启动，这个函数当然就是**void TcpServer::start()**了，我们来简单的看一下这个函数的代码

```
void TcpServer::start()
{
  if (started_.getAndSet(1) == 0)
  {
  //对线程池的一个启动处理
    threadPool_->start(threadInitCallback_);

    assert(!acceptor_->listenning());
    loop_->runInLoop(
    //当然这是调用了底层的listen监听咯
    boost::bind(&Acceptor::listen, get_pointer(acceptor_)));
  }
}

```
这个函数启动了，底层的**listen()**函数就相当于启动了，所以我们可以暂时称这是一个服务器的开关