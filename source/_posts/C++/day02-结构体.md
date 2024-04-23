---
title: day02-结构体
date: 2024-04-22 07:28:30
tags: 
	- C++
categories: 
	- C++

---

## 一、结构体

> **结构体（Structure）**在C语言中是一种<font color="red">非常重要</font>的数据结构，它允许你将多种不同类型的数据项（称为成员）组合成一个单一类型。
>
> 这使得结构体<font color="red">非常适合</font>用来创建**复杂的数据模型**，结构体的使用可以大大提高程序的**清晰度**和**易维护性**。



#### 1、常见的基本类型所占用字节数

```c
char 				// 1个字节
uint8_t 		// 1个字节
short 			// 2个字节
int 				// 4个字节
float 			// 4个字节
long				// 8个字节
double		  // 8个字节
long long 	// 8个字节
```



#### 2、下面结构体，占几个字节？

```c
typedef struct {
    unsigned char event;
    unsigned char status;
    unsigned char reserved[1];
} SAvEvent;
```

- 占用 **3** 个字节



#### 3、下面结构体，占几个字节？

```c
typedef struct {
    unsigned char status;
    int event;
    unsigned char reserved[1];
} SAvEvent;
```

- 占用 **12** 个字节



#### 4、下面结构体，占几个字节？

```c
typedef struct {
    int event;
    unsigned char status;
    unsigned char reserved[1];
} SAvEvent;
```

- 占用 **8** 个字节

## 二、内存对齐

#### 1、为什么要内存对齐？

> **不同类型的数据访问速度最优化时，其地址往往需要满足一定的对齐要求。**
>
> 例如，某些硬件平台上，访问4字节整数（如 `int` 类型）的地址可能需要是4的倍数。如果数据被放置在满足这些要求的地址上，CPU的数据访问速度就可以最大化。



#### 2、如何进行内存对齐？

- **对齐规则：**编译器将按照结构体中<font color="red">最宽基本类型成员</font>的对齐要求，为整个结构体选择一个“对齐因子”（也称为“对齐量”）。整个结构体的起始地址会按照这个对齐因子来对齐。结构体内的每个成员也会根据其类型的对齐要求，在结构体内部相对起始地址进行对齐。
- **填充字节（Padding）:**为了满足内存对齐的要求，编译器可能在结构体的成员之间或结构体的<font color="red">末尾拖入未使用的空间</font>（称为“填充字节”），这可以确保每个成员都在对其对齐要求的地址上开始。



#### 3、举例说明？

```c
struct Example {
    char a;    // 占用1字节
    int b;     // 占用4字节，假设需要4字节对齐
    char c;    // 占用1字节
};

```

- `a` 直接放在结构体的开始。

- 在 `a` 和 `b` 之间插入3个填充字节，以确保 `b` 的地址是4的倍数，满足 `int` 类型的对齐要求。

- `c` 紧跟在 `b` 后面，但此时结构体的总大小为9字节。为了使结构体数组中每个元素的地址都满足最严格的对齐要求（在这里是4字节对齐），在结构体的末尾还会添加3个填充字节，**使得整个结构体的大小变为12字节**。



#### 4、如何优化上面的结构体？

- 了解内存对齐可以帮助你更好地设计结构体以减少内存空间的浪费。
- 一个常见的技巧是按照 <font color="red">从最宽到最窄</font> 的成员顺序声明结构体成员，这样可以减少填充字节的数量。

```c
struct Example {
    char a;    // 占用1字节
    char c;    // 占用1字节
    int b;     // 占用4字节，假设需要4字节对齐
};
```

- **整个结构体的大小变为 8 字节**。

#### 5、什么柔性数组成员？

```c
typedef struct {
    unsigned char status;
    unsigned char reserved[1];
    int event;
} SAvEvent;

typedef struct {
    unsigned int  channel;
    SAvEvent stEvent[1];
} SMsgAVIoctrlListEventResp;
```

- 在`SMsgAVIoctrlListEventResp`结构体中，`stEvent[1]`被定义为只有一个元素的`SAvEvent`类型数组。然而，在实际使用中，它通常意味着这个结构体将用作一个包含数量不确定的`SAvEvent`元素的容器。





## 三、在OC中常用的相关函数记录

#### 1、头部数据转换 `SMsgAVIoctrlHead` 结构的获取

```objective-c
- (void)dataChannel:(nonnull RTCDataChannel *)dataChannel didReceiveMessageWithBuffer:(nonnull RTCDataBuffer *)buffer {
//    DDLogMethod_KVS(@"");
//    DDLogDebug(@"%@ size=%ld",NSStringFromSelector(_cmd),buffer.data.length);
    SMsgAVIoctrlHead msgAVIoctrlHead;
    if (buffer.data.length < sizeof(SMsgAVIoctrlHead)) {
        NSLog(@"buffer.data.length < sizeof(SMsgAVIoctrlHead)");
        return;
    }
    NSData *headerData = [buffer.data subdataWithRange:NSMakeRange(0, sizeof(SMsgAVIoctrlHead))];
    [headerData getBytes:&msgAVIoctrlHead length:sizeof(SMsgAVIoctrlHead)];
    NSLog(@"didReceiveMessageWithBuffer:%d,req:%d,action:recv",msgAVIoctrlHead.command,msgAVIoctrlHead.seq);
    LdsKvsRequest *request;
    @synchronized (self) {
        request = self.cmdQueue[@(msgAVIoctrlHead.seq)];
//        [self.cmdQueue removeObjectForKey:@(msgAVIoctrlHead.seq)];
        if (msgAVIoctrlHead.command == IOTYPE_USER_IPCAM_HASLISTEVENT_RESP || msgAVIoctrlHead.command == IOTYPE_USER_IPCAM_LISTEVENT_RESP || msgAVIoctrlHead.command == IOTYPE_USER_IPCAM_LISTEVENT_V2_RESP) {
            //需要拼包
        } else {
            [self.cmdQueue removeObjectForKey:@(msgAVIoctrlHead.seq)];
        }
        
    }
    if (msgAVIoctrlHead.length <= 0 && msgAVIoctrlHead.command != IOTYPE_CMD_AVIO_CTRL_HEARTBEAT_RESP) { //如果心跳，length可能为空
        DDLogWarn(@"[webrtc channelCommand]%@ didReceiveMessageWithBuffer unknown %@ cmd:%d",self.kvsServerMgr.localClientSenderId,buffer.data,msgAVIoctrlHead.command);
        return;
    }
    
    DDLogInfo(@"[webrtc channelCommand]%@ recv remote message seq:%d,controlCMD:%X",self.kvsServerMgr.localClientSenderId,msgAVIoctrlHead.seq,msgAVIoctrlHead.command);
    NSData *recvIOCtrlData = [buffer.data subdataWithRange:NSMakeRange(headerData.length, buffer.data.length-headerData.length)];
}
```



#### 2、结构体尾随数据的解析

```objective-c
        case IOTYPE_USER_IPCAM_LISTEVENT_V2_RESP: {
            SMsgAVIoctrlListEventV2Resp resp;
            size_t respHeaderSize = sizeof(int)*2+sizeof(char)*4;
            
            memset(&resp, 0, sizeof(SMsgAVIoctrlListEventV2Resp));
            memcpy(&resp, recvIOCtrlData.bytes, sizeof(SMsgAVIoctrlListEventV2Resp));
            
            size_t eventV2Size = sizeof(lds_avio_event_v2_t);
            size_t eventV2HeaderSize = sizeof(STimeDay) + sizeof(char) * 4;
            lds_avio_event_v2_t eventV2;
            lds_avio_event_item_t eventV2Item;
            size_t eventV2ItemSize = sizeof(uint8_t) * 2 + sizeof(uint64_t);
            LdsKVSLogDebug(@"[avRecvIOCtrl]resp.total ---- %d resp.count =[%d]  savSize*resp.count =[%ld], endFlag:%d",resp.total,resp.count,eventV2Size*resp.count,resp.endflag);
            
            NSMutableArray *eventV2Array = @[].mutableCopy;
            size_t eventV2ParseIndex = respHeaderSize; //数据解析起始点
            if (resp.total >= 0) {
                for (int i=0; i<resp.count; i++) {
                    // 1.事件数据解析
                    memset(&eventV2, 0, eventV2Size);
                    memcpy(&eventV2, recvIOCtrlData.bytes+eventV2ParseIndex, eventV2Size);
                    size_t currentEventV2Size = eventV2HeaderSize + eventV2ItemSize * eventV2.itemCount;
                    
                    // 2.事件里面的触发源解析
                    NSMutableArray *eventV2ItemArray = @[].mutableCopy;
                    for (int j = 0; j < eventV2.itemCount; j++) {
                        memset(&eventV2Item, 0, eventV2ItemSize);
                        memcpy(&eventV2Item, recvIOCtrlData.bytes + eventV2ParseIndex + eventV2ItemSize * j , eventV2ItemSize);
                        
                        LdsPlaybackEventItem *eventItem = [LdsPlaybackEventItem eventItemWithData:eventV2Item];
                        [eventV2ItemArray addObject:eventItem];
                    }
                    
                    
                    PlaybackEvent *event = [PlaybackEvent eventWithEventV2:eventV2];
                    event.eventItemArray = [eventV2ItemArray copy];
                    [eventV2Array addObject:event];
                    
                    eventV2ParseIndex += currentEventV2Size;
                }
            }
            dispatch_async(dispatch_get_main_queue(), ^{
//                [request successWithResult:array];
                if (!request.result) {
                    request.result = @[].mutableCopy;
                }
                NSMutableArray *resultArray = request.result;
                [resultArray addObjectsFromArray:eventV2Array];
                NSLog(@"WebRTC:IOTYPE_USER_IPCAM_LISTEVENT_RESP:%@,count:%d",resultArray,resultArray.count);
                if (resp.endflag) {
                    [request successWithResult:resultArray];
                }
            });
            break;
        }
```

- 结构体尾随数组元素，对端传递来的数据，是没进行内存对齐的。比如10个字节构成Event，那么对端传递30字节，就表示3个Event。

#### 3、对方传递来字节序，如何比较好的调试打印？

```c
po [recvIOCtrlData subdataWithRange:NSMakeRange(12, 10)]
<e8070416 01173528 0100>
```

- 这样就可以对比字节序和结构体去计算数据了













