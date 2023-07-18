---
title: day12-WebRTC服务质量3-NACK
date: 2023-06-30 09:31:29
tags: 
	- WebRTC
categories: 
	- WebRTC源码Repeat




---



## 一、认识NACK 、RTX【总体概念】

### 1、NACK的作用是什么？

- **NACK（Negative Acknowledge）用于告诉对方，丢了哪些包。**
- 当通信的双方传输数据的时候，接收方会通过SequenceNumber知道哪些包收到了，哪些包丢失了。
- 每隔一段时间，接收方就将丢失的SequenceNumber传给发送方。

### 2、RTX的作用是什么？

- **RTX（Real Time Retransmission）用户重传丢失的包。**
- 特点：重传的时候不会使用原来的PayloadType、SSRC、SequenceNumber，会使用新的。



### 3、NACK/RTX的工作机制，什么时候确认是否支持NACK和RTX？

- 在SDP协商的时候确认是否支持NACK或者RTX

- 核心机制：通过SequenceNumber找出丢失的包，发送给对端；对端通过RTX重传丢失的包。

![image-20230627070225068](day12-WebRTC服务质量3-NACK/image-20230627070225068.png)





- 在 12-9中详细讲述的NACK的整个机制，可以多听听。



### 4、如果没有RTX，WebRTC可以丢包重传吗？

- 可以，原因在后续讲解。
- 尝试回答下面的问题。



![image-20230627070906478](day12-WebRTC服务质量3-NACK/image-20230627070906478.png)

## 二、NACK的抓包分析流程【偏向抓包】

### 1、NACK的格式？PID、BLP分别是什么意思？

![image-20230629071606431](day12-WebRTC服务质量3-NACK/image-20230629071606431.png)

![image-20230629071635878](day12-WebRTC服务质量3-NACK/image-20230629071635878.png)



### 2、通过抓包看一下NACK的格式？

 ![image-20230629071843062](day12-WebRTC服务质量3-NACK/image-20230629071843062.png)



### 3、WebRTC接收NACK消息的过程？

![image-20230629075309875](day12-WebRTC服务质量3-NACK/image-20230629075309875.png)



![image-20230629075327283](day12-WebRTC服务质量3-NACK/image-20230629075327283.png)

![image-20230629075351373](day12-WebRTC服务质量3-NACK/image-20230629075351373.png)



![image-20230629075401352](day12-WebRTC服务质量3-NACK/image-20230629075401352.png)

![image-20230629075422708](day12-WebRTC服务质量3-NACK/image-20230629075422708.png)



![image-20230629075440888](day12-WebRTC服务质量3-NACK/image-20230629075440888.png)

### 4、RTX协议？

![image-20230629075522363](day12-WebRTC服务质量3-NACK/image-20230629075522363.png)





### 5、如何找到RTX包，并在Wireshark中抓取RTX？

![image-20230629075541704](day12-WebRTC服务质量3-NACK/image-20230629075541704.png)

![image-20230629075600282](day12-WebRTC服务质量3-NACK/image-20230629075600282.png)

![image-20230629075636619](day12-WebRTC服务质量3-NACK/image-20230629075636619.png)





![image-20230629075736486](day12-WebRTC服务质量3-NACK/image-20230629075736486.png)



### 6、如何找到NACK对应的RTX包？

![image-20230629075936660](day12-WebRTC服务质量3-NACK/image-20230629075936660.png)





![image-20230629080044584](day12-WebRTC服务质量3-NACK/image-20230629080044584.png)



![image-20230629080105726](day12-WebRTC服务质量3-NACK/image-20230629080105726.png)



### 7、WebRTC发送RTX包的过程？

![image-20230630070151147](day12-WebRTC服务质量3-NACK/image-20230630070151147.png)

![image-20230630070211860](day12-WebRTC服务质量3-NACK/image-20230630070211860.png)



![image-20230630070252324](day12-WebRTC服务质量3-NACK/image-20230630070252324.png)



![image-20230630070444635](day12-WebRTC服务质量3-NACK/image-20230630070444635.png)

![image-20230630070532659](day12-WebRTC服务质量3-NACK/image-20230630070532659.png)

![image-20230630070618387](day12-WebRTC服务质量3-NACK/image-20230630070618387.png)





## 三、NACK的代码流程【偏向代码，了解即可】

### 1、判断包位置的关键算法函数AheadOf（，后续用到再详细了解）

- SequenceNumber 是循环的

![image-20230627073438288](day12-WebRTC服务质量3-NACK/image-20230627073438288.png)

![image-20230627073537831](day12-WebRTC服务质量3-NACK/image-20230627073537831.png)

![image-20230627073619318](day12-WebRTC服务质量3-NACK/image-20230627073619318.png)

![image-20230627073637277](day12-WebRTC服务质量3-NACK/image-20230627073637277.png)

![image-20230627073644155](day12-WebRTC服务质量3-NACK/image-20230627073644155.png)

![image-20230627073657331](day12-WebRTC服务质量3-NACK/image-20230627073657331.png)

### 2、NACK的处理流程，也就是NACK的调用栈？



![image-20230629070250676](day12-WebRTC服务质量3-NACK/image-20230629070250676.png)

![image-20230629070318387](day12-WebRTC服务质量3-NACK/image-20230629070318387.png)

![image-20230629070344337](day12-WebRTC服务质量3-NACK/image-20230629070344337.png)

![image-20230629070404907](day12-WebRTC服务质量3-NACK/image-20230629070404907.png)

### 3、判断丢包的关键逻辑？

![image-20230629070448028](day12-WebRTC服务质量3-NACK/image-20230629070448028.png)

![image-20230629070554988](day12-WebRTC服务质量3-NACK/image-20230629070554988.png)

![image-20230629070609361](day12-WebRTC服务质量3-NACK/image-20230629070609361.png)

![image-20230629070625914](day12-WebRTC服务质量3-NACK/image-20230629070625914.png)

![image-20230629070642212](day12-WebRTC服务质量3-NACK/image-20230629070642212.png)



![image-20230629070656954](day12-WebRTC服务质量3-NACK/image-20230629070656954.png)

### 4、找到真正的丢包？

![image-20230629070740356](day12-WebRTC服务质量3-NACK/image-20230629070740356.png)

![image-20230629070755835](day12-WebRTC服务质量3-NACK/image-20230629070755835.png)

![image-20230629070815624](day12-WebRTC服务质量3-NACK/image-20230629070815624.png)

![image-20230629070907644](day12-WebRTC服务质量3-NACK/image-20230629070907644.png)

![image-20230629070937378](day12-WebRTC服务质量3-NACK/image-20230629070937378.png)



![image-20230629070953970](day12-WebRTC服务质量3-NACK/image-20230629070953970.png)



![image-20230629071047419](day12-WebRTC服务质量3-NACK/image-20230629071047419.png)



### 5、VP8关键帧的判断？

![image-20230629071215101](day12-WebRTC服务质量3-NACK/image-20230629071215101.png)



![image-20230629071250408](day12-WebRTC服务质量3-NACK/image-20230629071250408-7993974.png)

![image-20230629071307477](day12-WebRTC服务质量3-NACK/image-20230629071307477.png)





![image-20230629071339526](day12-WebRTC服务质量3-NACK/image-20230629071339526.png)





![image-20230629071407530](day12-WebRTC服务质量3-NACK/image-20230629071407530.png)



![image-20230629071434238](day12-WebRTC服务质量3-NACK/image-20230629071434238.png)

