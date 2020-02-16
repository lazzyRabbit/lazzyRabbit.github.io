---
title: 利用muduo原线程池修改的可调度线程池源码
date: 2017-09-24 16:14:07
tags:
 - Muduo
---

nothing

<!--more-->

由于muduo的线程池是抢占式的，在锁过多利用的情况下我们需要自己利用自己的规则去分配和调度线程，我这个实现的就是一个可自己分配和调度的线程池，其特点有以下几点

1. 每个线程都有自己的消息队列并且大小不限
2. 每个线程都可自由阻塞，但是消息队列正常接受消息
3. 可利用Run进行开启线程或者启动的时候利用一个列表开启线程池，很方便

```c++
/* 
 * File:   NonPreemptiveThreadPool.h
 * Author: zerber
 *
 * Created on 2017年9月1日, 下午1:27
 */
#pragma once

#include "local_common/Public.h"

#include <muduo/base/Thread.h>
#include <muduo/base/Mutex.h>
#include <muduo/base/Condition.h>
#include <muduo/base/CountDownLatch.h>

#ifndef NONPREEMPTIVETHREADPOOL_H
#define NONPREEMPTIVETHREADPOOL_H

typedef boost::function<void ()> Task;

struct RunThreadNode
{
    RunThreadNode(const std::string& name,int queuesize = 0):
    countDownLatch_(1),
    listMutex_(),
    blockMutex_(),
    notEmpty_(listMutex_),
    blockCond_(blockMutex_),
    isBolck_(false),
    threadName_(name),
    thread_(boost::bind(&RunThreadNode::RunInThread_,this),threadName_),
    running_(false),
    queuesize_(queuesize){}
    ~RunThreadNode()
    {
        if(running_)
        {
            ThreadStop();
        }
    }
    
    void PushTaskInQueue(const Task& task);
    void ThreadStart();
    void ThreadStop();
    const std::string& GetThreadName();
    void ThreadBlock();
    void ThreadWakeUp();
    
    muduo::CountDownLatch countDownLatch_;
private:
    Task Take_();
    void RunInThread_();

    muduo::MutexLock listMutex_;
    muduo::MutexLock blockMutex_;
    muduo::Condition notEmpty_;
    muduo::Condition blockCond_;
    bool isBolck_;
    std::string threadName_;

    muduo::Thread thread_;
    std::queue<Task> queue_;

    bool running_;
    int queuesize_;
};

typedef boost::shared_ptr<RunThreadNode> RunThreadNodePtr;

class NonPreemptiveThreadPool
{
public:
    //指定队列长度，指定生成哪些线程
    NonPreemptiveThreadPool(int queuesize,
            const std::string& name,
            int* threadList,
            int threadListNum);
    ~NonPreemptiveThreadPool();  

    void Run(int threadNum, const Task& f);
    void Stop(int threadNum);
    void AllStart();
    void AllStop();
    void ThreadBlock(int threadNum);
    void ThreadWakeUp(int threadNum);
private:
    
    boost::unordered_map<int, RunThreadNodePtr> threads_;
    int queuesize_;
    std::string name_;
};

#endif /* NONPREEMPTIVETHREADPOOL_H */
```

```c++
/* 
 * File:   NonPreemptiveThreadPool.cpp
 * Author: zerber
 *
 * Created on 2017年9月1日, 下午1:27
 */
#include "NonPreemptiveThreadPool.h"

#include <muduo/base/Exception.h>

void RunThreadNode::PushTaskInQueue(const Task& task)
{
    muduo::MutexLockGuard lock(listMutex_);
    
    queue_.push(task);
    notEmpty_.notify();
}

void RunThreadNode::ThreadStart()
{
    running_ = true;
    thread_.start();
}

void RunThreadNode::ThreadStop()
{
    {
        muduo::MutexLockGuard lock(listMutex_);
        running_ = false;
        notEmpty_.notify();
    }
    thread_.join();
    
    LOG_DEBUG << "thread " << threadName_ << " stop";
}

const std::string& RunThreadNode::GetThreadName()
{
    return threadName_;
}

void RunThreadNode::ThreadBlock()
{
    isBolck_ = true;
}

void RunThreadNode::ThreadWakeUp()
{
    if(isBolck_)
    {
        isBolck_ = false;
        blockCond_.notify();
    }
}

Task RunThreadNode::Take_()
{
    muduo::MutexLockGuard lock(listMutex_);
    while(queue_.empty() && running_)
    {
        notEmpty_.wait();
    }
    
    Task task;
    if(!queue_.empty())
    {
        task = queue_.front();
        queue_.pop();
    }
    return task;
}

void RunThreadNode::RunInThread_()
{
    muduo::MutexLockGuard lock(blockMutex_);
    try
    {
        while (running_)
        {
          Task task(Take_());
          while(isBolck_)
          {
              blockCond_.wait();
          }

          if(task)
          {
            task();
          }
        }
    }
    catch (const muduo::Exception& ex)
    {
      fprintf(stderr, "exception caught in ThreadPool %s\n", threadName_.c_str());
      fprintf(stderr, "reason: %s\n", ex.what());
      fprintf(stderr, "stack trace: %s\n", ex.stackTrace());
      abort();
    }
    catch (const std::exception& ex)
    {
      fprintf(stderr, "exception caught in ThreadPool %s\n", threadName_.c_str());
      fprintf(stderr, "reason: %s\n", ex.what());
      abort();
    }
    catch (...)
    {
      fprintf(stderr, "unknown exception caught in ThreadPool %s\n", threadName_.c_str());
      throw;
    }
}
////////////////////////////////////////////////////////////////////////////////

NonPreemptiveThreadPool::NonPreemptiveThreadPool(int queuesize, 
                                                 const std::string& name,
                                                 int* threadList = NULL, 
                                                 int threadListNum = 0)
:queuesize_(queuesize),
name_(name)        
{
    if(threadList != NULL && threadListNum > 0)
    {
        for(int i=0 ; i<threadListNum ; i++)
        {
            char id[32];
            snprintf(id, sizeof(id), "%d", threadList[i]);
            RunThreadNodePtr pCurRunThreadNode = boost::make_shared<RunThreadNode>(name_+id,queuesize_);
            threads_.insert(make_pair(threadList[i],pCurRunThreadNode));
        }
    }
}

NonPreemptiveThreadPool::~NonPreemptiveThreadPool()
{
    AllStop();
}

void NonPreemptiveThreadPool::Run(int threadNum, const Task& f)
{
    auto iter = threads_.find(threadNum);
    if(iter != threads_.end())
    {
        iter->second->PushTaskInQueue(f);
    }
    else
    {
        char id[32];
        snprintf(id, sizeof(id), "%d", threadNum);
        RunThreadNodePtr pCurRunThreadNode = boost::make_shared<RunThreadNode>(name_+id,queuesize_);
        threads_.insert(make_pair(threadNum,pCurRunThreadNode));
        pCurRunThreadNode->ThreadStart();
        LOG_DEBUG << "thread " << pCurRunThreadNode->GetThreadName() << " start";
    }
}

void NonPreemptiveThreadPool::Stop(int threadNum)
{
    auto iter = threads_.find(threadNum);
    if(iter == threads_.end()) return;
    
    iter->second->ThreadStop();
    LOG_DEBUG << "thread " << iter->second->GetThreadName() << " stop";
}

void NonPreemptiveThreadPool::AllStart()
{
    auto iter = threads_.begin();
    for(; iter!=threads_.end() ;iter++)
    {
        iter->second->ThreadStart();
    }
    LOG_DEBUG << "all threads has been start";
}

void NonPreemptiveThreadPool::AllStop()
{
    auto iter = threads_.begin();
    for(; iter!=threads_.end() ;iter++)
    {
        iter->second->ThreadStop();
    }
    LOG_DEBUG << "all threads has been stop";
}

void NonPreemptiveThreadPool::ThreadBlock(int threadNum)
{
    auto iter = threads_.find(threadNum);
    if(iter == threads_.end()) return;
    iter->second->ThreadBlock();
    LOG_DEBUG << "thread " << iter->second->GetThreadName() << " block";
}

void NonPreemptiveThreadPool::ThreadWakeUp(int threadNum)
{
    auto iter = threads_.find(threadNum);
    if(iter == threads_.end()) return;
    iter->second->ThreadWakeUp();
    LOG_DEBUG << "thread " << iter->second->GetThreadName() << " wake up";
}
```
