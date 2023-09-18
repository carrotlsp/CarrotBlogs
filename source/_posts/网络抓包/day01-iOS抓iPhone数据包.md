---
title: day01-iOS抓iPhone的wireshark数据包
date: 2023-09-18 07:28:30
tags: 
	- Wireshark
categories: 
	- Wireshark

---



> 在iOS应用开发过程中，通过抓包调试服务接口的场景时常出现。Charles和Wireshark是我在iOS开发过程中最常用的两款软件。



**Charles**是很强大的网络请求抓包工具，常用于抓包HTTP/HTTPS请求。而作者在做IoT项目时，智能硬件配网协议是基于TCP/UDP或者蓝牙的，需要用**Wireshark**进行抓包调试。[Wireshark官网](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.wireshark.org)



## 一、获取iPhone的UDID



从`Xcode菜单栏 -> Window -> Devices and Simulators`可以更方便地获取准确的UDID。图示如下：

![img](day01-iOS抓iPhone数据包/webp)





## 二、为iPhone创建虚拟网卡

```
$ rvictl -s decb66caf7012a7799c2c3edxxxxxxxx7f5a715e

Starting device decb66caf7012a7799c2c3edxxxxxxxx7f5a715e [SUCCEEDED] with interface rvi0
```



## 三、启动Wireshark，找到 rvi0 进行抓包



![img](day01-iOS抓iPhone数据包/webp-20230918170235617)





双击`rvi0`即可进入抓包界面。
若此时出现如下弹窗，则说明无权限访问该接口。



![img](day01-iOS抓iPhone数据包/webp-20230918170259224)



这时，退出Wireshark，然后在终端上使用下述命令重新打开Wireshark就可以了。

```
$ sudo /Applications/Wireshark.app/Contents/MacOS/Wireshark 
Password:
```



## 四、使用tcpdump抓包，借助wireshark分析包

```shell
# -i, 要监听的网卡名称，-i rvi0监听虚拟网卡。不设置的时候默认监听所有网卡流量。
# -w，保存的路径以及文件名。
sudo tcpdump -i rvi0  -w dump.pcap
```



## 五、一些注意事项【坑】



#### ① 时间戳不对的问题

直接用wireshark图形化界面抓取rvi0的数据，可能存在时间戳不对的问题。

比如一个http请求，请求时间和响应时间耗时为0，这是不可能的。

**解决方式：** 上述**方法四** tcpdump抓包，先把数据抓取，然后用wireshark打开查看即可。



