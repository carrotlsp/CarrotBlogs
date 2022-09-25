---

title: day11-深入理解WebRTC网络传输之二
categories:
  - WebRTC源码
tags:
  - WebRTC
date: 2022-09-04 20:41:31
---

## 二、ICE、Candidate

### 1、ICE的全称是什么？作用是什么？

- `Interactive Connectivity Establishment`

![image-20220904160012808](day11-深入理解WebRTC网络传输之二/image-20220904160012808.png)

![image-20220904160019088](day11-深入理解WebRTC网络传输之二/image-20220904160019088.png)

- ICE架构图，三台机器所有的连接方式

![image-20220904160029251](day11-深入理解WebRTC网络传输之二/image-20220904160029251.png)

- 图中包括①内网线路 ②p2p线路 ③TURN线路

![image-20220904160039515](day11-深入理解WebRTC网络传输之二/image-20220904160039515.png)

- 先收集candidate 2、再对candidate进行排序 3、最后进行连通性检测

### 2、什么是candidate？

- 每一个candate是一个网络地址信息
- 它包括：协议、IP、端口 和 类型

![image-20220904160159233](day11-深入理解WebRTC网络传输之二/image-20220904160159233.png)

### 3、对于WebRTC来说，它是如何收集candidate的呢？

![image-20220904160222201](day11-深入理解WebRTC网络传输之二/image-20220904160222201.png)

> 什么是网卡？
>
> 为什么有时候一个电脑只有一个网卡
>
> 为什么有时候一个电脑有多个网卡，对应有多个IP地址呢？
>
> Host类型的地址：从网卡上收集到的就是Host类型的地址
>
> Sflex类型：就是通过NAT对外部暴露的IP地址
>
> Relay地址：就是通过NAT无法进行穿越连接的时候，就需要中转服务器地址，这就是Relay地址



### 4、对于WebRTC来讲，candidate有哪几种类型以及优先级如何呢？

![image-20220904160425027](day11-深入理解WebRTC网络传输之二/image-20220904160425027.png)

- 重点理解什么是 `Prflx` 对端映射候选者地址
- `Prflx` 对端映射候选者地址：当我们获取另外三种地址后，需要通过信令服务器把这三种地址交给对端；另一端获取到这些candidate信息之后，就可以尝试对我们进行连接；在连接的过程中如果发现两端直接存在NAT，并且对端给我们返回的源地址是一个未知地址，这就说明对端在穿越NAT的过程中，NAT又给它重新分配了一个地址，这就叫做`Prflx地址`
- candidate优先级：Host > Srflx > Prflx > Relay

![image-20220904160444009](day11-深入理解WebRTC网络传输之二/image-20220904160444009.png)

### 4、在了解WebRTC具体如何收集Candidate之前，我们需要认识一个重要的类 `PortAllocator`

- PortAllocator作用：就像它的命名一般，是用于分配端口号的

![image-20220904160508477](day11-深入理解WebRTC网络传输之二/image-20220904160508477.png)

![image-20220904160512502](day11-深入理解WebRTC网络传输之二/image-20220904160512502.png)

![image-20220904160515783](day11-深入理解WebRTC网络传输之二/image-20220904160515783.png)

![image-20220904160518732](day11-深入理解WebRTC网络传输之二/image-20220904160518732.png)

![image-20220904160521948](day11-深入理解WebRTC网络传输之二/image-20220904160521948.png)

![image-20220904160525860](day11-深入理解WebRTC网络传输之二/image-20220904160525860.png)

![image-20220904160529949](day11-深入理解WebRTC网络传输之二/image-20220904160529949.png)

![image-20220904160533136](day11-深入理解WebRTC网络传输之二/image-20220904160533136.png)

![image-20220904160536403](day11-深入理解WebRTC网络传输之二/image-20220904160536403.png)

![image-20220904160540280](day11-深入理解WebRTC网络传输之二/image-20220904160540280.png)

![image-20220904160543543](day11-深入理解WebRTC网络传输之二/image-20220904160543543.png)

### 5、认识PortAllocatorSession对象？

![image-20220904160551975](day11-深入理解WebRTC网络传输之二/image-20220904160551975.png)



![image-20220904160555225](day11-深入理解WebRTC网络传输之二/image-20220904160555225.png)

![image-20220904160559422](day11-深入理解WebRTC网络传输之二/image-20220904160559422.png)

![image-20220904160603348](day11-深入理解WebRTC网络传输之二/image-20220904160603348.png)

![image-20220904160607620](day11-深入理解WebRTC网络传输之二/image-20220904160607620.png)

![image-20220904160610769](day11-深入理解WebRTC网络传输之二/image-20220904160610769.png)

![image-20220904160614583](day11-深入理解WebRTC网络传输之二/image-20220904160614583.png)

### 6、认识 AllocationSequence 对象的创建过程？

![image-20220904160626339](day11-深入理解WebRTC网络传输之二/image-20220904160626339.png)

![image-20220904160635083](day11-深入理解WebRTC网络传输之二/image-20220904160635083.png)

![image-20220904160638993](day11-深入理解WebRTC网络传输之二/image-20220904160638993.png)

![image-20220904160643002](day11-深入理解WebRTC网络传输之二/image-20220904160643002.png)

![image-20220904160646322](day11-深入理解WebRTC网络传输之二/image-20220904160646322.png)

![image-20220904160657401](day11-深入理解WebRTC网络传输之二/image-20220904160657401.png)

![image-20220904160700937](day11-深入理解WebRTC网络传输之二/image-20220904160700937.png)

![image-20220904160704028](day11-深入理解WebRTC网络传输之二/image-20220904160704028.png)

![image-20220904160707352](day11-深入理解WebRTC网络传输之二/image-20220904160707352.png)

![image-20220904160710680](day11-深入理解WebRTC网络传输之二/image-20220904160710680.png)

![image-20220904160714858](day11-深入理解WebRTC网络传输之二/image-20220904160714858.png)

- AllocationSequence作用：就是用于创建Candidate

### 7、WebRTC如何收集 `Candidate` ？

![image-20220904160728180](day11-深入理解WebRTC网络传输之二/image-20220904160728180.png)

![image-20220904160732083](day11-深入理解WebRTC网络传输之二/image-20220904160732083.png)

![image-20220904160734948](day11-深入理解WebRTC网络传输之二/image-20220904160734948.png)

![image-20220904160739031](day11-深入理解WebRTC网络传输之二/image-20220904160739031.png)

![image-20220904160742940](day11-深入理解WebRTC网络传输之二/image-20220904160742940.png)

![image-20220904160746554](day11-深入理解WebRTC网络传输之二/image-20220904160746554.png)

![image-20220904160751563](day11-深入理解WebRTC网络传输之二/image-20220904160751563.png)

![image-20220904160755088](day11-深入理解WebRTC网络传输之二/image-20220904160755088.png)

![image-20220904160758357](day11-深入理解WebRTC网络传输之二/image-20220904160758357.png)

![image-20220904160802530](day11-深入理解WebRTC网络传输之二/image-20220904160802530.png)

### 8、获取本地Candidate的过程？

![image-20220904160811744](day11-深入理解WebRTC网络传输之二/image-20220904160811744.png)

![image-20220904160815304](day11-深入理解WebRTC网络传输之二/image-20220904160815304.png)

![image-20220904160818666](day11-深入理解WebRTC网络传输之二/image-20220904160818666.png)

![image-20220904160821697](day11-深入理解WebRTC网络传输之二/image-20220904160821697.png)

![image-20220904160824796](day11-深入理解WebRTC网络传输之二/image-20220904160824796.png)

![image-20220904160828332](day11-深入理解WebRTC网络传输之二/image-20220904160828332.png)

![image-20220904160832009](day11-深入理解WebRTC网络传输之二/image-20220904160832009.png)

![image-20220904160835381](day11-深入理解WebRTC网络传输之二/image-20220904160835381.png)

![image-20220904160838751](day11-深入理解WebRTC网络传输之二/image-20220904160838751.png)



### 