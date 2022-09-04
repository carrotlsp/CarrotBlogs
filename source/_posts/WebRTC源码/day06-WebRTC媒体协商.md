---

title: day06-WebRTC媒体协商
categories:
  - WebRTC源码
tags:
  - WebRTC
date: 2022-08-15 20:41:31
---



## 一、认识SDP

### 1、为什么要选择SDP格式作为WebRTC的传输格式？

- 与XML相比较，SDP采用key-value的模式，数据存放比较紧凑，网络传输更加节约带宽
- SDP采用key-value模式，也更容易读写数据



### 2、SDP的结构图，太棒了！

![image-20220821085122868](day06-WebRTC媒体协商/image-20220821085122868.png)

![image-20220821085227377](day06-WebRTC媒体协商/image-20220821085227377.png)

- m：就是media，表示媒体曾
- a：就是attribute，表示属性
- c：就是connection，表示连接

- v：就是version，代表SDP的版本，SDP的版本要一致才能进行协商
- s：就是Session，表示会话，会话是一个全局的
- 0：就是ower，表示谁拥有这个会话



### 3、结合具体例子，学习SDP

![image-20220821085444989](day06-WebRTC媒体协商/image-20220821085444989.png)

- 后面详细分析SDP的时候，可以倒回来学习一下6-2章节



### 4、什么ICE-FULL？什么是ICE-LITE？

- 目前通常采取 <font color="red">ICE-LITE模式</font>

- 表示客户端与服务端双方发送candidate的时候，是进行双向还是单向的问题

![image-20220821085655333](day06-WebRTC媒体协商/image-20220821085655333.png)

5、什么是 <font color="red">PlanB</font>？什么是<font color="red">UnifiedPlan</font>？

- 就是当一个WebRTC有多个音频轨道的时候，用什么方式来区分而已。
- 目前最新版本的WebRTC基本都是使用 <font color="red">UnifiedPlan</font> 来进行传输。

![image-20220821090514813](day06-WebRTC媒体协商/image-20220821090514813.png)

![image-20220821090958900](day06-WebRTC媒体协商/image-20220821090958900.png)

### 6、老师小节

- 对于SDP中的每个字段，都要非常清除其含义，只有这样，才能学好WebRTC
- 只有了解了SDP中的每个信息，才能在阅读代码的时候，游刃有余

### 7、SDP的内容【从功能上】归纳整理，这图太强了吧？ 



![image-20220821091859540](day06-WebRTC媒体协商/image-20220821091859540.png)

### 8、SDP的内容【从WebRTC角度】归纳整理，这图太强了吧？ 

![image-20220821092002572](day06-WebRTC媒体协商/image-20220821092002572.png)

![image-20220821092012349](day06-WebRTC媒体协商/image-20220821092012349.png)

![image-20220821092017425](day06-WebRTC媒体协商/image-20220821092017425.png)



## 二、WebRTC生成SDP的过程

### 1、WebRTC如何确定SDP的媒体信息？

- WebRTC通过 `addTrack` 拿到媒体信息

![image-20220821102408714](day06-WebRTC媒体协商/image-20220821102408714.png)

### 2、`Addtrack` 具体做了什么事情呢？

![image-20220821102441787](day06-WebRTC媒体协商/image-20220821102441787.png)

![image-20220821102449496](day06-WebRTC媒体协商/image-20220821102449496.png)

![image-20220821102456496](day06-WebRTC媒体协商/image-20220821102456496.png)

### 3、从整体上看WebRTC的Session层

![image-20220821102517457](day06-WebRTC媒体协商/image-20220821102517457.png)

- 自己能绘制这样的图，就算掌握了
- 每一个 TtpTransceiver 就对应SDP中的一个 m 行
- RtpTransceiver 连着三端：上层的应用端、底层的传输端、中间的编解码器



### 4、下面几个问题解答？

![image-20220821102542271](day06-WebRTC媒体协商/image-20220821102542271.png)

![image-20220821102547759](day06-WebRTC媒体协商/image-20220821102547759.png)

![image-20220821102553061](day06-WebRTC媒体协商/image-20220821102553061.png)

### 5、编解码器信息的收集二

![image-20220821102610427](day06-WebRTC媒体协商/image-20220821102610427.png)

![image-20220821102614194](day06-WebRTC媒体协商/image-20220821102614194.png)

![image-20220821102619796](day06-WebRTC媒体协商/image-20220821102619796.png)

![image-20220821102624591](day06-WebRTC媒体协商/image-20220821102624591.png)

![image-20220821102630350](day06-WebRTC媒体协商/image-20220821102630350.png)

### 6、课程小节

- 前两个步骤都只是收集信息，什么时候开始将收集的信息生成SDP呢？

![image-20220821102639441](day06-WebRTC媒体协商/image-20220821102639441.png)

## 三、 createOffer源码分析

### 1、在创建offer的`之前`有一个特别重要的点，就是`生成证书`

![image-20220821102710268](day06-WebRTC媒体协商/image-20220821102710268.png)

### 2、WebRTC如何保证 createOffer必定在setLocalSDP之前执行呢？

- WebRTC内部是通过队列，来讲媒体协商的信息进行顺序化

![image-20220821102734876](day06-WebRTC媒体协商/image-20220821102734876.png)

- 四个媒体协商，都是有顺序的，就是通过上面的队列进行保证的

### 3、createOffer创建过程图，太棒了

![image-20220821102753302](day06-WebRTC媒体协商/image-20220821102753302.png)

![image-20220821102759479](day06-WebRTC媒体协商/image-20220821102759479.png)

![image-20220821102803055](day06-WebRTC媒体协商/image-20220821102803055.png)

### 4、createOffer的源码流程跟踪



![image-20220821102814134](day06-WebRTC媒体协商/image-20220821102814134.png)

![image-20220821102818413](day06-WebRTC媒体协商/image-20220821102818413.png)

![image-20220821102824286](day06-WebRTC媒体协商/image-20220821102824286.png)

![image-20220821102827462](day06-WebRTC媒体协商/image-20220821102827462.png)

![image-20220821102833661](day06-WebRTC媒体协商/image-20220821102833661.png)

![image-20220821102836761](day06-WebRTC媒体协商/image-20220821102836761.png)

### 四、 SetLocalDescription源码分析

#### 1、setLocalDescription的调用栈，图很棒啊

- 每一个transceiver都对应sdp中的一个m行

![image-20220821102855305](day06-WebRTC媒体协商/image-20220821102855305.png)

### 2、几个Channel的含义理解

- 这个图也太强了吧，是什么水平，能画出来这么帅的图

![image-20220821102908936](day06-WebRTC媒体协商/image-20220821102908936.png)



### 3、建立音视频流水线



![image-20220821102927936](day06-WebRTC媒体协商/image-20220821102927936.png)

### 4、Session层的理解

![image-20220821102935949](day06-WebRTC媒体协商/image-20220821102935949.png)



### 5、SetLocalDescription代码走读

![image-20220821102958257](day06-WebRTC媒体协商/image-20220821102958257.png)

![image-20220821103003230](day06-WebRTC媒体协商/image-20220821103003230.png)

![image-20220821103013018](day06-WebRTC媒体协商/image-20220821103013018.png)

![image-20220821103017745](day06-WebRTC媒体协商/image-20220821103017745.png)



## 四、candidte的收集

### 1、WebRTC什么时候开始收集candidate？

![image-20220821103050556](day06-WebRTC媒体协商/image-20220821103050556.png)

2、candidate收集过程的调用栈



- 这种方式去仔细阅读某个方法，是不是挺好的？

![image-20220821103102720](day06-WebRTC媒体协商/image-20220821103102720.png)

### 3、candidate收集过程的代码走读

![image-20220821103110610](day06-WebRTC媒体协商/image-20220821103110610.png)

![image-20220821103114334](day06-WebRTC媒体协商/image-20220821103114334.png)

![image-20220821103117676](day06-WebRTC媒体协商/image-20220821103117676.png)

![image-20220821103122612](day06-WebRTC媒体协商/image-20220821103122612.png)



### 4、生成SDP信息的位置在哪里？

![image-20220821103131120](day06-WebRTC媒体协商/image-20220821103131120.png)

### 5、JsepSessionDescription的类关系图？

![image-20220821103141613](day06-WebRTC媒体协商/image-20220821103141613.png)

### 5、WebRTC生成SDP信息的代码走读？

![image-20220821103149254](day06-WebRTC媒体协商/image-20220821103149254.png)

![image-20220821103151830](day06-WebRTC媒体协商/image-20220821103151830.png)

![image-20220821103155805](day06-WebRTC媒体协商/image-20220821103155805.png)

![image-20220821103159907](day06-WebRTC媒体协商/image-20220821103159907.png)



### 6、解析SDP的代码走读（和生成SDP类似，可以自己总结一个调用栈）

![image-20220821103208942](day06-WebRTC媒体协商/image-20220821103208942.png)

![image-20220821103212374](day06-WebRTC媒体协商/image-20220821103212374.png)

![image-20220821103215743](day06-WebRTC媒体协商/image-20220821103215743.png)

![image-20220821103220959](day06-WebRTC媒体协商/image-20220821103220959.png)



### 7、源码分析-createAnswer

- 生成Answer之前也要先创建证书

![image-20220821103307035](day06-WebRTC媒体协商/image-20220821103307035.png)![image-20220821103311725](day06-WebRTC媒体协商/image-20220821103311725.png)![image-20220821103318915](day06-WebRTC媒体协商/image-20220821103318915.png)![image-20220821103321550](day06-WebRTC媒体协商/image-20220821103321550.png)![image-20220821103328685](day06-WebRTC媒体协商/image-20220821103328685.png)

### 8、setRemoteDescription

![image-20220821103345839](day06-WebRTC媒体协商/image-20220821103345839.png)![image-20220821103348485](day06-WebRTC媒体协商/image-20220821103348485.png)![image-20220821103353549](day06-WebRTC媒体协商/image-20220821103353549.png)
