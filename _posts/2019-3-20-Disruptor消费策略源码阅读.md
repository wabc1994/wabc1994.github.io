---
layout: post
title:  "Disruptor waitStrategy和消费者如何消费"
date:   2015-02-15 22:14:54
categories: java
tags: 
  - Disruptor
  - 源码阅读
---

* content
{:toc}


# waitStrategy等待策略接口

协调策略主要有两个点
1. 生产者生产速度太快覆盖掉消费者还没来得及消费的序号， 生产者保持每个消费者的序号，比较序号大小关系来解决该问题
2. 生产者速度跟不上消费者的速度，因此需要进行一定程度的等待策略


功能包括：当没有可消费的事件时，根据特定的实现进行等待，有可消费事件时返回可事件序号；有新事件发布时通知等待的SequenceBarrier


只定义了两个方法

- waitfor()
- signalAllwhenBlocking()另一个是

```java
public interface WaitStrategy
{
    /**
     * Wait for the given sequence to be available.  It is possible for this method to return a value
     * less than the sequence number supplied depending on the implementation of the WaitStrategy.  A common
     * use for this is to signal a timeout.  Any EventProcessor that is using a WaitStrategy to get notifications
     * about message becoming available should remember to handle this case.  The {@link BatchEventProcessor} explicitly
     * handles this case and will signal a timeout if required.
     *
     * @param sequence          to be waited on.
     * @param cursor            the main sequence from ringbuffer. Wait/notify strategies will
     *                          need this as it's the only sequence that is also notified upon update.
     * @param dependentSequence on which to wait.
     * @param barrier           the processor is waiting on.
     * @return the sequence that is available which may be greater than the requested sequence.
     * @throws AlertException       if the status of the Disruptor has changed.
     * @throws InterruptedException if the thread is interrupted.
     * @throws TimeoutException if a timeout occurs before waiting completes (not used by some strategies)
     */
    long waitFor(long sequence, Sequence cursor, Sequence dependentSequence, SequenceBarrier barrier)
        throws AlertException, InterruptedException, TimeoutException;

    /**
     * Implementations should signal the waiting {@link EventProcessor}s that the cursor has advanced.
     */
    void signalAllWhenBlocking();
}
```


然后主要有四种等待策略，等待sequence变得可被消费，

1. BlockingWaitStrategy, 使用一个object 当中的synchrnoized锁机制情况
>Blocking strategy that uses a lock and condition variable for {@link EventProcessor}s waiting on a barrier.

2. BusySpinWaitStrategy的，自旋操作情况

>Busy Spin strategy that uses a busy spin loop for {@link com.lmax.disruptor.EventProcessor}s waiting on a barrier.


3. SleepingWaitingStrategy，方法 先自旋然后让出cpu

> Sleeping strategy that initially spins, then uses a Thread.yield(), and
 * eventually sleep (<code>LockSupport.parkNanos(n)</code>) for the minimum
 * number of nanos the OS and JVM will allow while the

4. timeoutBlockingWaitStrategy 跟BlockingwaitStrategy 一样，只不过只等待一定的时间就不再继续等待下去了，
Blocking strategy that uses a lock and condition variable for {@link EventProcessor}s waiting on a barrier.
 * However it will periodically wake up if it has been idle for specified period by throwing a

# SequenceBarrier 调节消费者和生产者直接的接口

>Wait for the given sequence to be available for consumption.

主要是在生成者和消费者之间架设一条桥梁，等待给定的序号可用

SequenceBarrier主要配合情况waitStratregy来使用

```java
public interface SequenceBarrier
	{
	
	long waitFor(long sequence) throws AlertException, InterruptedException, TimeoutException;
	
		 long getCursor();
		 
		 boolean isAlerted();
		 
		 void alert();
		 
		 void clearAlert();
		 
		 void checkAlert() throws AlertException;
	}
```

## 如何使用
一般都是ringbuffer关联一个 waaitstrategy和Sequencebarrier,

