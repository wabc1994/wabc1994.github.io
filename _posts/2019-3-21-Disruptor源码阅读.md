
---
layout: post
title:  "Ringbuffer数据结构"
date:   2018-7-15 8:50:9
categories: 源码阅读
comments: true
tags:
    - Java
    - Disruptor
---
* content
{:toc}

# Ringbuffer核心数据结构讲解

在ringbuffer之前有两个类， ringbuffer是存储和更新时间的容器

1. RingBufferPad

```java
abstract class RingBufferPad
{
    protected long p1, p2, p3, p4, p5, p6, p7；
}
```

2. RingBufferFields
**RingBufferFields底层数组**

   ```java
abstract class RingBufferFields<E> extends RingBufferPad
{
    private static final int BUFFER_PAD;
    private static final long REF_ARRAY_BASE;
    private static final int REF_ELEMENT_SHIFT;
    
    // 包含一个unsafe类对象情况
    private static final Unsafe UNSAFE = Util.getUnsafe();
    //indexMask = buffersize-1;
  
    private final long indexMask;
    
    //entries 数组存储真正的sequence
    private final Object[] entries;
    
    // ringbuffer数组大小
    protected final int bufferSize;
   
    protected final Sequencer sequencer;    
    this.bufferSize = sequencer.getBufferSize();
    }
    
    // 往这里面填充东西 数组里面 
     private void fill(EventFactory<E> eventFactory)
    {
        for (int i = 0; i < bufferSize; i++)
        {
        // 内存预分配原则，一次分配的基本情况
            entries[BUFFER_PAD + i] = eventFactory.newInstance();
        }
    } 
   ```
3. Ringbuffer 源码

构造函数
	- 确定生产类型,多生产者还是单生产者类型
	- buffersize 数组大小
	- 等待策略
	- EventFactory工厂接口
 
 
 **EventFactory **
 
 一般都是用户自定实现这个接口，定义任务接口，这个ringbuffer里面存储的是什么类型的数据情况
 
>event implementation storing the data for sharing during exchange or parallel coordination of an event.
 
该接口是由用户来定义的，确保内存要分配好的情况
 
 ```java
 public interface EventFactory<T>
 {
 // 直接newInstance一个实例出来的情况， 创建一个对象自定义event对象
		T newInstance();
 }
 ```
 
 
EventFactorys是用来生产Event的，然后Event就是Disruptor当中传输的事件
 

**RingBuffer的主要代码**
 
   ```java
   
   public final class RingBuffer<E> extends RingBufferFields<E> implements Cursored, EventSequencer<E>, EventSink<E>
{
    public static final long INITIAL_CURSOR_VALUE = Sequence.INITIAL_VALUE;
    protected long p1, p2, p3, p4, p5, p6, p7;

 public static <E> RingBuffer<E> create(
        ProducerType producerType,
        EventFactory<E> factory,
        int bufferSize,
        WaitStrategy waitStrategy)
       {
        switch (producerType)
        {
            case SINGLE:
                return createSingleProducer(factory, bufferSize, waitStrategy);
            case MULTI:
                return createMultiProducer(factory, bufferSize, waitStrategy);
            default:
                throw new IllegalStateException(producerType.toString());
        }
    }

  ```

**获取当前数组大小的情况** 

  ```java
 public int getBufferSize()
    {
        return bufferSize;
    }
  ```


**发布当前的事件，当生产者将一个事件填写在sequence里面后，ringbuffer就会将该事件发布，消费者可以进行消费了**

发布就是通知

```java
 @Override
    public void publish(long sequence)
    {
        sequencer.publish(sequence);
    }
```

**获取下一个sequence， 获取下一个可以发布序号**

 - the next sequence to publish to.


```java
public long next()
    {
        return sequencer.next();
    }
```


**批量获取的方式**
 - n number of slots to claim

```java
public long next(int n)
    {
        return sequencer.next(n);
    }
```

