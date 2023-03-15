---
title: day11-WebRTC网络传输2-PortAllocator、AllocationSequence
date: 2023-03-18 16:31:29
tags: 
	- WebRTC
categories: 
	- WebRTC源码Repeat

---



## day01指针关键知识点

### 1、PortAllocator 类关系图？

![image-20230316065618548](day11-WebRTC网络传输2/image-20230316065618548.png)

### 2、什么时候创建 BasicProtAllocator对象？

- 在 CreatePeerConnection 的时候，创建 BasicPortAllocator对象。
- 也就是一个 PeerConnection 对应一个 BasicPortAllocator 对象。

![image-20230316065812873](day11-WebRTC网络传输2/image-20230316065812873.png)

![image-20230316065817523](day11-WebRTC网络传输2/image-20230316065817523.png)

![image-20230316065821625](day11-WebRTC网络传输2/image-20230316065821625.png)

![image-20230316065825666](day11-WebRTC网络传输2/image-20230316065825666.png)

![image-20230316065829199](day11-WebRTC网络传输2/image-20230316065829199.png)

![image-20230316065833826](day11-WebRTC网络传输2/image-20230316065833826.png)

![image-20230316065838014](day11-WebRTC网络传输2/image-20230316065838014.png)

## 二、认识 AllocationSequence

### 1、AllocationSequence 类结构？

![image-20230316070014239](day11-WebRTC网络传输2/image-20230316070014239.png)

![image-20230316070019424](day11-WebRTC网络传输2/image-20230316070019424.png)

![image-20230316070024243](day11-WebRTC网络传输2/image-20230316070024243.png)

### 2、AllocationSequence 的作用是啥？

- AllocationSequence作用：就是用来创建 candidate 的

![image-20230316070142468](day11-WebRTC网络传输2/image-20230316070142468.png)

![image-20230316070149758](day11-WebRTC网络传输2/image-20230316070149758.png)

![image-20230316070157010](day11-WebRTC网络传输2/image-20230316070157010.png)



### 3、PeerConnection、PortAllocator、PortAllocationSession之间的数目关系？

- 每个PeerConnection 有一个 PortAllocator
- 每个 PortAllocator 对应一个 PortAllocationSession



### 4、Network、AllocationSequence、UDPPort、RelayPort、Candiate之间的数目关系？【最重要的概念点】

- 每个 Network 对应一个 AllocationSequence
- UDPPort/RelayPort 等都是 AllocationSequence 分配的
- 而 Candidate 又是有 UDPPort/RelayPort生成的

![image-20230316070609583](day11-WebRTC网络传输2/image-20230316070609583.png)



