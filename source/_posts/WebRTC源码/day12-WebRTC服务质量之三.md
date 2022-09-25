---

title: day12-WebRTC服务质量之三、Pacer、JitterBufferr
categories:
  - WebRTC源码
tags:
  - WebRTC
date: 2022-09-15 20:41:31
---

## 三、认识Pacer

### 1、Pacer的作用是什么？

- 让数据在网络上发送得更平滑，防止因数据的突增导致网络发生拥塞。

![image-20220915073211215](day12-WebRTC服务质量之三/image-20220915073211215.png)

### 2、Pacer是什么时候创建的？

![image-20220915073223661](day12-WebRTC服务质量之三/image-20220915073223661.png)

- 这种流程图，也是超级棒啊

### 3、关于Pacer的两个重要函数、两种工作模式？

![image-20220915073238172](day12-WebRTC服务质量之三/image-20220915073238172.png)

![image-20220915073241618](day12-WebRTC服务质量之三/image-20220915073241618.png)

![image-20220915073245124](day12-WebRTC服务质量之三/image-20220915073245124.png)

![image-20220915073248137](day12-WebRTC服务质量之三/image-20220915073248137.png)

- WebRTC如何处理是否需要发送保活包？当没有音视频数据时，才需要发送保活包。

![image-20220915073255149](day12-WebRTC服务质量之三/image-20220915073255149.png)

![image-20220915073258438](day12-WebRTC服务质量之三/image-20220915073258438.png)

### 4、认识Pacer中非常重要的数据结构 RoundBobinPacketQueue。

![image-20220915073304626](day12-WebRTC服务质量之三/image-20220915073304626.png)

- 对这个结构体，在 12-20 小节中有详细介绍

![image-20220915073314869](day12-WebRTC服务质量之三/image-20220915073314869.png)

![image-20220915073317996](day12-WebRTC服务质量之三/image-20220915073317996.png)

![image-20220915073321456](day12-WebRTC服务质量之三/image-20220915073321456.png)

### 5、认识Pacer中的 `IntervalBudget` ，它的作用是什么？

- IntervalBudget的作用：控制每次发送数据码流的大小。
- 有了目标码率，为什么还要intervalBudget？
- 之所以要intervalBudget，是因为我们WebRTC的拥塞控制获取的目标码率是以【秒】为单位的。
- 而比如时间周期每5毫秒要发送多少数据量，就是由intervalBudget控制的。

![image-20220915073352591](day12-WebRTC服务质量之三/image-20220915073352591.png)

![image-20220915073355372](day12-WebRTC服务质量之三/image-20220915073355372.png)

![image-20220915073358116](day12-WebRTC服务质量之三/image-20220915073358116.png)

![image-20220915073401429](day12-WebRTC服务质量之三/image-20220915073401429.png)

![image-20220915073404013](day12-WebRTC服务质量之三/image-20220915073404013.png)

![image-20220915073406822](day12-WebRTC服务质量之三/image-20220915073406822.png)

![image-20220915073410056](day12-WebRTC服务质量之三/image-20220915073410056.png)

![image-20220915073413349](day12-WebRTC服务质量之三/image-20220915073413349.png)

![image-20220915073416157](day12-WebRTC服务质量之三/image-20220915073416157.png)

### 6、WebRTC如何向Pacer中插入数据包？

![image-20220915073423193](day12-WebRTC服务质量之三/image-20220915073423193.png)

![image-20220915073425901](day12-WebRTC服务质量之三/image-20220915073425901.png)

![image-20220915073430660](day12-WebRTC服务质量之三/image-20220915073430660.png)

![image-20220915073433856](day12-WebRTC服务质量之三/image-20220915073433856.png)

![image-20220915073436911](day12-WebRTC服务质量之三/image-20220915073436911.png)



## 四、JitterBuffer



### 1、认识JitterBuffer类的关系图？

![image-20220915073511274](day12-WebRTC服务质量之三/image-20220915073511274.png)

![image-20220915073514475](day12-WebRTC服务质量之三/image-20220915073514475.png)

> 收到视频数据后，
> ① 先给到 RtpVideoStreamReceiver 再交由 PacketBuffer 组成一帧数据， 然后还给 RtpVideoStreamReceiver；
>
> ②再由 RtpVideoStreamReceiver 把这一帧数据给到 RtpFrameReferenceFinder 来找这一帧的依赖帧，找到所有依赖帧后，再还给 RtpVideoStreamReceiver；
>
> ③ RtpVideoStreamReceiver 再把数据保存到 Internal::VideoReceiveStream 中，这里面的数据帧就是一个个可以进行解码的数据帧了。
>
> 【掌握】上面的架构一定要掌握，才能继续看实现细节。

![image-20220915073533358](day12-WebRTC服务质量之三/image-20220915073533358.png)

![image-20220915073537194](day12-WebRTC服务质量之三/image-20220915073537194.png)

![image-20220915073539861](day12-WebRTC服务质量之三/image-20220915073539861.png)

![image-20220915073542837](day12-WebRTC服务质量之三/image-20220915073542837.png)

![image-20220915073546596](day12-WebRTC服务质量之三/image-20220915073546596.png)

![image-20220915073549633](day12-WebRTC服务质量之三/image-20220915073549633.png)

![image-20220915073552549](day12-WebRTC服务质量之三/image-20220915073552549.png)

### 3、WebRTC中 如何查找完整帧的？

![image-20220915073600453](day12-WebRTC服务质量之三/image-20220915073600453.png)

![image-20220915073603727](day12-WebRTC服务质量之三/image-20220915073603727.png)

![image-20220915073606714](day12-WebRTC服务质量之三/image-20220915073606714.png)

![image-20220915073609613](day12-WebRTC服务质量之三/image-20220915073609613.png)

### 4、WebRTC中RtpFrameReferenceFinder的作用是什么？

- RtpFrameReferenceFinder的作用：查找解码帧的参考帧

![image-20220915073621322](day12-WebRTC服务质量之三/image-20220915073621322.png)

- 这个GOP就体现了一个GOP中可能有多个I帧，但是仅有一个IDR帧。
- 【注意】WebRTC编码出的数据流一般只包含I帧和P帧，通常不包含B帧。所以查找参考帧只要看前面一帧数据

![image-20220915073631046](day12-WebRTC服务质量之三/image-20220915073631046.png)

![image-20220915073634077](day12-WebRTC服务质量之三/image-20220915073634077.png)

![image-20220915073636853](day12-WebRTC服务质量之三/image-20220915073636853.png)

![image-20220915073640039](day12-WebRTC服务质量之三/image-20220915073640039.png)

![image-20220915073642840](day12-WebRTC服务质量之三/image-20220915073642840.png)

### 4、WebRTC中RtpFrameReferenceFinder的如何查找参考帧的？

![image-20220915073648981](day12-WebRTC服务质量之三/image-20220915073648981.png)

![image-20220915073651929](day12-WebRTC服务质量之三/image-20220915073651929.png)

![image-20220915073654592](day12-WebRTC服务质量之三/image-20220915073654592.png)

![image-20220915073657252](day12-WebRTC服务质量之三/image-20220915073657252.png)

![image-20220915073700529](day12-WebRTC服务质量之三/image-20220915073700529.png)

- 在实时通信流中，一般包含B帧，所以非关机键帧的参考帧就是前面一帧。

![image-20220915073707842](day12-WebRTC服务质量之三/image-20220915073707842.png)

### 5、WebRTC中的FrameBuffer

一旦进入FrameBuffer的帧，随时都可能被解码，播放。



![image-20220915073720871](day12-WebRTC服务质量之三/image-20220915073720871.png)

![image-20220915073723693](day12-WebRTC服务质量之三/image-20220915073723693.png)

![image-20220915073726826](day12-WebRTC服务质量之三/image-20220915073726826.png)

![image-20220915073730929](day12-WebRTC服务质量之三/image-20220915073730929.png)

![image-20220915073733969](day12-WebRTC服务质量之三/image-20220915073733969.png)







