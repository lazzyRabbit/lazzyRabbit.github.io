---
title: Muduo库的框架剖析及总结(三)
date: 2016-08-06 21:59:56
tags:
- Muduo
---

有些人觉得我的博客口水话很多，嗯，我个人考虑了一下，可能我写东西确实没有那么高大上，口水话很多，但是我个人确实非常喜欢这种看着非常亲切的风格，其实我觉得还好，只要你能读懂我的博客就行！！！有问题和意见可以提，有问题可以相互探讨、留言，甚至在留言板骂我都行，对我个人也是一种鼓励吧，好了还是太多废话了！！！

<!--more-->

我前面有说过把Muduo库分成大致四个模块，那么我们今天把Channel模块和Eventloop这两个模块给搞定吧

Channel我们前面说过了它是一个事件处理器，看看它的对象

```c++
  boost::weak_ptr<void> tie_;
  bool tied_;
  bool eventHandling_;
  bool addedToLoop_;
  //读处理的回调
  ReadEventCallback readCallback_;
  EventCallback writeCallback_;
  EventCallback closeCallback_;
  EventCallback errorCallback_;
```

再看看它的构造函数：

```c++
Channel::Channel(EventLoop* loop, int fd__)
  : loop_(loop),
    fd_(fd__),
    events_(0),
    revents_(0),
    index_(-1),
    logHup_(true),
    tied_(false),
    eventHandling_(false),
    addedToLoop_(false)
{
}
```

我们前面知道Acceptor和TcpConnection都用了Channel的**setReadCallback**，而这个函数呢就是相当于设置了Channel中的ReadBack回调函数，而这个ReadBack函数的作用是在Epoll一旦触发了就绪事件的回调函数

```c++
  void setReadCallback(const ReadEventCallback& cb)
  { readCallback_ = cb; }
```

不过在这里强调的Channel则内的主要函数则不是这个
我们这里第一个应该强调的函数应该是：
**Channel::handleEvent**

```c++
//调用底下的处理事件
void Channel::handleEvent(Timestamp receiveTime)
{
  boost::shared_ptr<void> guard;
  if (tied_)
  {
    guard = tie_.lock();
    if (guard)
    {
      handleEventWithGuard(receiveTime);
    }
  }
  else
  {
    handleEventWithGuard(receiveTime);
  }
}
```

而它的底层调用的是：
**Channel::handleEventWithGuard**

```c++
//对各种事件的一个处理
void Channel::handleEventWithGuard(Timestamp receiveTime)
{
  eventHandling_ = true;
  LOG_TRACE << reventsToString();
  if ((revents_ & POLLHUP) && !(revents_ & POLLIN))
  {
    if (logHup_)
    {
      LOG_WARN << "Channel::handle_event() POLLHUP";
    }
    if (closeCallback_) closeCallback_();
  }

  if (revents_ & POLLNVAL)
  {
    LOG_WARN << "Channel::handle_event() POLLNVAL";
  }

  if (revents_ & (POLLERR | POLLNVAL))
  {
    if (errorCallback_) errorCallback_();
  }
  //读事件处理
  if (revents_ & (POLLIN | POLLPRI | POLLRDHUP))
  {
    if (readCallback_) readCallback_(receiveTime);
  }
  //写事件处理
  if (revents_ & POLLOUT)
  {
    if (writeCallback_) writeCallback_();
  }
  eventHandling_ = false;
}
```

其实从这里我们能看出来这个函数其实是对当有就绪事件发生时判断事件是可读还是可写时间，判断完后调用一个可读事件函数或者一个可写事件函数，这是Channel中的最重要的一个函数，当然还有另一个重要的函数：**void enableReading()**
在前面的TcpConnection中的**TcpConnection::connectEstablished()**我们调用过它，还有前面**Acceptor**中的**Acceptor::listen()**中我们也调用过它，我们这里就直接告诉它的功能了吧，就是起到一个注册事件的功能，我们来看一下它的代码：

```c++
 void enableReading() { events_ |= kReadEvent; update(); }
```
里面有个update()：

```c++
void Channel::update()
{
  addedToLoop_ = true;
  loop_->updateChannel(this);
}
```
我们可以看到它调用了Eventloop中的updateChannel()，虽然我们还没说到Eventloop，但是我们先走进去继续往下看吧，等后面说完Eventloop再来把之前的坑给补上：

```c++
//添加事件
void EventLoop::updateChannel(Channel* channel)
{
  assert(channel->ownerLoop() == this);
  assertInLoopThread();
  poller_->updateChannel(channel);
}
```

我们看到它又调用了一个**poller****底下的updateChannel()**(我想吐槽为什么封装那么多)，我们现在这里先不细说**poller**，因为它是一个抽象类，它会供我们去选择**epoll**或者**poll**，我后面并没有分析poll的部分，而只是分析了epoll，直接进入epoll看就好咯：

```c++
//注册事件
void EPollPoller::updateChannel(Channel* channel)
{
  Poller::assertInLoopThread();
  LOG_TRACE << "fd = " << channel->fd() << " events = " << channel->events();
  const int index = channel->index();
  if (index == kNew || index == kDeleted)
  {
    // a new one, add with EPOLL_CTL_ADD
    int fd = channel->fd();
    if (index == kNew)
    {
      assert(channels_.find(fd) == channels_.end());
      channels_[fd] = channel;
    }
    else // index == kDeleted
    {
      assert(channels_.find(fd) != channels_.end());
      assert(channels_[fd] == channel);
    }
    channel->set_index(kAdded);
    update(EPOLL_CTL_ADD, channel);
  }
  else
  {
    // update existing one with EPOLL_CTL_MOD/DEL
    int fd = channel->fd();
    (void)fd;
    assert(channels_.find(fd) != channels_.end());
    assert(channels_[fd] == channel);
    assert(index == kAdded);
    if (channel->isNoneEvent())
    {
      update(EPOLL_CTL_DEL, channel);
      channel->set_index(kDeleted);
    }
    else
    {
      update(EPOLL_CTL_MOD, channel);
    }
  }
}
```

最后还是调用了一个EPollPoller::update()：

```c++
//注册删除事件核心
void EPollPoller::update(int operation, Channel* channel)
{
  struct epoll_event event;
  bzero(&event, sizeof event);
  event.events = channel->events();
  event.data.ptr = channel;
  int fd = channel->fd();
  //核心
  if (::epoll_ctl(epollfd_, operation, fd, &event) < 0)
  {
    if (operation == EPOLL_CTL_DEL)
    {
      LOG_SYSERR << "epoll_ctl op=" << operation << " fd=" << fd;
    }
    else
    {
      LOG_SYSFATAL << "epoll_ctl op=" << operation << " fd=" << fd;
    }
  }
}
```

当然，在这里你就能看到它实际上调用了**epoll_ctl**，所以无论是注册事件还是删除事件其实底层都是调用了这个函数，啊，这里说得有些深入Eventloop中了，待会说到这个地方，会把这里的一些坑给补上，这里我就要说明一点就是**void enableReading()**是一个注册事件的函数，而它底层调用指向Eventloop中的**updateChannel()**,我们一直往下深入就会发现实际上它调用了**epoll_ctl**

--------------------------------------------------

好吧Channel也说得差不多了，该说Eventloop这个家伙了，所谓的**Eventloop**在这里就类似于一个**事件分流器**，我们把我们所谓的一些事件封装成功能类，比如TcpConnection和Acceptor，当然这些事件有它们的文件描述符，OK！这个时候我们的**Channel**就相当于一个**事件处理器**，它把对于这个事件的一些方法收集在一起，当有就绪事件发生的时候，就会回调这些方法，我们把我们事件的文件描述符传入到Channel中，再将Channel放入到Eventloop中，那么剩下的事情就由Eventloop去处理吧

我们来看一下Eventloop这个类，它实际上是独立于TcpServer而存在，它，当然，我们只能说我们在结构上可以暂时这么说，因为当你建立一个Eventloop是独立创建然后传入指针进入TcpServer中的，而不是TcpServer创建好的，还是老规矩，老看一下它的成员变量：

```c++
bool looping_; /* atomic */
  bool quit_; /* atomic and shared between threads, okay on x86, I guess. */
  bool eventHandling_; /* atomic */
  bool callingPendingFunctors_; /* atomic */
  int64_t iteration_;
  const pid_t threadId_;
  Timestamp pollReturnTime_;//时间戳
  boost::scoped_ptr<Poller> poller_;//底层的poll或者epoll
  boost::scoped_ptr<TimerQueue> timerQueue_;//时间队列

typedef std::vector<Channel*> ChannelList;
  ChannelList activeChannels_;//就绪事件
  Channel* currentActiveChannel_;//当前就绪事件
```

构造函数：

```c++
EventLoop::EventLoop()
  : looping_(false),
    quit_(false),
    eventHandling_(false),
    callingPendingFunctors_(false),
    iteration_(0),
    threadId_(CurrentThread::tid()),
    poller_(Poller::newDefaultPoller(this)),//选择poll或者epoll
    timerQueue_(new TimerQueue(this)),//epoll_wait的时间列表
    wakeupFd_(createEventfd()),//线程间通信需要的fd
    wakeupChannel_(new Channel(this, wakeupFd_)),
    currentActiveChannel_(NULL)
{
	//...貌似应该和ioloop有关，跟主线程和线程之间的响应有关
  wakeupChannel_->setReadCallback(
      boost::bind(&EventLoop::handleRead, this));
  // we are always reading the wakeupfd
  wakeupChannel_->enableReading();
}
```
好吧，其实它的重点是这个：
**poller_(Poller::newDefaultPoller(this))**

```c++
Poller* Poller::newDefaultPoller(EventLoop* loop)
{
  if (::getenv("MUDUO_USE_POLL"))
  {
    return new PollPoller(loop);
  }
  else
  {
    return new EPollPoller(loop);
  }
}

```

之前我也说过，它是对I/O复用，poll和epoll的一个选择，我所选择的是epoll
epoll的构造函数：

```c++
EPollPoller::EPollPoller(EventLoop* loop)
  : Poller(loop),
  //构造一个epoll
    epollfd_(::epoll_create1(EPOLL_CLOEXEC)),
    events_(kInitEventListSize)
```

我们能看到它调用了**epoll_create**，创建一个**epoll事件队列**

我们再转回Eventloop中：
我们要知道Eventloop最重要的一个函数就是**loop()**，我们应该分析的是这个函数：

```c++
void EventLoop::loop()
{
  assert(!looping_);
  assertInLoopThread();
  looping_ = true;
  quit_ = false;  // FIXME: what if someone calls quit() before loop() ?
  LOG_TRACE << "EventLoop " << this << " start looping";

//quit这个其实和线程有点关系，当然你要了解更多详细的请去看陈硕的书= =！！
  while (!quit_)
  {
	  //前面其实我们有看到这就是一个就是就绪事件列表哦
    activeChannels_.clear();//删除事件列表所有元素
    pollReturnTime_ = poller_->poll(kPollTimeMs, &activeChannels_);//!!!最重要的函数(我们可以看到这里它把activeChannels_传了进去)
    ++iteration_;
    if (Logger::logLevel() <= Logger::TRACE)
    {
      printActiveChannels();
    }
    // TODO sort channel by priority
    eventHandling_ = true;
    for (ChannelList::iterator it = activeChannels_.begin();
    //把传进去的activeChannels_的就绪事件提取出来
        it != activeChannels_.end(); ++it)
    {
      currentActiveChannel_ = *it;
	  //根据就绪事件处理事件
      currentActiveChannel_->handleEvent(pollReturnTime_);
    }
    currentActiveChannel_ = NULL;
    eventHandling_ = false;
    doPendingFunctors();
  }

  LOG_TRACE << "EventLoop " << this << " stop looping";
  looping_ = false;
}

```

好吧poll这个函数比较重要，当然它是调用了**EPollPoller::poll()**，好吧来看看：

```c++
Timestamp EPollPoller::poll(int timeoutMs, ChannelList* activeChannels)
{
//我们看到这里调用了epoll_wait，这里当然就是监听就绪事件的文件描述符了
  int numEvents = ::epoll_wait(epollfd_,
                               &*events_.begin(),
                               static_cast<int>(events_.size()),
                               timeoutMs);
  int savedErrno = errno;
  Timestamp now(Timestamp::now());
  if (numEvents > 0)
  {
    LOG_TRACE << numEvents << " events happended";
    //将就绪事件放入就绪队列拉到Eventloop中统一处理
    fillActiveChannels(numEvents, activeChannels);
    if (implicit_cast<size_t>(numEvents) == events_.size())
    {
      events_.resize(events_.size()*2);
    }
  }
  else if (numEvents == 0)
  {
    LOG_TRACE << " nothing happended";
  }
  else
  {
    // error happens, log uncommon ones
    if (savedErrno != EINTR)
    {
      errno = savedErrno;
      LOG_SYSERR << "EPollPoller::poll()";
    }
  }
  return now;
}
```

**fillActiveChannels(numEvents, activeChannels)**这个得看一下：

```c++
//将就绪事件放到activeChannels中
void EPollPoller::fillActiveChannels(int numEvents,
                                     ChannelList* activeChannels) const
{
  assert(implicit_cast<size_t>(numEvents) <= events_.size());
  for (int i = 0; i < numEvents; ++i)
  {
  //将就绪事件取出来放入到就绪事件列表中
    Channel* channel = static_cast<Channel*>(events_[i].data.ptr);
#ifndef NDEBUG
    int fd = channel->fd();
    ChannelMap::const_iterator it = channels_.find(fd);
    assert(it != channels_.end());
    assert(it->second == channel);
#endif
    channel->set_revents(events_[i].events);
    activeChannels->push_back(channel);
  }
}
```

其实可以看到这就是一个提取就绪事件的一个过程，我们把就绪事件放入到activeChannels中，在Eventloop中再把从epoll中提取到的事件提取出来 ，并调用了currentActiveChannel_的handleEvent,而我们在分析Channel的时候我们已经分析了handleEvent的作用，那么实际上这里就相当于有就绪事件然后返回调用它的回调函数了，嗯，就是这么一个过程！！！