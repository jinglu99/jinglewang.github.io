---
title: RPC
date: 2018/2/5 17:48:25  # 文章发表时间
toc: true
tags:
- Java
- RPC
categories:
- Java
- RPC
thumbnail: http://pic.jingl.wang/2019-03-02-145509.png

---

​	第一次用到RPC框架是在实习公司的那段时间做一个搜索引擎的项目，使用了阿里的Dubbo，用来打通不同工程间的信息交互，当时对RPC框架并不是非常了解，只知道用它可以将远程服务像本地方法一样得调用，虽然对它有着一点好奇，却也并没有打算去深入了解他。后来，大四准备毕业设计了，也没有什么好的想法，所以就想写一个自己的RPC框架，重复造个轮子，权当学习，本文只是简单的介绍一下RPC框架。

<!--more-->
## RPC是什么

​	远程过程调用（Remote Procedure Call，缩写为 RPC），顾名思义，就是一个允许在分布式环境中通过网络像调用本地方法一样调用其他远程服务器提供的服务的协议。它假定了某些通信协议的存在，如TCP、UDP、HTTP，并利用这些通信协议进行信息传递，在OSI模型中它跨越了应用层和传输层，所以在使用RPC协议时，并不需要关注底层的通信，这大大简化了分布式系统的开发难度。

​	RPC采用的是C/S模型，请求程序是一个客户机，服务提供方是一个服务器。

![](http://pic.jingl.wang/2018-02-05-162611.png)

如上图所示，Client发起一个请求，RPC框架将这个请求发送到服务提供方Server，此时Client将会一直等待直到Server处理完毕并返回结果或超时，此时Server接受到Client发送的请求，并对这个请求进行解析，计算结果，将结果以同样的方式返回，然后Server重新进入监听状态，以迎接下一个请求的到来。

​	总的来说，RPC框架就是一个不同系统之间进行交互的聊天工具。在开发分布式应用时，完全不需要考虑底层到底是用了HTTP协议，还是其他通信协议，因为这是RPC框架自己的事情，而你只要告诉RPC框架，你需要调用的是什么方法。





## RPC存在的意义

​	在DUBBO的USER BOOK中，介绍了四种随着互联网发展，网站应用不断扩大规模是，必定会经历的4中网站架构：单一应用架构、垂直应用架构、分布式服务架构和流动计算架构。

![](http://pic.jingl.wang/2018-02-05-162636.png)

​	在起步阶段，网站的流量很小，为减少部署节点和成本，可以将所有的功能放在一个应用中，随着网站的发展，流量慢慢变大，单一应用架构已经不足以支撑日益增加的访问量，此时可以将一个应用拆分成多个互不相干的应用，以提升单个应用的效率，这便是垂直应用架构，随着网站规模进一步的扩大，垂直应用越来越多，应用间的交互不可避免，所以需要将核心的业务从应用中抽离出来，变成公共的服务，而此时，一个能整合业务复用及整合的RPC框架就是关键。当服务越来越多，服务的管理就成了一个问题，这时候灵活的服务治理（SOA)就是最重要的地方。

​	无论是什么网站，殊途同归，微服务总是最终的归宿，如何让各个服务之间相互的调用变得简单和高效，这就是一个RPC框架应该去解决的问题，



## 结构

在 Nelson 1984年发表的论文[ Implementing Remote Procedure Calls](http://birrell.org/andrew/papers/ImplementingRPC.pdf) 中提出，RPC由5个部分组成：

 * User
 * User Stub
 * RPCRuntime
 * Server Stub
 * Server





这5个部分的联系如下图所示

![](http://pic.jingl.wang/2018-02-06-053158.png)

根据 Nelson 的描述，当远程调用发生时，User调用的是本地的User Stub，User stub 将请求根据协议打包成一个消息，通过RPCRuntime传递到服务端，此时，客户端将进入等待状态。服务端的RPCRuntime接受客户端的请求消息，接着经由Server Stub拆包，调用本地方法进行相应计算，获得结果。最后将结果经由相同的链路发送到客户端User手中，进行返回。

​	传统的RPC模型只描述了点到点的调用流程，在实际运用中，还需要考虑负载均衡、服务的高可用等问题，所以产品级的RPC框架一般还会有一个注册中心来提供服务的发现、注销和负载均衡等功能。

![](http://pic.jingl.wang/2018-02-09-023427.png)

在Client 和 Server 直接还是如 Nelson 的模型一样进行连接，与Nelson 不同的是Client和Server都需要与一个注册中心Register进行通信，当有新的Server发布时，Server 需要向 Register 进行注册， 此时，Register 会通知所有的Client有新的服务发布，并更新Client的缓存，当Client发起远程调用时，会在缓存中选取一台Server进行通信。Client 和 Server 与 Register 都有心跳检测，所以，当集群中某一台服务器下线时，Register 会马上知道，并会通知所有的Client该服务器下线，刷新Client缓存。



## 流程

总结一下在 Nelson 描述的调用模型中，远程服务调用的步骤：

* client调用client stub，这是一次本地过程调用。
* client stub将参数打包成一个消息，然后发送这个消息。
* client所在的系统将消息发送给server。
* server的的系统将收到的包传给server stub。
* server stub解包得到参数。
* 最后server stub调用服务过程. 返回结果按照相反的步骤传给client。




## REFER

[Go RPC 开发指南](https://www.gitbook.com/book/smallnest/go-rpc-programming-guide/details)

[深入浅出 RPC - 浅出篇](http://blog.csdn.net/mindfloating/article/details/39473807)