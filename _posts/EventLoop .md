# EventLoop 

I/O线程：包含一个EventLoop对象的线程

EventLoop有许多变量，几个bool变量，looping_：是否正在执行loop循环；quit_:是否已经调用quit()函数退出loop循环；eventHandling是否正在处理event事件；callingPendingFunctors是否正在调用pendingFunctors_的函数对象。

一个eventLoop 可以对应多个channel



# Channel 
Channel 是Reactor 结构当中的”事件", 也就是reactor基于事件模型当中的事件源问题

## channel 使用

不单独使用，它常常包含在其他类中（Acceptor、Connector、EventLoop、TimerQueue、TcpConnection）使用。

## 包含
1. 一个文件描述符或者socket描述符 ，实际不拥有，只是建立一种映射关系, I/O 复用类当中还有一个 map<int,channel> 
2. 在该描述符上要进行的回调函数，也就是处理函数

## channel与epoll_event的关系
struct epoll_event event基本关系
event.events = channel->events();
  event.data.ptr = channel;

主要
```
  ReadEventCallback readCallback_;
  EventCallback writeCallback_;
  EventCallback closeCallback_;
  EventCallback errorCallback_;
  EventCallback 定时器处理函数
```

# 参考链接
[muduo::EventLoop分析](https://blog.csdn.net/KangRoger/article/details/47266785)