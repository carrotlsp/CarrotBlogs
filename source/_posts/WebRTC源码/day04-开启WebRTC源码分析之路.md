---
title: day04-开启WebRTC源码分析之路
categories:
  - WebRTC源码
tags:
  - WebRTC
date: 2022-07-28 09:41:31
---



## 一、认识官方提供的demo

### 1、学习WebRTC要牢牢抓住哪个demo？为什么？

- 抓住：peerConnection_client 这个demo【只有Windows环境有】
- 原因：因为这个demo几乎囊括了WebRTC所有主要用法



### 2、peerconnection_client 的主要工作流程是什么？

![image-20220815065720677](day04-开启WebRTC源码分析之路/image-20220815065720677.png)



### 3、peerconnection_client 类关系图

![image-20220815065751100](day04-开启WebRTC源码分析之路/image-20220815065751100.png)



### 4、peerconnection_client 的时序图

- 这些图真的太棒了，很容易帮助理解东西
- 一定要学会来怎么绘制这些图

![image-20220815065959477](day04-开启WebRTC源码分析之路/image-20220815065959477.png)

![image-20220815070015114](day04-开启WebRTC源码分析之路/image-20220815070015114.png)

### 5、媒体协商的步骤，一共8个步骤，一定要学会来！

- 下面这两个图也太棒了

![image-20220815070125005](day04-开启WebRTC源码分析之路/image-20220815070125005.png)

![image-20220815070348527](day04-开启WebRTC源码分析之路/image-20220815070348527.png)

### 6、传统协商存在什么问题？如何解决？（暂时了解吧，好像还没用到）

![image-20220815070641167](day04-开启WebRTC源码分析之路/image-20220815070641167.png)

![image-20220815070650570](day04-开启WebRTC源码分析之路/image-20220815070650570.png)

![image-20220815070701533](day04-开启WebRTC源码分析之路/image-20220815070701533.png)

![image-20220815070712749](day04-开启WebRTC源码分析之路/image-20220815070712749.png)

![image-20220815070718373](day04-开启WebRTC源码分析之路/image-20220815070718373.png)



### 7、信令处理时序图

- 图太好了

![image-20220815070756560](day04-开启WebRTC源码分析之路/image-20220815070756560.png)

### 8、WebRTC Native 开发过程

- 图也是超级棒

![image-20220815070903392](day04-开启WebRTC源码分析之路/image-20220815070903392.png)
