---
layout: post
title:  "Service Mesher"
date:   2019-03-25 22:14:54
categories: 微服务
comments: true
tags:
    - 服务网格
    - 为服务器
---

* content
{:toc}

# service mesh

最近一直想关注下比较前沿的后端技术，所以就打算学习一下比较火的service mesh 服务网格技术。


**前言**
新一代的网络通信技术，不同应用之间的通信如何做，底层的，如何将不同service 之间通信服务模块抽象出来，让应用更加专注于业务实现即可，而无需关系与依赖模块之间的相互通信。

[![VMc6TP.md.jpg](https://s2.ax1x.com/2019/05/30/VMc6TP.md.jpg)](https://imgchr.com/i/VMc6TP)

## 背景 

微服务分层处理，各个组件组合在一起就是很复杂，容易出现问题，特别是在承载大量服务的云原生环境下，各种服务互相之间的调用形成错综复杂的关系，如何这种复杂的环境构建一个高效，健壮高可用的系统就成为一个头疼的问题， 比如微服务系支持熔断，降级，服务治疗等这是肯定要支持的，因此服务网格也被称为微服务2.0吧，

[![E2BiZV.md.jpg](https://s2.ax1x.com/2019/05/10/E2BiZV.md.jpg)](https://imgchr.com/i/E2BiZV)

## 定义


先来看看英文定义
> A service mesh is a dedicated infrastructure layer for handling service-to-service communication. It is responsible for the reliable delivery of requests through the complex topology of services that comprise a modern, cloud native application. In practise, the service mesh is typically implemented as an array of lightweight network proxies that are deployed alongside application code, without the application needing to be aware.


**中文自己的理解**
将业务层与通信逻辑层分离，是一个基础服务通信模块，用于处理服务通信，云原生应用有着复杂的拓扑结构关系，服务网格负责在这些拓扑中实现请求的可靠传递，在实践当中，**服务网格通常实现为一组轻量级网络代理**，他们与**应用程序**部署在一起，而对应用程序透明。可以说是一种轻量级的网络代理，并且这种代理。



## cloud native
**什么叫做cloud native application**
>企业采用基于Cloud Native的技术和管理方法，可以更好的把业务迁移到云平台，从而享受云的高效和按需资源能力，云原生是以云架构为优先的应用开发模型, 云原生并不是一种具体的技术，而是一类思想的集合，包括DevOps、持续交付、微服务、敏捷基础设施、康威定律。

cloud Native具备有一下特性:
1. 以云为基础
2. 云服务
3. 无服务
4. 可扩展
5. 敏捷
6. 云优先
7. 等等



可以理解为一种网络模型 

如何理解:
> 可以将它比如是应用程序或者微服务之间的TCP/IP，负责服务之间的通信，


其实仔细想一下，之前在写web服务器的时候，后面思考的那些一个完成高可用的服务器应该要考虑的问题，其实在服务网格当中均有相应的实现，比如下面的

- 服务发现
- 最基本的负载均衡
- 路由
- 流量控制
- 可高的通信机制
- 记录服务系统运行的正常状态，比如监控和日志系统

## 实质
单机服务网格：sidecar-独立的processer，部署在应用层的底层，跟应用部署在同一个台机器上面，应用程序A的通信交给代理SideCar A

两个应用程序通信变成四个进行通信的过程，应用程序A(比如最顶层的网关API) 与应用程序B()

Proxy就是Sidecar类型的东西

[![E2Bfe0.md.jpg](https://s2.ax1x.com/2019/05/10/E2Bfe0.md.jpg)](https://imgchr.com/i/E2Bfe0)


## sidecar
既然service mesh要达到将service的网络通信模块剥离出来，扮演这么一个功能的就是sidecar，sidecar负责接管对应服务的接入流量和出流量，并将微服务架构中以前就有的公共库，framework实现的熔断、限流、降级、服务发现、调用链分布式跟踪以及立体监控等功能。


# Istio
# 


# 参考链接
[简单理解下什么是 Cloud Native（云原生）_慕课手记](https://www.imooc.com/article/281379?block_id=tuijian_wz)

