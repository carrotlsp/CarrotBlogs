---

title: day09-音频引擎
categories:
  - WebRTC源码
tags:
  - WebRTC
date: 2022-08-28 20:41:31
---

### 1、章节概述，知道学完本章能知道些什么？

![image-20220920073222523](day09-音频引擎/image-20220920073222523.png)

![image-20220920073227377](day09-音频引擎/image-20220920073227377.png)

### 2、音频引擎类关系图？

![image-20220920073243176](day09-音频引擎/image-20220920073243176.png)

- `adm`：AudioDeviceModule，通过它我们可以从设备中读取数据，还可以将数据写入到扬声器中。
- `encoder_factory`：我们最终的编码器就是通过它创建出来的。
- `decoder_factory`：我们最终的解码器就是通过它创建出来的。
- `audio_mixer`：混音器，是将多路流混成一路流，最终通过扬声器播放出来。
- `apm`：处理三A问题，自动增益、降噪、回音消除。
- `audio_state`：字面意思是音频状态的管理，实际上是对音频流的管理。
- `send_codecs`：是一个列表，它里面包含了WebRTC所支持的所有编码器。列表中的每一项就是一个编码器，比如Opus
- `revc_codecs`：是一个列表，它里面包含了WebRTC所支持的所有解码器。列表中的每一项就是一个解码器。
- `channels`：也是一个列表，包含了WebRTC所有使用的通道，其中每一项是一个WebRtcVoiceMediaChannel，表示在SDP中的每一个m行都是一个channel，有音频channel、视频channel、数据channel。
- `WebRtcVoiceEngine`是一个全局的，所以不管有多少个peerid，最终形成了多少路流，每一个流都会是一个channel，它会将所有的流都汇总到channels中
- 【这数据架构上的东西，一定要熟记于心】

### 3、音频引擎数据流转图？

![image-20220920073402094](day09-音频引擎/image-20220920073402094.png)

- WebRTC分三层：Session层、引擎层、设备层。其中引擎层是最为关键的一层。
- 【这幅流程图，对应 9-2有详细讲解，要能自己完全说出来整个流程】
- 对于AudioEngine和VideoEngine来讲是全局唯一的

### 4、创建音频引擎的过程？

![image-20220920073423595](day09-音频引擎/image-20220920073423595.png)

- 入口函数：initializePeerConnection
- MediaEngineDepenDecies：是一个参数合集，将参数打包到一起，传递方便，扩展性好。
- 【这个图也要能自说自话，在9-3章节】

- 然后根据上面的流程图，去走一遍代码流程。

![image-20220920073440075](day09-音频引擎/image-20220920073440075.png)



### 5、音频引擎初始化的过程？

![image-20220920073448676](day09-音频引擎/image-20220920073448676.png)

- 对上述流程进行代码走读

![image-20220920073501239](day09-音频引擎/image-20220920073501239.png)

- 重点是【WebRTC是如何收集音频解码器的】

### 6、AduioState对象的作用是什么？创建过程是怎么样的？

- AudioState作用：从名字上看是控制音频状态的，实际上是用来控制音频流的。

- 里面主要有两个模块ADM、AudioTransport



![image-20220920073523888](day09-音频引擎/image-20220920073523888.png)

![image-20220920073526911](day09-音频引擎/image-20220920073526911.png)

![image-20220920073529982](day09-音频引擎/image-20220920073529982.png)

![image-20220920073533262](day09-音频引擎/image-20220920073533262.png)

- 【自己源码分析，AudioState的创建过程】

![image-20220920073543705](day09-音频引擎/image-20220920073543705.png)

### 7、WebRTC如何获取采集到的音频数据？

![image-20220920073550764](day09-音频引擎/image-20220920073550764.png)

- 无论是给扬声器的播放的声音，还是从麦克风采集到的声音，都有会经过AudioDeviceBuffer这个过程。

![image-20220920073559878](day09-音频引擎/image-20220920073559878.png)

![image-20220920073602713](day09-音频引擎/image-20220920073602713.png)

![image-20220920073605521](day09-音频引擎/image-20220920073605521.png)

- 【源码分析】获取采集到的数据

![image-20220920073613059](day09-音频引擎/image-20220920073613059.png)

![image-20220920073616593](day09-音频引擎/image-20220920073616593.png)

![image-20220920073620207](day09-音频引擎/image-20220920073620207.png)

> 总结：①创建AudioDeviceBuffer ②会将AudioTransport注册到AudioDeviceBuffer中 ③当通过系统的音频采集线程采集到音频后，会交由AudioDeviceBuffer处理，会通过回调对象将数据回调给音频引擎层；

### 8、Channel、Stream与编码器之间的关系？

- 在Session层(也就是api层)，一个stream里面可以有多少track。
- 一个Session层的track，对应引擎层中的channel，不同于在引擎层中会把音频、视频的cahnnel分开来存储。
- 在引擎层中的channel又可以包括很多个stream(这里的stream和session层中的stream不是同一个概念)，既有发送的stream又有接收的stream。
- 所以说引擎层中的channel是双向的，既有流入的stram也有发出的stream。

![image-20220920073651732](day09-音频引擎/image-20220920073651732.png)

- 在call层，又包含了channelSend，它的作用是用于连接音频编码器的。
- 在call层，又包含了channelReceive，它的作用是用于连接音频解码器的。
- 引擎层的stream和Call层的stream是一一对应的。

### 9、ChannelSend与音频编码器是如何连接的？

![image-20220920073707642](day09-音频引擎/image-20220920073707642.png)

![image-20220920073711036](day09-音频引擎/image-20220920073711036.png)

- ChannelReceive 与 音频解码器的关系？

- 【这两张图核心轨迹关系图，需要掌握】

### 10、WebRTC创建音频编码器？

![image-20220920073736466](day09-音频引擎/image-20220920073736466.png)

![image-20220920073739456](day09-音频引擎/image-20220920073739456.png)

![image-20220920073742368](day09-音频引擎/image-20220920073742368.png)

![image-20220920073745931](day09-音频引擎/image-20220920073745931.png)

- 【借助上面流程图，对源码进行阅读理解】

![image-20220920073758383](day09-音频引擎/image-20220920073758383.png)

![image-20220920073810109](day09-音频引擎/image-20220920073810109.png)

- 找到第一个最合适的codec就返回，所以我们需要把优先选择的解码器名字，放在前面。

![image-20220920073818218](day09-音频引擎/image-20220920073818218.png)

### 11、WebRTC是如何创建Opus编码器的？

![image-20220920073825731](day09-音频引擎/image-20220920073825731.png)

- webrtc_opus：当我们使用单通道或者双通道进行通信时，选择webrtc_opus。（这也是最常见的选择）
- webrtc_multiopus：超过两个通道的编码器，就选择这个了。

![image-20220920073835571](day09-音频引擎/image-20220920073835571.png)

![image-20220920073838852](day09-音频引擎/image-20220920073838852.png)

- 【根据上面的逻辑，进行源码解读】

![image-20220920073847451](day09-音频引擎/image-20220920073847451.png)

- 【如果追踪不到，可以结合9-9进行结合理解】

### 12、WebRTC的音频是如何编码的？

![image-20220920073900297](day09-音频引擎/image-20220920073900297.png)

- 根据上面的流程图，【源码分析】结合9-10理解音频编码的过程。

![image-20220920073909672](day09-音频引擎/image-20220920073909672.png)

### 13、音频解码器的创建过程？

![image-20220920073916608](day09-音频引擎/image-20220920073916608.png)

![image-20220920073919844](day09-音频引擎/image-20220920073919844.png)



![image-20220920073923221](day09-音频引擎/image-20220920073923221.png)

![image-20220920073926264](day09-音频引擎/image-20220920073926264.png)

![image-20220920073929581](day09-音频引擎/image-20220920073929581.png)

- NetEq与Decoder之间的关系

![image-20220920073936703](day09-音频引擎/image-20220920073936703.png)

- 【源码分析】根据上面介绍的知识，对源码进行跟踪，具体可以结合 9-11 进行理解

![image-20220920073945982](day09-音频引擎/image-20220920073945982.png)

### 14、WebRTC音频解码的过程？

![image-20220920073952558](day09-音频引擎/image-20220920073952558.png)

![image-20220920073956061](day09-音频引擎/image-20220920073956061.png)

- 主要看右边的过程
- 根据上面的图，进行【源码分析】

![image-20220920074014812](day09-音频引擎/image-20220920074014812.png)









