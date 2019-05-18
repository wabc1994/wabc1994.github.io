---
layout: post
title:  "Disruptor源码阅读之Sequence类"
date:   2018-10-01 8:50:9
categories: Java
comments: true
tags:
    - Java    
    - 源码阅读
    - 并发无锁队列
    - Disruptor
---

* content
{:toc}

# sequence

**ringbuffer与sequence**
>一个Ringbuffer里面是有多个Event组成的， Event里面存储数据，然后每个Event采用sequence 来标记下标



[![ALdDbR.md.png](https://s2.ax1x.com/2019/04/13/ALdDbR.md.png)](https://imgchr.com/i/ALdDbR)

1.在这里主要是接伪共享的问题， 通过左右填充的问题

在回答sequence这个点的时候

1. volatile 修改value，value代表的序号的index
2. 原子Unsafe类  
  - CAS操作
  - 获取对象的偏移地址情况
3. 对象操作

```java
//返回对象成员属性在内存地址相对于此对象的内存地址的偏移量public native long objectFieldOffset(Field 
```

4. 左右填充解决伪共享问题
```java
class LhsPadding
{
    protected long p1, p2, p3, p4, p5, p6, p7;
}
class Value extends LhsPadding
{
    protected volatile long value;
}
class RhsPadding extends Value
{
    protected long p9, p10, p11, p12, p13, p14, p15;
}
```

5. sequence 源码

使用静态代码块获取一个unsafe对象

```java

public class Sequence extends RhsPadding
{
    static final long INITIAL_VALUE = -1L;
    // unsafe类对象
    private static final Unsafe UNSAFE;
    // 一个ringbuffer里面第一个sequence里面的内存地址偏移情况
    private static final long VALUE_OFFSET;
    // 静态代码块获取一个unsafe类对象
    static
    {
        UNSAFE = Util.getUnsafe();
        try
        {
            VALUE_OFFSET = UNSAFE.objectFieldOffset(Value.class.getDeclaredField("value"));
        }
        catch (final Exception e)
        {
            throw new RuntimeException(e);
        }
    }
    //默认构造函数，初始化为-1
    public Sequence()
    {
        this(INITIAL_VALUE);
    } 
    // 其他构造函数的情况
    public Sequence(final long initialValue)
    {
        UNSAFE.putOrderedLong(this, VALUE_OFFSET, initialValue);
    }
    // 获取该序列代表的value
    public long get()
    {
        return value;
    }
     // 通过unsafe类操作获改变该序号的值
    public void set(final long value)
    {
        UNSAFE.putOrderedLong(this, VALUE_OFFSET, value);
    } 
    //多线程环境下对一个sequence的采用CAS操作，
    
     public boolean compareAndSet(final long expectedValue, final long newValue)
    {
        return UNSAFE.compareAndSwapLong(this, VALUE_OFFSET, expectedValue, newValue);
    }
    // Sequence增加一操作
    public long incrementAndGet()
    {
        return addAndGet(1L);
    }
    //将当期sequence增加 increment  采用循环的方式自增情况
    public long addAndGet(final long increment)
    {
        long currentValue;
        long newValue;
        do
        {
            currentValue = get();
            newValue = currentValue + increment;
        }
        while (!compareAndSet(currentValue, newValue));

        return newValue;
    }
 ```

