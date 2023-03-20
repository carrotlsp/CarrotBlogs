---
title: day11-WebRTC网络传输4-Turn协议、Turn连接、Turn消息机制、Turn类型的candidate
date: 2023-03-22 16:31:29
tags: 
	- WebRTC
categories: 
	- WebRTC源码Repeat

---



## 一、Turn原理、Turn连接

### 1、Turn协议工作原理？

> 在Turn协议中，主动发起请求的一端叫做 Client 端。
>
> Turn Server 会为 Client 端分配 relay 3478端口，用于接受 Peer 端发送来的数据。
>
> 被动端称为 Peer 端，当它知道 Client 端的 relay 地址之后，它就可以向 relay 地址直接发送数据。
>
> Turn Server 收到 Peer 端数据后，将会输出通过 3478 转发给Client端。
>
> 而 Turn Client 端发送数据到 Turn Server 之后，Turn Server 会解析数据的目的地址，然后会通过 relay 地址，转给对应的 PeerA 或 PeerB。

![image-20230317065814179](day11-WebRTC网络传输4/image-20230317065814179.png)



### 2、Turn连接的 5 元组，为什么不同呢？

![image-20230317065930504](day11-WebRTC网络传输4/image-20230317065930504.png)

### 3、TurnClient 与 TurnServer 的连接过程？

- 这个过程，可以倒回来多听几遍，很有用。

![image-20230317070114891](day11-WebRTC网络传输4/image-20230317070114891.png)

![image-20230317070314842](day11-WebRTC网络传输4/image-20230317070314842.png)

![image-20230317070319231](day11-WebRTC网络传输4/image-20230317070319231.png)

- 上面图中并没有带有凭证，所以TURN Server会给 TURN Client 端发送一个 401.

![image-20230317070344237](day11-WebRTC网络传输4/image-20230317070344237.png)

- 0x0113 中的 11 代表错误消息，然后带了个ERROR-CODE 401 就是表示未授权

![image-20230317070413142](day11-WebRTC网络传输4/image-20230317070413142.png)

- 上图是TURN Client收到 401之后，重新带上凭证去获取，这个凭证通常是USERNAME

![image-20230317070422646](day11-WebRTC网络传输4/image-20230317070422646.png)

> 上图 是Allocate 成功响应的消息类型，就会带有relay地址
>
> XOR-RELAYED-ADDRESS 是服务端为我们分配的 Relay地址
>
> XOR-MAPPER-ADDRESS：是本机映射的外网地址
>
> LIFETIME：表示我们这个连接可以持续多长时间，这里是以秒为单位的 600秒。



### 4、TurnClient 如何与 TurnServer 进行保活的呢？

![image-20230317070549194](day11-WebRTC网络传输4/image-20230317070549194.png)

- 上图是保活机制，如果需要连接一直有效，那么就需要进行【保活】

![image-20230317070600820](day11-WebRTC网络传输4/image-20230317070600820.png)

![image-20230317070605678](day11-WebRTC网络传输4/image-20230317070605678.png)

- 上面两图就是保活的request 和 response的抓包，LIFETIME=0表示服务端已经关闭连接了。



## 二、Turn协议数据传输机制？

### 1、TurnServer 如何判断哪些来源数据是需要进行传输的？哪些来源数据是需要丢弃的？

![image-20230317070853524](day11-WebRTC网络传输4/image-20230317070853524.png)

- 需要 Client 向 TURN Server 端进行授权申请，将合法的用户Peer A告诉 TURN Server，后续TURN Server就只转发 Client 和 Peer A之间的数据。

![image-20230317070919880](day11-WebRTC网络传输4/image-20230317070919880.png)

![image-20230317070928462](day11-WebRTC网络传输4/image-20230317070928462.png)

### 2、Turn传输数据的机制有哪两种？

- Send/Data 机制
- ChannelData 机制

### 3、Send 与 Data 机制是怎么样的？

![image-20230317072905447](day11-WebRTC网络传输4/image-20230317072905447.png)

![image-20230317072910891](day11-WebRTC网络传输4/image-20230317072910891.png)

![image-20230317072916388](day11-WebRTC网络传输4/image-20230317072916388.png)

### 4、ChannelData 机制是怎么样的？

![image-20230317072957363](day11-WebRTC网络传输4/image-20230317072957363.png)

![image-20230317073008866](day11-WebRTC网络传输4/image-20230317073008866.png)

![image-20230317073012681](day11-WebRTC网络传输4/image-20230317073012681.png)

![image-20230317073018109](day11-WebRTC网络传输4/image-20230317073018109.png)

![image-20230317073023233](day11-WebRTC网络传输4/image-20230317073023233.png)

![image-20230317073028401](day11-WebRTC网络传输4/image-20230317073028401.png)

## 三、收集 Turn 类型的 Candidate

### 1、WebRTC STUN/TURN 消息类型？

![image-20230321063931016](day11-WebRTC网络传输4/image-20230321063931016.png)



### 2、代码逻辑

![image-20230321064014465](day11-WebRTC网络传输4/image-20230321064014465.png)

- 上面的步骤，需要的时候自己走一遍，理解每个方法都做了些什么？

### 3、那么 WebRTC 收到来自 TURN 的 AllocateResponse 会怎么做呢？

![image-20230321064117503](day11-WebRTC网络传输4/image-20230321064117503.png)



![image-20230321064122084](day11-WebRTC网络传输4/image-20230321064122084.png)



![image-20230321064127081](day11-WebRTC网络传输4/image-20230321064127081.png)



