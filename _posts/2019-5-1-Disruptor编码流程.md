---
layout: post
title:  "Disruptor编码总体流程"
date:   2018-10-01 8:50:9
categories: Java
comments: true
tags:
    - Disruptor
    - 源码阅读
   
---

* content
{:toc}






# Disruptor工作流程

总体概览Disruptor编程模型的情况


[![EHjgH0.md.png](https://s2.ax1x.com/2019/05/16/EHjgH0.md.png)](https://imgchr.com/i/EHjgH0)

调价消费者监听类
Disruptor 要和消费者关联起来disruptor.handleEventWith(**消费者处理事件** ),可以绑定多个消费者处理

可以把Disruptor看做一个线程池的ThreadPoolExecutors服务，一步一把这个东西给构建出来


## 一Disruptor 组件配置
- 定义event,考虑内存是否等方面的情况
- 定义eventFactory工程， 重写方法newinstance()方法
- 定义consumer消费者，消费者是直接放进入Disruptor构造方法当中的，
  - Disruptor方法，构造方法
  - eventfactory
  - ringbuffer大小
  - exector线程池
  - 生产者类型
  - 消费者从等待策略
 
 ## 生产者组件投递数据
 
 投递数据sendData数据之类，情况,
 
 如何投递数据这一块
 
 在这里面的情况
  ```
  //获取得到下标
  long sequence  =ringBuffer=.next()
  
 // 我们知道Disruptor 采用内存预分配的策略，每一个Disruptor在使用之前都需要预先指定好大小的ringbuffer并且指定这个ringbuffer是存储什么类型的数据木 
 // 比如我们上面所讲解的orderEvent，所有在所有ringbuffe所有的sequence 当中都是一个orderEvent对象了，可能就有的orderEvent 为空而已
 // 所以下一步
  orderEvent envet = ringbuffer.get(sequence)
  event.setValue(data.getLong());
  //然后再有一个提交操作， 
  ringbuffer.publish(sequence) // 发布同一个
  ```
 
 
 ## 思考问题
 为何需要进行next? 这个点看起来有点奇怪
 
 
 

  
  [![EHvRsA.md.png](https://s2.ax1x.com/2019/05/16/EHvRsA.md.png)](https://imgchr.com/i/EHvRsA)
