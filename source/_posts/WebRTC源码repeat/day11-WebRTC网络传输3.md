---
title: day11-WebRTC网络传输3-获取本地Candidate、认识Stun、StunRequest、StunResponse
date: 2023-03-20 16:31:29
tags: 
	- WebRTC
categories: 
	- WebRTC源码Repeat

---



## 一、获取本地Candidate

### 1、获取本地Candidate的过程？

![image-20230316073044450](day11-WebRTC网络传输3/image-20230316073044450.png)

![image-20230316073049698](day11-WebRTC网络传输3/image-20230316073049698.png)

![image-20230316073059188](day11-WebRTC网络传输3/image-20230316073059188.png)

## 二、理解Stun协议

### 1、STUN的全称是什么？STUN存在的意义是什么？STUN的工作模式是什么？

- Session Tranversal Utilities for NAT

- STUN存在的目的就是进行 NAT 穿越
- STUN是典型的客户端/服务器模式。客户端发送请求、服务端进行响应。



### 2、 STUN消息类型？

![image-20230316073400268](day11-WebRTC网络传输3/image-20230316073400268.png)

![image-20230316073409245](day11-WebRTC网络传输3/image-20230316073409245.png)

![image-20230316073413934](day11-WebRTC网络传输3/image-20230316073413934.png)

> N：None
> M：Must
> O:Options



### 3、认识 StunAttribute 类结构图？

![image-20230316073505754](day11-WebRTC网络传输3/image-20230316073505754.png)

![image-20230316073509553](day11-WebRTC网络传输3/image-20230316073509553.png)

![image-20230316073513670](day11-WebRTC网络传输3/image-20230316073513670.png)

### 4、认识StunBindingRequest ？

![image-20230316073527129](day11-WebRTC网络传输3/image-20230316073527129.png)

![image-20230316073531529](day11-WebRTC网络传输3/image-20230316073531529.png)

![image-20230316073534911](day11-WebRTC网络传输3/image-20230316073534911.png)

![image-20230316073538630](day11-WebRTC网络传输3/image-20230316073538630.png)

![image-20230316073542629](day11-WebRTC网络传输3/image-20230316073542629.png)

![image-20230316073545958](day11-WebRTC网络传输3/image-20230316073545958.png)

![image-20230316073550099](day11-WebRTC网络传输3/image-20230316073550099.png)

![image-20230316073553869](day11-WebRTC网络传输3/image-20230316073553869.png)



### 5、如何解析 StunBindingResponse 消息

![image-20230316073628090](day11-WebRTC网络传输3/image-20230316073628090.png)

![image-20230316073632229](day11-WebRTC网络传输3/image-20230316073632229.png)



![image-20230316073638085](day11-WebRTC网络传输3/image-20230316073638085.png)
