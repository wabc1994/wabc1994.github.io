---
layout: post
title:  "ThreadLocal源码深度剖析"
date:   2018-10-03 10:17:10
categories: Java
comments: true
tags:
    - Java
    - JDK源码阅读
    - 并发编程
   
---

* content
{:toc}

# ThreadLocal
ThreadLocal有个静态内部类ThreadLocalMap,然后每个Thread对象都有个一个Map对象，map的类型是ThreadLocal.ThreadLocalMap.然后Map当中的Entry 是一个弱引用， key是ThreadLocal 实例, value 是线程对该变量的值，

ThreadLocal出现问题主要是与线程池使用，ThreadLocal的声明周期跟thread绑定在一起，如果线程池里面的线程一直复用的话，对value的引用链就一直存在，但是value又使用不到，就是一个垃圾，导致无法对这部分内存如果在使用后没有正常得进行remove 操作，就导致key为null的Entry里面的value不能被GC收回掉，因此如果线程存在，这部分内存就一直存在的，


ThreadLocal并不是为线程保存对象的副本，它仅仅只起到一个索引的作用。它的主要目的视为每一个线程隔离一个类的实例**牵桥搭线使用，如果Threadlocal这个线断了，被gc了，那么value也就无法使用了** 

 [![VlDPfg.md.png](https://s2.ax1x.com/2019/05/31/VlDPfg.md.png)](https://imgchr.com/i/VlDPfg)

**Current Thread与Object value之间存在一条强引用关系，但是弱引用的ThreadLocal已经被回收了，线程又不可以通过ThreadLocal取到该value,存在关系，但是又使用不到，这么就很尴尬哈哈哈，要是这种情况很多，程序在线程池部门使用不当出现bug，严重情况下直接OOM**

会导致内存泄漏，当线程数量变得很多，然后线程又不能及时销毁的，就会噪声内存溢出OOM。


weakReference<ThreadLocal>代表ThreadLocal是弱引用的情况，

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    // 往ThreadLocal里实际塞入的值
    Object value;
    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

**为什么需要声明为弱引用？**
弱引用的对象活不过下一次垃圾GC之前的, 所以



**threadlocalmap源码remove**


```java
private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
         if (e.get() == key) {
         // 清空就可以啦
            e.clear();
            expungeStaleEntry(i);
            return;
        }
    }
}
```
# 测试
模拟源码

### 不适用remove()方法
- 在线程使用完不使用remove, 清空key为null的Entry,然后value部分的内存就可以释放了

```java
public class ThreadLocalTest {
    static class LocalVariable {
        private Long[] a = new Long[1024*1024];
    }
    final static ThreadPoolExecutor poolExecutor = new ThreadPoolExecutor(5, 5, 1, TimeUnit.MINUTES,
            new LinkedBlockingQueue<>());

    // 一般我们定义threadlocal都是定义为private static

    final static ThreadLocal<LocalVariable> localVariable = new ThreadLocal<LocalVariable>();

    public static void main(String[] args) throws InterruptedException{
        for(int i=0;i<50;i++){
            poolExecutor.execute(new Runnable() {
                @Override
                public void run() {
                    localVariable.set(new LocalVariable());

                    System.out.println("use local varaible");

                    //localVariable.remove();
                }
            });
            Thread.sleep(1000);
        }
        System.out.println("pool execute over");
    }
}
```

- jps查看java的进程情况
- jmap  查看某个特定线程的内存情况
- jconsole 查看某个进程的具体情况，包括内存，cpu,线程 gc 情况
- jstat -gcutil 进程 查看某个进程的gc的情况


这个是不使用remove()方法的执行完毕后的内存情况，接近100M
[![AXkS4f.md.png](https://s2.ax1x.com/2019/04/14/AXkS4f.md.png)](https://imgchr.com/i/AXkS4f)


### 在线程使用后remove()
- 将remove注解去掉，然后我们就有了下面的测试情况

[![AXkADs.md.png](https://s2.ax1x.com/2019/04/14/AXkADs.md.png)](https://imgchr.com/i/AXkADs)

执行完毕后内存占用情况不到55M。

## 使用建议

- 使用者需要手动调用remove函数，删除不再使用的ThreadLocal.
- 还有尽量将ThreadLocal设置成private static的，这样ThreadLocal会尽量和线程本身一起消亡。
- 需要好好了解Thread, ThreadLocal，ThreadLocalMap，Object value之间的关系情况， Thread持有一个ThreadLocalMap,然后ThreadlocalMap通过对ThreadLocal作为key映射得到Object value，是这么个访问逻辑。Threadlocal作为一个简单的索引的
                
