---
title: 对于sentinel的SPI接口模式分析
date: 2018-12-10 23:54:47
tags:
- sentinel
---

由于最近时间特别忙，所以自己写这一篇新的blog思考了很长的时间     

由于自己要写一个简单版本的进程monitor，但是又想去网上去寻找一些新的灵感，所以找到了sentinel这个东西，sentinel本身是阿里的一个开源框架，其实代码本身并不复杂，但是它的框架却相当的灵活，可以自由的加入你所需要做的业务处理，加入框架中，这跟他的SPI链式模式不无关系，所以在这里帮分析一下。

<!--more-->

**在此鸣谢 逅弈 的博文
我只是个人转载而已,都来自同一个作者下的文章，我就转载他的博客就好了，在这里我想说我真的很感谢他，他的博文分析的很详细:**

*https://www.jianshu.com/u/51121bddcd2a*

**也同时鸣谢github中sentinel的开源文档:**

*https://github.com/alibaba/Sentinel/wiki/Sentinel%E5%B7%A5%E4%BD%9C%E4%B8%BB%E6%B5%81%E7%A8%8B*

### 起源

我想说在我们讨论这个话题之前我希望大家先去看一下sentinel的开源文档，对slot的一个定义，这边文章具体的就是说制造slot结构的，slot在sentinel中的概念为插槽。

sentinel的具体流程大概是这样的

SphU或SphO生成一个Entry，这个Entry包含了生成的资源名字以及生成的时间id，然后我们在生成Entry的时候我们生成一个slotchain

slotchain整个一个流程是这样的:

**NodeSelectorSlot-->ClusterBuilderSlot-->StatisticSlot-->FlowSlot-->AuthoritySlot-->DegradeSlot-->SystemSlot**

每一个插槽代表一个功能(具体功能看文档)

我们一个资源通过这个slotchain这么一走下去啊，他会经历过一系列的统计，判断qps，然后最后判断是否允许它本次操作，最后达到限流的这么一个作用，就相当于一系列的过滤器，最后过滤完了以后看是否合格，再允许本次操作，而且中间非常的灵活，就好像我们一个过滤器里面最上层是石头，然后经过细沙，经过活性炭，最后过滤出来的才是干净的水。这其中slot就相当于其中一种介质，然后我们可以根据自己的业务需要，添加自己的业务介质，所以说这个框架虽然看起来很简单，但是却相当的灵活。

### 结构

其实slot的本质很简单，就是一个通过继承搭建起来的(我想说这个继承关系被我老大批的不要不要的，不过后面我自己写的时候也觉得这个结构写起来不太方便)   

整个继承关系是这样的:         
**DefaultProcessorSlotChain —> ProcessorSlotChain —> AbstractLinkedProcessorSlot —> ProcessorSlot**

我们大概来看一下每一个结构的功能     
最基础的子类:       
**ProcessorSlot**      
它是一个基本的接口     
上面的每一层都基本是这个接口的实现


```java
public interface ProcessorSlot<T> {
	// entry调用具体的每个slot的实现
	void entry(Context context, ResourceWrapper resourceWrapper, T param, int count, boolean prioritized,
               Object... args) throws Throwable;
 	
 	// 进入到下一个slot
 	void fireEntry(Context context, ResourceWrapper resourceWrapper, Object obj, int count, boolean prioritized,
                   Object... args) throws Throwable;
	
	// stop调用具体的每个slot的实现
	void exit(Context context, ResourceWrapper resourceWrapper, int count, Object... args);
	
	// 进入到下一个slot 
	void fireExit(Context context, ResourceWrapper resourceWrapper, int count, Object... args);
}
```

这是每一个slot的接口，后期所有的slot实现接口当然都是由这个组成
**AbstractLinkedProcessorSlot**     
对ProcessorSlot的一个基础实现

```java
public abstract class AbstractLinkedProcessorSlot<T> implements ProcessorSlot<T> {
	private AbstractLinkedProcessorSlot<?> next = null;

    @Override
    public void fireEntry(Context context, ResourceWrapper resourceWrapper, Object obj, int count, boolean prioritized, Object... args)
        throws Throwable {
        if (next != null) {
            next.transformEntry(context, resourceWrapper, obj, count, prioritized, args);
        }
    }
	
	// 其实这里我也不知道用这个大概的意思是啥。。。
    @SuppressWarnings("unchecked")
    void transformEntry(Context context, ResourceWrapper resourceWrapper, Object o, int count, boolean prioritized, Object... args)
        throws Throwable {
        T t = (T)o;
        entry(context, resourceWrapper, t, count, prioritized, args);
    }
	
    @Override
    public void fireExit(Context context, ResourceWrapper resourceWrapper, int count, Object... args) {
        if (next != null) {
            next.exit(context, resourceWrapper, count, args);
        }
    }

    public AbstractLinkedProcessorSlot<?> getNext() {
        return next;
    }

    public void setNext(AbstractLinkedProcessorSlot<?> next) {
        this.next = next;
    }
} 
```

**ProcessorSlotChain**      
组成slot的一个基础类，就是对slot本身的一个管理吧，就是比如添加一个slot，是首部添加还是末尾添加

```java
public abstract class ProcessorSlotChain extends AbstractLinkedProcessorSlot<Object> {
	public abstract void addFirst(AbstractLinkedProcessorSlot<?> protocolProcessor);
	public abstract void addLast(AbstractLinkedProcessorSlot<?> protocolProcessor);
}
```

**DefaultProcessorSlotChain**   
对一个链进行管理的默认处理，相当于链头部节点

```java
public class DefaultProcessorSlotChain extends ProcessorSlotChain {
//由于是头部节点，所以需要特殊处理
	AbstractLinkedProcessorSlot<?> first = new AbstractLinkedProcessorSlot<Object>() {

        @Override
        public void entry(Context context, ResourceWrapper resourceWrapper, Object t, int count, boolean prioritized, Object... args)
            throws Throwable {
            super.fireEntry(context, resourceWrapper, t, count, prioritized, args);
        }

        @Override
        public void exit(Context context, ResourceWrapper resourceWrapper, int count, Object... args) {
            super.fireExit(context, resourceWrapper, count, args);
        }

    };
    AbstractLinkedProcessorSlot<?> end = first;
    
    // 添加到队列头部
    @Override
    public void addFirst(AbstractLinkedProcessorSlot<?> protocolProcessor) {
        protocolProcessor.setNext(first.getNext());
        first.setNext(protocolProcessor);
        if (end == first) {
            end = protocolProcessor;
        }
    }
    
    // 添加到队列末尾
    @Override
    public void addLast(AbstractLinkedProcessorSlot<?> protocolProcessor) {
        end.setNext(protocolProcessor);
        end = protocolProcessor;
    }
    
}

```

### 总结

其实总的来说这个结构写起来还是相当复杂的，但是其实没必要，假如说把它做成模版的链式结构可能相对来说更灵活一些，但是可能考虑到比如StatisticSlot要先跑到后面作限流控制再返回到前面的统计节点，所以这么设计。


----------
