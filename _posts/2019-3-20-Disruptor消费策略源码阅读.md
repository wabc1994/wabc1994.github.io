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


## BlockingWaitStrategy

主要是Object自带的锁机制，使用synchronzied 和notify()

```java
**
 * Blocking strategy that uses a lock and condition variable for {@link EventProcessor}s waiting on a barrier.
 * <p>
 * This strategy can be used when throughput and low-latency are not as important as CPU resource.
 */
public final class BlockingWaitStrategy implements WaitStrategy
{
    private final Object mutex = new Object();

    @Override
    public long waitFor(long sequence, Sequence cursorSequence, Sequence dependentSequence, SequenceBarrier barrier)
        throws AlertException, InterruptedException
    {
        long availableSequence;
        if (cursorSequence.get() < sequence)
        {
            synchronized (mutex)
            {
                while (cursorSequence.get() < sequence)
                {
                    barrier.checkAlert();
                    mutex.wait();
                }
            }
        }

        while ((availableSequence = dependentSequence.get()) < sequence)
        {
            barrier.checkAlert();
            ThreadHints.onSpinWait();
        }

        return availableSequence;
    }

    @Override
    public void signalAllWhenBlocking()
    {
        synchronized (mutex)
        {
            mutex.notifyAll();
        }
    }
```


## BusySpinStrategy
```java
import com.lmax.disruptor.util.ThreadHints;

/**
 * Busy Spin strategy that uses a busy spin loop for {@link com.lmax.disruptor.EventProcessor}s waiting on a barrier.
 * <p>
 * This strategy will use CPU resource to avoid syscalls which can introduce latency jitter.  It is best
 * used when threads can be bound to specific CPU cores.
 */
public final class BusySpinWaitStrategy implements WaitStrategy
{
    @Override
    public long waitFor(
        final long sequence, Sequence cursor, final Sequence dependentSequence, final SequenceBarrier barrier)
        throws AlertException, InterruptedException
    {
        long availableSequence;

        while ((availableSequence = dependentSequence.get()) < sequence)
        {
            barrier.checkAlert();
            ThreadHints.onSpinWait();
        }

        return availableSequence;
    }

```
## SleepingWaitingStrategy
主要是使用LockSupport 来阻塞线程, 先有一个尝试尝试获取的次数，如果尝试不成功的话就就进行调用yeild(),让出CPU, 如果实在不行就调用sleep() 睡眠线程 调用LockSupport.park() 阻塞线程

在Java多线程中，当需要阻塞或者唤醒一个线程时，都会使用LockSupport工具类来完成相应的工作。LockSupport定义了一组公共静态方法，这些方法提供了最基本的线程阻塞和唤醒功能，而LockSupport也因此成为了构建同步组件的基础工具

void parkNanos(long nanos)

阻塞当前线程，超时返回，阻塞时间最长不超过nanos纳秒

源码当中对该waitStrategy策略的说明情况


>Sleeping strategy that initially spins, then uses a Thread.yield(), and
eventually sleep (<code>LockSupport.parkNanos(n)</code>) for the minimum
number of nanos the OS and JVM will allow while the EventProcessor are waiting on a barrier.
This strategy is a good compromise between performance and CPU resource.
Latency spikes can occur after quiet periods.  It will also reduce the impact
on the producing thread as it will not need signal any conditional variables
to wake up the event handling thread.

```java
public final class SleepingWaitStrategy implements WaitStrategy
{
    private static final int DEFAULT_RETRIES = 200;
    private static final long DEFAULT_SLEEP = 100;
    private final int retries;
    private final long sleepTimeNs;
    public SleepingWaitStrategy()
    {
        this(DEFAULT_RETRIES, DEFAULT_SLEEP);
    }
    public SleepingWaitStrategy(int retries)
    {
        this(retries, DEFAULT_SLEEP);
    }
    public SleepingWaitStrategy(int retries, long sleepTimeNs)
    {
        this.retries = retries;
        this.sleepTimeNs = sleepTimeNs;
    }
    @Override
    public long waitFor(
        final long sequence, Sequence cursor, final Sequence dependentSequence, final SequenceBarrier barrier)
        throws AlertException
    {
        long availableSequence;
        int counter = retries;

        while ((availableSequence = dependentSequence.get()) < sequence)
        {
            counter = applyWaitMethod(barrier, counter);
        }ong
        return availableSequence;
    } 
    @Override
    public void signalAllWhenBlocking()
    {
    }
    private int applyWaitMethod(final SequenceBarrier barrier, int counter)
        throws AlertException
    {
     // 消费协调者
        barrier.checkAlert();
        if (counter > 100)
        {
            --counter;
        }
        else if (counter > 0)
        {
	// 不成功， 就让出CPU
            --counter;
            Thread.yield();
        }
        else
        {
	// 阻塞睡眠线程
            LockSupport.parkNanos(sleepTimeNs);
        }
        return counter;
    }
}
```

# SequenceBarrier接口 调节消费者和生产者直接的接口

Barrier 就像一个障碍，当没有可消耗的序号时，就阻挡消费者就行消费行为，当
>Wait for the given sequence to be available for consumption.

主要是在生成者和消费者之间架设一条桥梁，等待给定的序号可用，这是一个接口，需要我们进行实现接口

SequenceBarrier主要配合情况waitStratregy来使用

```java
public interface SequenceBarrier
	{
	// 等seqence之前的序号可以用
	long waitFor(long sequence) throws AlertException, InterruptedException, TimeoutException;
	// 下面的操作主要是操作消费者的消费序号的问题
	//getCursor() 是获取生产者最后在Ringbuffer里面publish
	
	//value of the cursor for entries that have been published.
		 long getCursor();
		 // 判断是否满足该种条件的情况
		 boolean isAlerted();
		 // 对消费者的消费需要进行一个修改
		 void alert();
		 //清空该种修改情况
		 void clearAlert();
		 void checkAlert() throws AlertException;
	}
```

## 如何使用
一般都是Ringbuffer关联一个Waitstrategy和Sequencebarrier,  

看下面的图
[![ALTCu9.md.png](https://s2.ax1x.com/2019/04/13/ALTCu9.md.png)](https://imgchr.com/i/ALTCu9)
比如 ConsumerBarrier extends SequenceBarrier 对象——这个对象由RingBuffer创建并且代表消费者与RingBuffer进行交互

```
final long availableSeq = consumerBarrier.waitFor(nextSequence);

```
比如我们如果设置nextSequence为12的话，现在生产者还没生产到12号，那么我们的消费者就要等待，采用上述给出的四个策略当中的某个，


ConsumerBarrier返回RingBuffer的最大可访问序号——在上面的例子中是12。 

当满足条件可以消费后
