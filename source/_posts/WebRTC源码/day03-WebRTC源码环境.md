---
title: day03-WebRTC源码环境
categories:
  - WebRTC源码
tags:
  - WebRTC
date: 2022-08-25 08:41:31
---



## 一、WebRTC整体架构

### 1、【重要】WebRTC的架构图



![image-20220727074051056](day03-WebRTC源码环境/image-20220727074051056.png)



### 2、【重要】WebRTC的数据流转图

- RTCP包是用于控制RTP包的
- RTP包里面都是媒体数据

![image-20220727074121906](day03-WebRTC源码环境/image-20220727074121906.png)



## 二、WebRTC源码分析环境的搭建

### 1、哪里获取WebRTC的源码？



