---
layout: post
title:  "Service Mesher"
date:   2019-03-25 22:14:54
categories: 微服务
comments: true
tags:
    - 服务网格
    - 微服务
---

* content
{:toc}

# service mesh
最近一直想关注下比较前沿的后端技术，所以就打算学习一下比较火的service mesh 服务网格技术。

**前言**
新一代的网络通信技术，不同应用之间的通信如何做，底层的，如何将不同service 之间通信服务模块抽象出来，让应用更加专注于业务实现即可，而无需关系与依赖模块之间的相互通信。互联网的体量变得越来越大，从最开始的单体应用，到SOA，再到现在面向分布式的微服务系统，将一个应用分拆多个服务，同时为了达到高可用的目的，目前一般而言考虑到负载均衡高可用，一个服务一般部署多个实例。

[![VMc6TP.md.jpg](https://s2.ax1x.com/2019/05/30/VMc6TP.md.jpg)](https://imgchr.com/i/VMc6TP)

## 背景 

微服务分层处理，各个组件组合在一起就是很复杂，容易出现问题，特别是在承载大量服务的云原生环境下，各种服务互相之间的调用形成错综复杂的关系，并且应用部署在云原生的环境当中，比如容器docker,或者k8s当中等容器当中，如何这种复杂的环境构建一个高效，健壮高可用的系统就成为一个头疼的问题，比如微服务系支持熔断，降级，服务限流，认证和授权，服务监控，等这是肯定要支持的在，这也就是服务治理的问题，微服务是怎样达到上面的目的呢，一般都是将服务治理代码和业务代码放在一起，聚合在一起，不同服务之间通过SDK进行通信，对业务代码有侵入性，这也无法做到SDK升级对业务透明。

[![VHDe5n.md.png](https://s2.ax1x.com/2019/06/17/VHDe5n.md.png)](https://imgchr.com/i/VHDe5n)

**将微服务框目前存在的问题总结如下**
- 业务代码与微服务框架SDK强耦合(在同一个进程中)
- 业务代码与微服务框架升级绑定，无法实现平滑升级
- 微服务框架SDK多语言并行与维护成本高，基于不同语言开发的应用程序如何进行沟通
- 异构服务框架难以共存完成渐进式演进

说白了我们能不能将不同之间的通信以及常用的服务治理功能从业务代理剥离开来，让业务开发者只关心自身业务，引入一个代理为不同应用层之间完成通信功能，该代理对应用层完全透明，是一个独立的进程，对应用层的通信进行拦截，这就是在为部署在云原生环境当中微服务之引入service mesh的原因。

[![E2BiZV.md.jpg](https://s2.ax1x.com/2019/05/10/E2BiZV.md.jpg)](https://imgchr.com/i/E2BiZV)

## 定义
先来看看英文定义
> A service mesh is a dedicated infrastructure layer for handling service-to-service communication. It is responsible for the reliable delivery of requests through the complex topology of services that comprise a modern, cloud native application. In practise, the service mesh is typically implemented as an array of lightweight network proxies that are deployed alongside application code, without the application needing to be aware.

**中文自己的理解**
将业务层与通信逻辑层分离，是一个基础服务通信模块，用于处理服务通信，云原生应用有着复杂的拓扑结构关系，服务网格负责在这些拓扑中实现请求的可靠传递，在实践当中，**服务网格通常实现为一组轻量级网络代理**，他们与**应用程序**部署在一起，而对应用程序透明。可以说是一种轻量级的网络代理，可以将sevice mesh比做微服务之间的TCP/IP层，负责服务之间的调用，限流，熔断以及监控等服务治理功能。

因此，可以总结下Service Mesh具有如下的优点：
- 屏蔽分布式系统通信的复杂性(负载均衡、服务发现、认证授权、监控追踪、流量监控),服务只用关心业务逻辑
- 真正的语言无关，服务可以使用任何语言编写，只需要和Service Mesh通信即可
- 对应用透明，对Service Mesh 组件可以单独升级;

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

其实仔细想一下，之前在写web服务器的时候，后面思考的那些一个完成高可用的服务器应该要考虑的问题，其实在服务网格当中均有相应的实现，比如下面的

- 服务发现
- 最基本的负载均衡
- 路由
- 流量控制
- 可高的通信机制
- 记录服务系统运行的正常状态，比如监控和日志系统
- 多种语言机制支持
- 动态配置

这些都是由应用层代理来实现的，将这些功能功能嵌套在业务代码当中，在编写业务代码的过程当中，

## TCP 端到端通信
我们知道TCP五层协议中，从上到下是应用层，运输层，网络层，数据链路层，物理层， 

[![VolJZ4.md.png](https://s2.ax1x.com/2019/06/15/VolJZ4.md.png)](https://imgchr.com/i/VolJZ4)

应用层通过TCP传输层屏蔽底层的网络细节，通过端口到端口，找到不同主机上面的不同应用程序，完成通信过程，这个过程让人从外面看就好像端到端的通信，当然这是一种应用程序之间的逻辑通信，主机上面的应用程序不需要知道具体底层的网络通信细节，TCP通过建立连接的(一个四元组，源ip+端口：目的ip+端口建立了一条虚拟的连接)方式让不可靠的底层网络通信，加上各种可靠传输机制，比如无差错传输、超时重传、自动重传协议AQS、流量控制、拥塞控制、序列号机制、滑动窗口机制、ACK确认机制，为应用程序之间建立一个高效的通信机制，目前互联网上面应用层的服务基本上都是通过TCP或者UDP来完成通信。

**TCP可靠传输之一：无差错情况**

[![VodMKs.md.png](https://s2.ax1x.com/2019/06/15/VodMKs.md.png)](https://imgchr.com/i/VodMKs)

**TCP可靠传输之二：滑动窗口机制**

![VodsIK.png](https://s2.ax1x.com/2019/06/15/VodsIK.png)

**TCP可靠传输之三：拥塞控制**

[![VowJeI.md.png](https://s2.ax1x.com/2019/06/15/VowJeI.md.png)](https://imgchr.com/i/VowJeI)

同样在SOA服务架构横行的今天，人们还是从原始的经典的TCP协议家族触发，能不能在今天错杂复杂的应用服务构建一个端到到端的通信，屏蔽不同service通信之间的
细节，为service通信之间需要的各种功能，比如服务治理、熔断降级、限流、授权、网关、认证、重试机制、动态配置、监控等功能。

**按照我自己的比如理解，其实可以把service mesh 比如tcp层传输层，微服务当中的service比如最顶层的应用层，然后Service Mesh提供的服务治理、熔断降级、限流、授权、网关、监控、重试机制等基础服务功能类比于TCP当中的提供的超时重试、ACK确认机制、流量控制、拥塞控制、滑动窗口等机制，通过service mesh提供的这些基础设施服务，我们通过Sevice Mesh提供的这些基础设施服务，为应用程序不同service 提供一个健壮高可用的通信服务。**

## Service Mesh实质
单机服务网格：sidecar-独立的processer，部署在应用层的底层，跟应用部署在同一个台机器上面，应用程序A的通信交给代理SideCar A
两个应用程序通信变成四个进行通信的过程，应用程序A(比如最顶层的网关API) 与应用程序B()



[![E2Bfe0.md.jpg](https://s2.ax1x.com/2019/05/10/E2Bfe0.md.jpg)](https://imgchr.com/i/E2Bfe0)


## 架构
Service Mesh 架构分为两个部分：
- data plane，数据平面本质上是处理服务之间通信的代理服务。在Istio中，数据平面被部署为“边车代理”，即添加到主应用程序的支持服务;例如，在Kubernetes基础架构中，代理与具有共享网络命名空间的应用程序部署在同一个窗格中，数据平面还可以为您体用可观察性，特别是日志和度量聚合的形式
  如何理解Service Mesh 数据平面这个东西？
  可以拿NGINX,HAPROXY来做一个案列，NGINX,HAProxy 这些负载均衡均提供了数据平面功能，在Sevice Mesh 当中Envoy充当Istio的数据平面，数据平面的角色可以理解为SideCar 
  
- control plane,控制平面，主要包括以下三个组件
    - Pilot(领航员)： 为数据平面提供服务发现，为流量管理功能实现灵活的路由，弹性机制(超时，重试，熔断等)，流量控制特性
    - Mixer(混合器)：Mixer是一个平台独立的组件，它的职责就是在Service Mesh实施访问控制和使用策略，并从envoy 代理和其它服务中收集自动测量数据
    - Istio-Auth:  使用强大的健服务之间以及服务和终端用户间的认证。可以将Service Mesh 中未加密的流量升级，并为运维人员提供服务标识和
  
[![VoBPKJ.md.png](https://s2.ax1x.com/2019/06/15/VoBPKJ.md.png)](https://imgchr.com/i/VoBPKJ)



Istio当中充当数据平面的就是Envoy，但是我们也可以使用NGINX替代Envoy的功能，

[![VoytTs.md.png](https://s2.ax1x.com/2019/06/15/VoytTs.md.png)](https://imgchr.com/i/VoytTs)


## SideCar
既然service mesh要达到将service的网络通信模块剥离出来，扮演这么一个功能的就是sidecar，sidecar负责接管对应服务的接入流量和出流量，并将微服务架构中以前就有的公共库，framework实现的熔断、限流、降级、服务发现、调用链分布式跟踪以及立体监控等功能。sidecar是服务代理模式，是一种边车模式，也就是边缘，sidecar和应用程序分别部署在不同的进程中。不同的Sevice Mesh 当中具体对sidecar实现具有不同的实现方案。

[![Vou9je.md.png](https://s2.ax1x.com/2019/06/15/Vou9je.md.png)](https://imgchr.com/i/Vou9je)



# Service Mesh应用难点

当然，Service Mesh也面临一些挑战，：
- Service Mesh 组件以代理模式拦截计算并转发请求，一定程度会降低通信性能，并增加系统资源开销,多了中间一个层，增加了网络通信的时间
- Service Mesh 组件接管了网络流量，因此服务的整体稳定性也依赖与Service Mesh, 同时额外引入的大量Service Mesh服务实例的运维和管理也是一个挑战
- Service Mesh 一般还包括数据平面和控制平面

# 参考链接
- [简单理解下什么是 Cloud Native（云原生)](https://www.imooc.com/article/281379?block_id=tuijian_wz)
- [阿里巴巴中间团队](http://jm.taobao.org/2018/07/05/Mesh%E4%BD%93%E7%B3%BB%E4%B8%AD%E7%9A%84Envoy/#%E8%83%8C%E6%99%AF)
