---
title: day03-利用Wireshark分析H264码流
date: 2023-09-18 07:33:29
tags: 
	- Wireshark
categories: 
	- Wireshark

---



## 一、找到脚本放置的路径

### ①使用前面介绍的抓包方式，抓取数据包。



### ②找到连续的UDP大片数据，大概率就是I帧数据。

右键 —> DecodeAs -> 把current中的值改成RTP

![image-20230918192409693](day03-利用Wireshark分析H264码流/image-20230918192409693.png)



![image-20230918192558959](day03-利用Wireshark分析H264码流/image-20230918192558959.png)



![image-20230918192854378](day03-利用Wireshark分析H264码流/image-20230918192854378.png)



### ③我们可以看到PT=127



然后在 WireShark 工具栏中选择 Edit –> preferences –> protocols –> H264，把“H264 dynamic payload types”设成 127，点击 OK。

![image-20230918193350436](day03-利用Wireshark分析H264码流/image-20230918193350436.png)



### ④在wireshark如果发生丢包，怎么查看

将数据导出车csv格式的数据，用excel打开，提取seq进行排序。



![image-20230918195121425](day03-利用Wireshark分析H264码流/image-20230918195121425.png)
