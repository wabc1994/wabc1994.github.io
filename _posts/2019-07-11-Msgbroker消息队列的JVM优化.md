---
layout: post
title:  "Msgbroker消息队列的JVM优化"
date:   2019-07-11 8:50:9
categories: Java
comments: true
tags:   
    - 消息队列
   
---

* content
{:toc}


# 消息队列

消息队列 - - 消息堆积问题 以及这个问题引发出来的gc问题，


为何GC 会影响消息队列的， 主要是因为GC会导致stop the world（stw），继而影响这个消息的投递，

缓存系统

**这里面有一个问题就是你消息队列不是采用持久化到数据库了吗，为何还会造成 java gc问题，主要是msgbroker在消息落盘的同时，将消息缓存在GC内存里面， 主要是靠两个东西**

1. 从两个角度来解决这个问题
	- 单次零时处理方案，
	- 通用架构设计解决方案，永久性，

	

消息堆积的原因
- broker 频数GC
**消息堆积问题**	
订阅端异常场景下的自我保护

解决方案
**自适应投递限流算法**
	
1. Notify，主要面向需要更加安全可靠地交易类场景，无序，推模式1. 
2. RocketQ(MetaQ) 主要面向消息有序的场景，能够提供更大的消息堆积能力


如何没有事务回查机制，就会消息堆积，因为第一阶段就把消息发送到broker上面的数据落盘了


gc的优化主要包括下面几个探索
1. 分代回收的假设
2. **老年对新生代的引用作为GC**
3. old-gen-scanning 优化探索

msgbroker gc 时间的优化主要包括下面几个部分

1. old-gen-scanning ParGCCardsPerStrideChunk参数调优 

2. 消息缓存优化，从JVM 堆内内存linkedHashMap LRU cache 和FIFO cache再到不用JVM管理的 堆外内存情况

3. 防止消息堆积导致投递上下游链路的崩溃的 自适应投递算法，根据consumer端的消费能力来投递, 这里面也是采用限流当中的令牌桶算法，拿到令牌通就投递消息，没有拿到令牌通就不会投递消息的情况 


# msgbroker 发送阶段 

发送消息分两个阶段的情况 
1. 第一个阶段，发送消息后，注册一个spring事务同步器，并且消息是否提交放在一个commited字段里面
2. 第二阶段，事务执行情况发送给broker，同时伴有会查阶段的情况

 producer的事务执行是放在一个事务模板里面，




