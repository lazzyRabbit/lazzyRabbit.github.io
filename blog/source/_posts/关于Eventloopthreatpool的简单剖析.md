---
title: 关于Eventloopthreatpool的简单剖析
date: 2015-11-18 15:05:58
tags:
- Muduo
---

最近对 muduo 库本身的 eventloopthreadpool 做了一个简单的剖析，写一个简单的总结，首先，我觉得简单介绍一下:

eventloopthreadpool 是基于 muduo 库中 Tcpserver 这个类专门做的一个线程池，它的模式属于半同步半异步，每一个线程中都有一个自己的 eventloop，而每一个 eventloop 底层都是一个 poll 或者 epoll，它利用了自身的 poll 或者 epoll 在没有时间的时候阻塞住，在有时间发生的时候，epoll 监听到了时间就会去处理事件。我们来画一下图，模式大概是这样的:

```

(轮询箭头)------>|----------eventloopthread[0]--------Thread(处理)
                |
baseloop------->|-----eventloopthread[1]--------Thread(处理)
(Accept)        |
                |-----eventloopthread[2]--------Thread(处理)

```

好，线程池的结构大概是这样子，说到这个线程池，我们首先不得不说一个玩意，互斥锁，在muduo库中，对互斥锁进行了很好的一个封装，我们来看一下:

```c++
class MutexLockGuard : boost::noncopyable
{
 public:
  explicit MutexLockGuard(MutexLock& mutex)
    : mutex_(mutex)
  {
    mutex_.lock();
  }

  ~MutexLockGuard()
  {
    mutex_.unlock();
  }

 private:

  MutexLock& mutex_;
};
```

这就是我说的互斥锁，这是对它进行的一层封装，

```c++
  void lock()
  {
    MCHECK(pthread_mutex_lock(&mutex_));
    assignHolder();
  }

  void unlock()
  {
    unassignHolder();
    MCHECK(pthread_mutex_unlock(&mutex_));
  }
```

我将它内部的代码展示出来大家可以很清楚的看到这是一个锁的封装了吧，这个锁最大的好处在于它比较智能，当我们创建它的时候，构造函数会自动帮我们生成一个锁，当我们在区域内，创造一个锁的时候，它会锁住，当出了这个区域的时候，锁自动解开。

现在我们来分析一下主要流程:

![eventloopthreadpool的结构图](eventloopthreadpool的结构图.jpeg)

我们主要通过这个图去分析它的整体框架，看这个线程池到底做了个啥事，如何多个线程同时运作的。

我们先分析一下它的类中用到成员变量和每个类的构造函数

```c++
class Condition
{
    MutexLock& mutex_;//锁
    pthread_cond_t pcond_;//条件变量
}
```

```c++
  class EventLoopThreadPool
  {
    EventLoop* baseLoop_; //基于接受主线程socket的eventloop
    bool started_; //线程池是否开始运作
    int numThreads_; //线程的数量
    boost::ptr_vector<EventLoopThread> threads_; //线程池
    std::vector<EventLoop*> loops_; //线程池中的eventloop数组
  }
```

```c++
class EventLoopThread
{
  EventLoop* loop_; //基于接受主线程socket的eventloop
  Thread thread_; //线程
  MutexLock mutex_; //锁
  Condition cond_; //条件变量
  ThreadInitCallback callback_; //外部参数以获得线程信息
}
```

```c++
class Thread
{
  pthread_t  pthreadId_; //线程ID
  ThreadFunc func_; //线程函数
}
```

```c++
struct ThreadData
{
  typedef muduo::Thread::ThreadFunc ThreadFunc;
  ThreadFunc func_; //线程函数
}
```

我们看，它首先是从eventloopthreadpool中的start()这个函数当入口启动这个线程池的，内部有一个for循环，经过这个for循环就创建了一个线程池。

我们看，它创建了一个eventloop对象，并且把这个对象压入了eventloopthread的数组当中，并且提取出了它里面的eventloop压入了loops当中。

继续往下走，看下startloop干了个啥，是不是调用了一个thread.start()啊，底下加了个锁，锁底下的代码段是对主线程阻塞的一个判断，暂时先不理它。

继续往下走，进入thread中的start()，thread中的start()干了一件事，创建了一个ThreadData并且当做参数放入了线程函数当中，这个线程函数又是个啥玩意？是个startThread()对不对，接着进去，诶，有一个名叫runInThread()的函数，接着进入runInThread()，诶，发现了一些神奇的事，话不多说，上代码:

```c++
  void runInThread()
  {
    try
    {
      func_();
      muduo::CurrentThread::t_threadName = "finished";
    }
  }
```

诶嘿，发现没有，这里调用了一个func\_()，很眼熟吧，好像在前面见过，进入ThreadData的构造函数，func\_()是啥，实质上是一个**boost::function< void () >**的函数回调指针。我们发现这里给func\_()赋值,赋值也是外部传进来的，往回走，发现它调用的是Thread中start()的func\_(),Thread中start()的func\_()也是别人构造函数里面赋值的继续往前走，Eventloopthread在创建thread的时候给他赋了一个值，这个值就是 **func\_()**，这个 func\_() 实质上就是 Eventloop 中的 threadFunc()，诶，进去看一下，一个临时变量的Eventloop，先调用了一下 callback，先不理它，看后面，给Eventloop 中的loop_ 赋值，然后调用一个 notify()，往回看， notify()实质上就是一个 **pthread\_cond\_signal(&pcond\_)**，而我们之前Eventloop中看到有一个成员 Condition cond\_ 这实际上就是对条件变量的一个封装。

此处有一个**pthread\_cond\_signal**，它要唤醒一个线程，唤醒哪个线程，诶这个不清楚，记得前面有一个cond\_.wait() 对不对，它要唤醒这个cond\_.wait()在startloop中，刚开始不知道它是干嘛的，现在知道了，当我们子线程创建的时候，它要创建一个eventloop啊，在还没保证它创建之前，我们的主线程不能继续往下创建新的线程，必须要等这个eventloop创建完成才能继续往下创建，因为eventloop才是这每一个线程的核心啊，当eventloop没创建完成时，我们会让其锁住，并让它成为阻塞状态，等到eventloop创建完成时，才会让主线程继续运行，继续创建下一个。

往下走你会发现它调用了**loop.loop()**，实质上调用了 poll 或者epoll ，利用其自身的阻塞性将线程给阻塞住了。

当我们完成这一步以后，我们还得回归一下前面一个地方，值得注意的地方就是在eventloopthreadpool中，它把每一个eventloop都提取出来放到一个数组loops中了，这样我们在提取它的loop是更方便了不是吗？

看到这我们大概会明白它的创建机制了吧。

创建好了要将其使用，咋用，话不多说上代码:

```c++
void TcpServer::newConnection(int sockfd, const InetAddress& peerAddr)
{  
 EventLoop* ioLoop = threadPool_->getNextLoop();
 TcpConnectionPtr conn(new TcpConnection(ioLoop,connName,sockfd,localAddr,peerAddr);

  connections_[connName] = conn;
  conn->setConnectionCallback(connectionCallback_);
  conn->setMessageCallback(messageCallback_);
  conn->setWriteCompleteCallback(writeCompleteCallback_);
  conn->setCloseCallback(
      boost::bind(&TcpServer::removeConnection, this, _1)); // FIXME: unsafe
  ioLoop->runInLoop(boost::bind(&TcpConnection::connectEstablished, conn));
 }
```

这是TcpServer中的TcpServer::newConnection()，调用了**threadPool_->getNextLoop()** ，当我们分析过muduo库我们会知道，当TcpServer中的主线程每收到一个新的socket的时候，我们就将socket时间放到子线程中的eventloop中去处理，也就是说不同的子线程中的eventloop会有不同的socket，当我们不同的socket接受到消息的时候会在不同的子线程中同时处理，也就是说这样就大大提高了效率。

我们进**threadPool_->getNextLoop()** 中看看，我们会发现，它是轮询的调用eventloop的，也就是说主线程分配的socket是轮询进入不同的eventloop中的。

**最后再总结一下：**

```
1、eventloopthreadpool是一个半同步半异步的机制
2、eventloopthreadpool实际上是其中包含了一个eventloopthread的数组，而每一个eventloopthread对象实际上都管理着一个Thread，而每一个Thread对象实际上就是对一个线程的封装
3、每一个eventloopthread对应一个线程，每一个eventloopthread又包含一个eventloop，也就是相当于一个线程对应一个eventloop，在不同的线程操作着不同的eventloop，实际上就是同时操作不同的poll或者epoll，效率大大的提高了
4、当我们从主线程的eventloop中的Accept中listen到一个新的socket后轮询的发给每一个eventloop中，交由不同的eventloop去处理
5、它实质上的线程函数是void EventLoopThread::threadFunc()
6、值得注意的一点是eventloop在每个线程中的loop还没有创建出来之前会阻塞住主线程，而在loop创建出来之后主线程释放，运行到下面loop.loop()后，会利用poll或者epoll自身的阻塞将线程阻塞住并进入循环接受消息的状态(PS:线程池每一条线程在不用的时候自身都必须进入阻塞状态，否则线程运行完就会释放了，线程池也就没什么意义了)，下面的loop_ = NULL;也就不会执行了。
```

好了，大概就是总结了那么多。。。

