---
title: day03-WebRTC源码环境
categories:
  - WebRTC源码
tags:
  - WebRTC
date: 2022-07-25 08:41:31
---



## 一、WebRTC整体架构

### 1、【重要】WebRTC的架构图



![image-20220727074051056](day03-WebRTC源码环境/image-20220727074051056.png)



### 2、【重要】WebRTC的数据流转图

- RTCP包是用于控制RTP包的
- RTP包里面都是媒体数据

![image-20220727074121906](day03-WebRTC源码环境/image-20220727074121906.png)



## 二、WebRTC资料路径

### 1、哪里获取WebRTC的源码？

- [通过官网获取](https://webrtc.googlesource.com/src/+/refs/heads/main)
- [通过声网镜像获取](https://webrtc.org.cn/mirror/)



![image-20220815064121684](day03-WebRTC源码环境/image-20220815064121684.png)



![image-20220815064130960](day03-WebRTC源码环境/image-20220815064130960.png)



### 2、记一个音视频参考文章

- avdancedu.com

![image-20220815064246826](day03-WebRTC源码环境/image-20220815064246826.png)



### 3、认识depot_tools 、gn、ninjia（说出大体功能即可）

- depot_tools： Google 官方提供的一个用来 <font color="red">checkout 、 compile 、 run 和 submit</font> 的工具集，可以帮助我们更好的学习和调试 Chromium 代码

- gn：它与cmake功能相同
- ninjia：它与make功能相同

## 三、[Mac下编译WebRTC（Mac和iOS版本）](https://segmentfault.com/a/1190000041179228)

#### 前言

随着新冠疫情的影响，这两年音视频的需求呈爆发式增长。在音视频领域中，WebRTC可以说是一个绕不开宝库，包括了音视频采集、编解码、传输、渲染的全过程。本文主要记录下在Mac平台上编译WebRTC Mac和iOS版本的全过程。

#### 设置代理

因为众所周知的原因，要下载WebRTC的源码是需要代理工具的。

```routeros
export http_porxy="http://127.0.0.1:21087"
export https_porxy="http://127.0.0.1:21087"
```

#### 安装工具depot_tools

git clone获取depot_tools

```awk
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
```

将depot_tools的路径配置到环境变量中

```routeros
export PATH=$PWD/depot_tools:$PATH
```

#### 下载webrtc源码

```bash
mkdir webrtc
cd webrtc
fetch --nohooks webrtc_ios
gclient sync
```

默认下载的是最新的源码，如果想要切换到指定分支，可以使用以下命令：

```mipsasm
# 查看可用版本分支
git branch -r
# 切换到m79分支
git checkout branch-heads/m79
gclient sync
# 或者强制切换到指定commit（b484ec0082948ae086c2ba4142b4d2bf8bc4dd4b是m79最后一次提交的commit id）
gclient sync -r b484ec0082948ae086c2ba4142b4d2bf8bc4dd4b --force
```

[可以在从这里获取webrtc所有release版本的信息](https://link.segmentfault.com/?enc=1eawLpdlwuggWFandAe3yA%3D%3D.ytdz0xpOZcM0IpYUvf2%2B65IcPLFY4c5NqXvnkBnneGU%2BnLMGSIlB5buxIcEhA2fW)

#### 编译

Mac版本：

```jboss-cli
gn gen out/mac-release --args='target_os="mac" target_cpu="x64" is_debug=false use_rtti=true is_component_build=false rtc_use_h264=false rtc_include_tests=false' --ide=xcode
ninja -C out/mac-release
```

编译成功后会在src\out\xxxx\下生成all.xcworkspace文件。打开就可以构建、调试webrtc的项目。其中APPRTCMobile是谷歌提供的示例demo，可以在Mac下直接编译运行。
