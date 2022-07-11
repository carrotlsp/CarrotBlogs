---
title: day17-【实战】iOS端的WebRTC代码实现思路
categories:
  - WebRTC入门
tags:
  - 音视频
  - WebRTC
date: 2022-06-18 07:37:21
---





## 一、单方面准备阶段（设备A）

### 1、创建 `RTCPeerConnectionFactory` 工厂类

```objective-c
- (void)createPeerConnectionFactory {
    //设置SSL传输
    [RTCPeerConnectionFactory initialize];
    
    //如果点对点工厂为空
    if (!factory)
    {
        RTCDefaultVideoDecoderFactory* decoderFactory = [[RTCDefaultVideoDecoderFactory alloc] init];
        RTCDefaultVideoEncoderFactory* encoderFactory = [[RTCDefaultVideoEncoderFactory alloc] init];
        NSArray* codecs = [encoderFactory supportedCodecs];
        [encoderFactory setPreferredCodec:codecs[2]];
        
        factory = [[RTCPeerConnectionFactory alloc] initWithEncoderFactory: encoderFactory
                                                            decoderFactory: decoderFactory];
    }
}
```



### 2、初始化本地音视频流

```objective-c
- (void) captureLocalMedia {
    NSDictionary* mandatoryConstraints = @{};
    RTCMediaConstraints* constraints =
    [[RTCMediaConstraints alloc] initWithMandatoryConstraints:mandatoryConstraints
                                          optionalConstraints:nil];
    
    RTCAudioSource* audioSource = [factory audioSourceWithConstraints: constraints];
    //self.audioTrack = [factory audioTrackWithTrackId:@"ARDAMSa0"];
    audioTrack = [factory audioTrackWithSource:audioSource trackId:@"ADRAMSa0"];
    
    NSArray<AVCaptureDevice* >* captureDevices = [RTCCameraVideoCapturer captureDevices];
    AVCaptureDevicePosition position = AVCaptureDevicePositionFront;
    AVCaptureDevice* device = captureDevices[0];
    for (AVCaptureDevice* obj in captureDevices) {
        if (obj.position == position) {
            device = obj;
            break;
        }
    }
    
    //检测摄像头权限
    AVAuthorizationStatus authStatus = [AVCaptureDevice authorizationStatusForMediaType:AVMediaTypeVideo];
    if(authStatus == AVAuthorizationStatusRestricted || authStatus == AVAuthorizationStatusDenied) {
        NSLog(@"相机访问受限");
        return;
    }
    
    if (device) {
        RTCVideoSource* videoSource = [factory videoSource];
        capture = [[RTCCameraVideoCapturer alloc] initWithDelegate:videoSource];
        AVCaptureDeviceFormat* format = [[RTCCameraVideoCapturer supportedFormatsForDevice:device] lastObject];
        CGFloat fps = [[format videoSupportedFrameRateRanges] firstObject].maxFrameRate;
        videoTrack = [factory videoTrackWithSource:videoSource trackId:@"ARDAMSv0"];
        self.localVideoView.captureSession = capture.captureSession;
        [capture startCaptureWithDevice:device
                                     format:format
                                        fps:fps];
        
    }
}
```

- 这时候屏幕上应该可以看到自己的视频流在播放了。

### 3、初始化SocketIO，并 `connect` 连接到信令服务器

```objective-c
- (void)createConnect: (NSString*) addr {
    NSLog(@"the server addr is %@", addr);
    /*
     log 是否打印日志
     forcePolling  是否强制使用轮询
     reconnectAttempts 重连次数，-1表示一直重连
     reconnectWait 重连间隔时间
     forceWebsockets 是否强制使用websocket
     */
    NSURL* url = [[NSURL alloc] initWithString:addr];
    manager = [[SocketManager alloc] initWithSocketURL:url
                                                config:@{@"log": @YES,
                                                         @"forcePolling":@YES,
                                                         @"forceWebsockets":@NO,
                                                         @"reconnectAttempts":@(5),
                                                         @"reconnectWait":@(1)
                                                         }];
    socket = manager.defaultSocket;
    
    [socket on:@"connect" callback:^(NSArray* data, SocketAckEmitter* ack) {
        NSLog(@"socket connected");
        [self.delegate connected];
    }];
    
    [socket on:@"error" callback:^(NSArray* data, SocketAckEmitter* ack) {
        NSLog(@"socket connect_error");
        [self.delegate connect_error];
    }];
    
    [socket on:@"reconnectAttempt" callback:^(NSArray* data, SocketAckEmitter* ack) {
        NSLog(@"socket reconnectAttempt");
        [self.delegate reconnectAttempt];
    }];
   
    [socket on:@"joined" callback:^(NSArray * data, SocketAckEmitter * ack) {
        NSString* room = [data objectAtIndex:0];
        
        NSLog(@"joined room(%@)", room);
        
        [self.delegate joined:room];
        
    }];
    
    [socket on:@"leaved" callback:^(NSArray * data, SocketAckEmitter * ack) {
        NSString* room = [data objectAtIndex:0];
        
        NSLog(@"leaved room(%@)", room);
        
        [self.delegate leaved:room];
    }];
    
    [socket on:@"full" callback:^(NSArray * data, SocketAckEmitter * ack) {
        NSString* room = [data objectAtIndex:0];
        
        NSLog(@"room(%@) is full", room);
        [self.delegate full:room];
    }];
    
    [socket on:@"otherjoin" callback:^(NSArray * data, SocketAckEmitter * ack) {
        NSString* room = [data objectAtIndex:0];
        NSString* uid = [data objectAtIndex:1];
        
        NSLog(@"other user(%@) has been joined into room(%@)", room, uid);
        [self.delegate otherjoin:room User:uid];
    }];
    
    [socket on:@"bye" callback:^(NSArray * data, SocketAckEmitter * ack) {
        NSString* room = [data objectAtIndex:0];
        NSString* uid = [data objectAtIndex:1];
        
        NSLog(@"user(%@) has leaved from room(%@)", room, uid);
        [self.delegate byeFrom:room User:uid];
    }];
    
    [socket on:@"message" callback:^(NSArray * data, SocketAckEmitter * ack){
        NSString* room = [data objectAtIndex:0];
        NSDictionary* msg = [data objectAtIndex:1];
        
        NSLog(@"onMessage, room(%@), data(%@)", room, msg);
        
        NSString* type = msg[@"type"];
        if( [type isEqualToString:@"offer"]){
            [self.delegate offer: room message: msg];
        }else if( [type isEqualToString:@"answer"]){
            [self.delegate answer:room message: msg];
        }else if( [type isEqualToString:@"candidate"]){
            [self.delegate candidate:room message: msg];
        }else {
            NSLog(@"the msg is invalid!");
        }
    }];
    
    // 连接超时时间设置为3秒
    [socket connectWithTimeoutAfter: 3.0 withHandler:^(void){
        NSLog(@"socket connect_timeout 3.0s");
        [self.delegate connect_timeout];
    }];
}
```



### 4、设备A发送 `join` 指令给服务器，并收到 `joined` 指令后，会创建 `RTCPeerConnection` 对象

```objective-c
- (RTCPeerConnection *)createPeerConnection {
    //得到ICEServer
    if (!ICEServers) {
        ICEServers = [NSMutableArray array];
        [ICEServers addObject:[self defaultSTUNServer]];
    }

    //用工厂来创建连接
    RTCConfiguration* configuration = [[RTCConfiguration alloc] init];
    [configuration setIceServers:ICEServers];
    RTCPeerConnection* conn = [factory
                                     peerConnectionWithConfiguration:configuration
                                                         constraints:[self defaultPeerConnContraints]
                                                            delegate:self];
    
    NSArray<NSString*>* mediaStreamLabels = @[@"ARDAMS"];
    [conn addTrack:videoTrack streamIds:mediaStreamLabels];
    [conn addTrack:audioTrack streamIds:mediaStreamLabels];

    return conn;
}
```



## 二、有另一个设备（设备B）加入房间阶段 - 媒体协商

### 1、A端会触发 `otherjoin` 信令，此刻有两个peerConnection，进入媒体协商阶段（创建offer、设置localOffer、发送offer）

```objective-c
// 约束信息
- (RTCMediaConstraints*)defaultPeerConnContraints {
    RTCMediaConstraints* mediaConstraints =
    [[RTCMediaConstraints alloc] initWithMandatoryConstraints:@{
                                                                kRTCMediaConstraintsOfferToReceiveAudio:kRTCMediaConstraintsValueTrue,
                                                                kRTCMediaConstraintsOfferToReceiveVideo:kRTCMediaConstraintsValueTrue
                                                                }
                                          optionalConstraints:@{ @"DtlsSrtpKeyAgreement" : @"true" }];
    return mediaConstraints;
}

// 创建offer
- (void)doStartCall {
    NSLog(@"Start Call, Wait ...");
    [self addLogToScreen: @"Start Call, Wait ..."];
    if (!peerConnection) {
        peerConnection = [self createPeerConnection];
    }
    
    [peerConnection offerForConstraints:[self defaultPeerConnContraints]
                      completionHandler:^(RTCSessionDescription * _Nullable sdp, NSError * _Nullable error) {
                          if(error){
                              NSLog(@"Failed to create offer SDP, err=%@", error);
                          } else {
                              __weak RTCPeerConnection* weakPeerConnction = self->peerConnection;
                              [self setLocalOffer: weakPeerConnction withSdp: sdp];
                          }
                      }];
}

// 发送offer给对端
- (void)setLocalOffer:(RTCPeerConnection*)pc withSdp:(RTCSessionDescription*) sdp{
    
    [pc setLocalDescription:sdp completionHandler:^(NSError * _Nullable error) {
        if (!error) {
            NSLog(@"Successed to set local offer sdp!");
        }else{
            NSLog(@"Failed to set local offer sdp, err=%@", error);
        }
    }];
    
    __weak NSString* weakMyRoom = myRoom;
    dispatch_async(dispatch_get_main_queue(), ^{
        
        NSDictionary* dict = [[NSDictionary alloc] initWithObjects:@[@"offer", sdp.sdp]
                                                           forKeys: @[@"type", @"sdp"]];
        
        [[SignalClient getInstance] sendMessage: weakMyRoom
                                        withMsg: dict];
    });
}
```



### 2、B端收到Offer后，会创建answer，并且发送给A端；A端收到answer会触发如下流程

```objective-c
// A端收到来自B端的Answer，并进行设置
- (void)answer:(NSString *)room message:(NSDictionary *)dict {
    NSLog(@"have received a answer message %@", dict);
    
    NSString *remoteAnswerSdp = dict[@"sdp"];
    RTCSessionDescription *remoteSdp = [[RTCSessionDescription alloc]
                                           initWithType:RTCSdpTypeAnswer
                                                    sdp: remoteAnswerSdp];
    [peerConnection setRemoteDescription:remoteSdp
                       completionHandler:^(NSError * _Nullable error) {
        if(!error){
            NSLog(@"Success to set remote Answer SDP");
        }else{
            NSLog(@"Failure to set remote Answer SDP, err=%@", error);
        }
    }];
}
```



## 三、有另一个设备（设备B）加入房间阶段 - 快速ICE阶段

### 1、创建peerConnection并且设置ICEServer之后，WebRTC就会去收集本设备的candidate信息，并触发如下回调

```objective-c
/** 默默收集candidate信息，然后通过信令服务器发送给对端 */
/** New ice candidate has been found. */
- (void)peerConnection:(RTCPeerConnection *)peerConnection
didGenerateIceCandidate:(RTCIceCandidate *)candidate{
    NSLog(@"%s",__func__);
    
    NSString* weakMyRoom = myRoom;
    dispatch_async(dispatch_get_main_queue(), ^{
    
        NSDictionary* dict = [[NSDictionary alloc] initWithObjects:@[@"candidate",
                                                                [NSString stringWithFormat:@"%d", candidate.sdpMLineIndex],
                                                                candidate.sdpMid,
                                                                candidate.sdp]
                                                           forKeys:@[@"type", @"label", @"id", @"candidate"]];
    
        [[SignalClient getInstance] sendMessage: weakMyRoom
                                    withMsg:dict];
    });
}
```



### 2、对端收到信令服务器转发而来的candidate，会触发如下操作，完成ICE的candidate交互流程。在所有candidate交互完成后，会自动进入ICE阶段

```objective-c
//收到信令服务器转发而来的candidate
- (void)candidate:(NSString *)room message:(NSDictionary *)dict {
    NSLog(@"have received a message %@", dict);
    
    NSString* desc = dict[@"sdp"];
    NSString* sdpMLineIndex = dict[@"label"];
    int index = [sdpMLineIndex intValue];
    NSString* sdpMid = dict[@"id"];
    
    
    RTCIceCandidate *candidate = [[RTCIceCandidate alloc] initWithSdp:desc
                                                        sdpMLineIndex:index
                                                               sdpMid:sdpMid];;
    [peerConnection addIceCandidate:candidate];
}
```



- 至此 `媒体协商` 和 `ICE` 流程都已经完成



## 四、媒体流传输阶段

### 1、收到数据流如何展示？

```objective-c
// 创建remoteTrack播放的视图
- (void)viewDidLoad {
    [super viewDidLoad];
    self.remoteVideoView = [[RTCEAGLVideoView alloc] initWithFrame:self.view.bounds];
    self.remoteVideoView.delegate = self;
    [self.view addSubview:self.remoteVideoView];
 }

/** 当接收到远端的数据流时，进行播放 */
/** Called when a receiver and its track are created. */
- (void)peerConnection:(RTCPeerConnection *)peerConnection
        didAddReceiver:(RTCRtpReceiver *)rtpReceiver
               streams:(NSArray<RTCMediaStream *> *)mediaStreams {
    NSLog(@"%s",__func__);
    
    RTCMediaStreamTrack* track = rtpReceiver.track;
    if([track.kind isEqualToString:kRTCMediaStreamTrackKindVideo]){
        if(!self.remoteVideoView){
            NSLog(@"error:remoteVideoView have not been created!");
            return;
        }
        
        remoteVideoTrack = (RTCVideoTrack*)track;
        [remoteVideoTrack addRenderer: self.remoteVideoView];
    }
}
```





## 五、核心代码记录

### 1、实现一个房间加入的视图

```objective-c
#import "ViewController.h"
#import "CallViewController.h"
#import "SignalClient.h"

@interface ViewController ()
{
    SignalClient *sigclient;
}

@property (strong, nonatomic) UILabel* addrLabel;
@property (strong, nonatomic) UITextField* addr;

@property (strong, nonatomic) UILabel* roomLabel;
@property (strong, nonatomic) UITextField* room;

@property (strong, nonatomic) UIButton* joinBtn;

@property (strong, nonatomic) CallViewController* call;

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.

    CGFloat width = self.view.bounds.size.width;
    
    self.addrLabel = [[UILabel alloc] init];
    [self.addrLabel setText:@"ADDR:"];
    [self.addrLabel setFrame:CGRectMake(20, 100, 60, 40)];
    [self.view addSubview:self.addrLabel];
    
    self.addr = [[UITextField alloc] init];
    [self.addr setText:@"https://webrtc.runduck.cn"];
    [self.addr setFrame:CGRectMake(80, 100, width-100, 40)];
    [self.addr setTextColor:[UIColor blackColor]];
    [self.addr setBorderStyle:UITextBorderStyleRoundedRect];
    [self.addr setEnabled:TRUE];
    [self.view addSubview:self.addr];
    
    self.roomLabel = [[UILabel alloc] init];
    [self.roomLabel setText:@"ROOM:"];
    [self.roomLabel setFrame:CGRectMake(20, 150, 60, 40)];
    [self.view addSubview:self.roomLabel];
    
    self.room = [[UITextField alloc] init];
    [self.room setText:@"123123"];
    [self.room setFrame:CGRectMake(80, 150, width-100, 40)];
    [self.room setTextColor:[UIColor blackColor]];
    [self.room setBorderStyle:UITextBorderStyleRoundedRect];
    [self.room setEnabled:TRUE];
    [self.view addSubview:self.room];
    
    self.joinBtn = [[UIButton alloc] init];
    [self.joinBtn setTitleColor:[UIColor whiteColor] forState:UIControlStateNormal];
    [self.joinBtn setTintColor:[UIColor whiteColor]];
    [self.joinBtn setTitle:@"join" forState:UIControlStateNormal];
    [self.joinBtn setBackgroundColor:[UIColor grayColor]];
    [self.joinBtn setShowsTouchWhenHighlighted:YES];
    [self.joinBtn setFrame:CGRectMake(40, 200, width-80, 40)];
    
    [self.joinBtn addTarget:self action:@selector(btnClick:) forControlEvents:UIControlEventTouchUpInside];
    
    [self.view addSubview:self.joinBtn];
}

/** 点击空白处回收键盘 */
- (void)touchesBegan:(NSSet<UITouch *> *)touches   withEvent:(UIEvent *)event
{
    [self.view endEditing:YES];
}

- (void)btnClick:(UIButton*)sender{
    NSLog(@"on click!");

    self.call = [[CallViewController alloc] initAddr:self.addr.text withRoom: self.room.text];
    [self.call.view setFrame:self.view.bounds];
    [self.call.view setBackgroundColor:[UIColor whiteColor]];

    [self addChildViewController:self.call];
    [self.call didMoveToParentViewController:self];
    
    [self.view addSubview:self.call.view];
    
    [[SignalClient getInstance] createConnect: self.addr.text];
}

#pragma mark protocal EventNotify
- (void) leave {
//    [self.view removeFromSuperview];
    
}


@end
```



### 2、信令服务器相关代码

```objective-c
#import "SignalClient.h"
@import SocketIO;

@interface SignalClient()
{
    
    SocketManager* manager;
    SocketIOClient* socket;
    
    NSString* room;
}

@end

@implementation SignalClient

static SignalClient* m_instance = nil;

+ (SignalClient*) getInstance {
    
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        m_instance = [[self alloc]init];
    });
    
    return m_instance;
}

- (void) createConnect: (NSString*) addr {
    NSLog(@"the server addr is %@", addr);
    /*
     log 是否打印日志
     forcePolling  是否强制使用轮询
     reconnectAttempts 重连次数，-1表示一直重连
     reconnectWait 重连间隔时间
     forceWebsockets 是否强制使用websocket
     */
    NSURL* url = [[NSURL alloc] initWithString:addr];
    manager = [[SocketManager alloc] initWithSocketURL:url
                                                config:@{@"log": @YES,
                                                         @"forcePolling":@YES,
                                                         @"forceWebsockets":@NO,
                                                         @"reconnectAttempts":@(5),
                                                         @"reconnectWait":@(1)
                                                         }];
    socket = manager.defaultSocket;
//    socket = [manager socketForNamespace:@"/socket.io"];
    
//    socket = [manager socketForNamespace:@""];
    
    [socket on:@"connect" callback:^(NSArray* data, SocketAckEmitter* ack) {
        NSLog(@"socket connected");
        [self.delegate connected];
    }];
    
    [socket on:@"error" callback:^(NSArray* data, SocketAckEmitter* ack) {
        NSLog(@"socket connect_error");
        [self.delegate connect_error];
    }];
    
    [socket on:@"reconnectAttempt" callback:^(NSArray* data, SocketAckEmitter* ack) {
        NSLog(@"socket reconnectAttempt");
        [self.delegate reconnectAttempt];
    }];
    //    [socket on:@"currentAmount" callback:^(NSArray* data, SocketAckEmitter* ack) {
    //        double cur = [[data objectAtIndex:0] floatValue];
    //
    //        [[socket emitWithAck:@"canUpdate" with:@[@(cur)]] timingOutAfter:0 callback:^(NSArray* data) {
    //            [socket emit:@"update" with:@[@{@"amount": @(cur + 2.50)}]];
    //        }];
    //
    //        [ack with:@[@"Got your currentAmount, ", @"dude"]];
    //    }];
    
    [socket on:@"joined" callback:^(NSArray * data, SocketAckEmitter * ack) {
        NSString* room = [data objectAtIndex:0];
        
        NSLog(@"joined room(%@)", room);
        
        [self.delegate joined:room];
        
    }];
    
    [socket on:@"leaved" callback:^(NSArray * data, SocketAckEmitter * ack) {
        NSString* room = [data objectAtIndex:0];
        
        NSLog(@"leaved room(%@)", room);
        
        [self.delegate leaved:room];
    }];
    
    [socket on:@"full" callback:^(NSArray * data, SocketAckEmitter * ack) {
        NSString* room = [data objectAtIndex:0];
        
        NSLog(@"room(%@) is full", room);
        [self.delegate full:room];
    }];
    
    [socket on:@"otherjoin" callback:^(NSArray * data, SocketAckEmitter * ack) {
        NSString* room = [data objectAtIndex:0];
        NSString* uid = [data objectAtIndex:1];
        
        NSLog(@"other user(%@) has been joined into room(%@)", room, uid);
        [self.delegate otherjoin:room User:uid];
    }];
    
    [socket on:@"bye" callback:^(NSArray * data, SocketAckEmitter * ack) {
        NSString* room = [data objectAtIndex:0];
        NSString* uid = [data objectAtIndex:1];
        
        NSLog(@"user(%@) has leaved from room(%@)", room, uid);
        [self.delegate byeFrom:room User:uid];
    }];
    
    [socket on:@"message" callback:^(NSArray * data, SocketAckEmitter * ack){
        NSString* room = [data objectAtIndex:0];
        NSDictionary* msg = [data objectAtIndex:1];
        
        NSLog(@"onMessage, room(%@), data(%@)", room, msg);
        
        NSString* type = msg[@"type"];
        if( [type isEqualToString:@"offer"]){
            [self.delegate offer: room message: msg];
        }else if( [type isEqualToString:@"answer"]){
            [self.delegate answer:room message: msg];
        }else if( [type isEqualToString:@"candidate"]){
            [self.delegate candidate:room message: msg];
        }else {
            NSLog(@"the msg is invalid!");
        }
    }];
    
    // 连接超时时间设置为3秒
    [socket connectWithTimeoutAfter: 3.0 withHandler:^(void){
        NSLog(@"socket connect_timeout 3.0s");
        [self.delegate connect_timeout];
    }];
}

- (void) joinRoom:(NSString*) room {
    NSLog(@"join room(%@)", room);
    
    if(socket.status == SocketIOStatusConnected){
        [socket emit:@"join" with:@[room]];
    }
}

- (void) leaveRoom:(NSString*) room {
    NSLog(@"leave room(%@)", room);
    
    if(socket.status == SocketIOStatusConnected){
        [socket emit:@"leave" with:@[room]];
    }
}

- (void) sendMessage: (NSString*) room withMsg:(NSDictionary*) msg {
    if(socket.status == SocketIOStatusConnected) {
        if(msg){
            NSLog(@"json:%@", msg);
            [socket emit:@"message" with:@[room, msg]];
        }else{
            NSLog(@"error: msg is nil!");
        }
        
    } else {
        NSLog(@"the socket has been disconnect!");
    }
}

@end

```



### 3、WebRTC相关代码



```objective-c
#import "CallViewController.h"
#import "SignalClient.h"

#import <WebRTC/WebRTC.h>
#import <MBProgressHUD/MBProgressHUD.h>

@interface CallViewController() <SignalEventNotify, RTCPeerConnectionDelegate, RTCVideoViewDelegate>
{
    
    NSString* myAddr;
    NSString* myRoom;
    
    NSString* myState;
    
    SignalClient* sigclient;
    
    RTCPeerConnectionFactory* factory;
    RTCCameraVideoCapturer* capture;
    //RTCMediaStream* localStream; //???
    RTCPeerConnection* peerConnection;
    
    RTCVideoTrack* videoTrack;
    RTCAudioTrack* audioTrack;
    
    RTCVideoTrack* remoteVideoTrack;
    CGSize remoteVideoSize;
    
    NSMutableArray* ICEServers;

}

@property (strong, nonatomic) RTCEAGLVideoView *remoteVideoView;
@property (strong, nonatomic) RTCCameraPreviewView *localVideoView;

@property (strong, nonatomic) UIButton* leaveBtn;


@property (strong, nonatomic) dispatch_source_t timer;

@end

@implementation CallViewController

static CGFloat const kLocalVideoViewSize = 120;
static CGFloat const kLocalVideoViewPadding = 8;

//static NSString *const RTCSTUNServerURL = @"stun:stun.l.google.com:19302";
//static NSString *const RTCSTUNServerURL = @"turn:xxx.xxx.xxx:3478";
static NSString *const RTCSTUNServerURL = @"turn:47.95.15.179:3478";
static int logY = 0;



- (instancetype) initAddr:(NSString*) addr withRoom:(NSString*) room {
    self = [super init];
    if (self) {
        myAddr = addr;
        myRoom = room;
    }
    return self;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    logY = 0;
    CGRect bounds = self.view.bounds;
    
    self.remoteVideoView = [[RTCEAGLVideoView alloc] initWithFrame:self.view.bounds];
    self.remoteVideoView.delegate = self;
    //[self.remoteVideoView set]
    //[self.remoteVideoView setBackgroundColor:[UIColor yellowColor]];
    [self.view addSubview:self.remoteVideoView];
    
    self.localVideoView = [[RTCCameraPreviewView alloc] initWithFrame:CGRectZero];
    [self.view addSubview:self.localVideoView];
    
    // Aspect fit local video view into a square box.
    CGRect localVideoFrame =
    CGRectMake(0, 0, kLocalVideoViewSize, kLocalVideoViewSize);
    // Place the view in the bottom right.
    localVideoFrame.origin.x = CGRectGetMaxX(bounds)
    - localVideoFrame.size.width - kLocalVideoViewPadding;
    localVideoFrame.origin.y = CGRectGetMaxY(bounds)
    - localVideoFrame.size.height - kLocalVideoViewPadding;
    [self.localVideoView setFrame: localVideoFrame];

    self.leaveBtn = [[UIButton alloc] init];
    [self.leaveBtn setTitleColor:[UIColor whiteColor] forState:UIControlStateNormal];
    [self.leaveBtn setTintColor:[UIColor whiteColor]];
    [self.leaveBtn setTitle:@"leave" forState:UIControlStateNormal];
    [self.leaveBtn setBackgroundColor:[UIColor greenColor]];
    [self.leaveBtn setShowsTouchWhenHighlighted:YES];
    [self.leaveBtn.layer setCornerRadius:40];
    [self.leaveBtn.layer setBorderWidth:1];
    [self.leaveBtn setClipsToBounds:FALSE];
    [self.leaveBtn setFrame:CGRectMake(self.view.bounds.size.width/2-40,
                                       self.view.bounds.size.height-140,
                                       80,
                                       80)];
    
    [self.leaveBtn addTarget:self
                      action:@selector(leaveRoom:)
            forControlEvents:UIControlEventTouchUpInside];
    
    [self.view addSubview:self.leaveBtn];
    
    [self createPeerConnectionFactory];
    
    //[self startTimer];
    
    //创建本地流
    [self captureLocalMedia];

    sigclient = [SignalClient getInstance];
    sigclient.delegate = self;
    ////[sigclient createConnect:myAddr];
    
    myState = @"init";
    
}

-(void)addLogToScreen:(NSString *)format, ...{
    
    va_list paramList;
    va_start(paramList,format);
    NSString* log = [[NSString alloc]initWithFormat:format arguments:paramList];
    va_end(paramList);
    
    CGRect labelRect = CGRectMake(0, logY++ * 20, 500, 200);
    UILabel *label = [[UILabel alloc] initWithFrame:labelRect];
    label.text = log;
    label.textColor = [UIColor redColor];
    [self.view addSubview:label];
}

//- (void) startTimer {
//    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
//    dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
//    self.timer = timer;
//    dispatch_source_set_timer(timer, DISPATCH_TIME_NOW, 200 * NSEC_PER_SEC, 0 * NSEC_PER_SEC);
//
//    __weak NSString* weakMyState = myState;
//    __weak NSString* weakMyRoom = myRoom;
//    dispatch_source_set_event_handler(timer, ^{
//
//        dispatch_
//
//        if([weakMyState isEqualToString:@"init"]) {
//            NSLog(@"%@",[NSThread currentThread]);
//            [[SignalClient getInstance] joinRoom:weakMyRoom];
//        }else {
//            [timer ]
//        }
//    });
//    dispatch_resume(timer);
//}

- (void)layoutSubviews {
    CGRect bounds = self.view.bounds;
    if (remoteVideoSize.width > 0 && remoteVideoSize.height > 0) {
        // Aspect fill remote video into bounds.
        CGRect remoteVideoFrame =
        AVMakeRectWithAspectRatioInsideRect(remoteVideoSize, bounds);
        CGFloat scale = 1;
        if (remoteVideoFrame.size.width > remoteVideoFrame.size.height) {
            // Scale by height.
            scale = bounds.size.height / remoteVideoFrame.size.height;
        } else {
            // Scale by width.
            scale = bounds.size.width / remoteVideoFrame.size.width;
        }
        remoteVideoFrame.size.height *= scale;
        remoteVideoFrame.size.width *= scale;
        self.remoteVideoView.frame = remoteVideoFrame;
        self.remoteVideoView.center =
        CGPointMake(CGRectGetMidX(bounds), CGRectGetMidY(bounds));
    } else {
        self.remoteVideoView.frame = bounds;
    }

}

- (void) leaveRoom:(UIButton*) sender {
   
    [self willMoveToParentViewController:nil];
    [self.view removeFromSuperview];
    [self removeFromParentViewController];
    
    if (!sigclient) {
        sigclient = [SignalClient getInstance];
    }
    
    if(![myState isEqualToString:@"leaved"]){
        [sigclient leaveRoom: myRoom];
    }
    
    if(peerConnection){
        [peerConnection close];
        peerConnection = nil;
    }
    
    NSLog(@"leave room(%@)", myRoom);
    [self addLogToScreen: @"leave room(%@)", myRoom];
}

#pragma mark - SignalEventNotify

- (void) leaved:(NSString *)room {
    NSLog(@"leaved room(%@) notify!", room);
    [self addLogToScreen: @"leaved room(%@) notify!", room];
}

- (void) joined:(NSString *)room {
    NSLog(@"joined room(%@) notify!", room);
    [self addLogToScreen: @"joined room(%@) notify!", room];
    
    myState = @"joined";
    
    //这里应该创建PeerConnection
    if (!peerConnection) {
        peerConnection = [self createPeerConnection];
    }
}

- (void) otherjoin:(NSString *)room User:(NSString *)uid {
    NSLog(@"other user(%@) has been joined into room(%@) notify!", uid, room);
    [self addLogToScreen: @"other user(%@) has been joined into room(%@) notify!", uid, room];
    if([myState isEqualToString:@"joined_unbind"]){
        if (!peerConnection) {
            peerConnection = [self createPeerConnection];
        }
    }
    
    myState =@"joined_conn";
    //调用call， 进行媒体协商
    [self doStartCall];
    
}

- (void) full:(NSString *)room {
    NSLog(@"the room(%@) is full notify!", room);
    [self addLogToScreen: @"the room(%@) is full notify!", room];
    myState = @"leaved";
    
    if(peerConnection) {
        [peerConnection close];
        peerConnection = nil;
    }
    
    //弹出提醒添加成功
    MBProgressHUD *hud= [[MBProgressHUD alloc] initWithView:self.view];
    [hud setRemoveFromSuperViewOnHide:YES];
    hud.label.text = @"房间已满";
    UIView* view = [[UIView alloc] initWithFrame:CGRectMake(0,0, 50, 50)];
    [hud setCustomView:view];
    [hud setMode:MBProgressHUDModeCustomView];
    [self.view addSubview:hud];
    [hud showAnimated:YES];
    [hud hideAnimated:YES afterDelay:1.0]; //设置1秒钟后自动消失
    
    if(self.localVideoView) {
        //[self.localVideoView removeFromSuperview];
        //self.localVideoView = nil;
    }
    
    if(self.remoteVideoView) {
        //[self.localVideoView removeFromSuperview];
        //self.remoteVideoView = nil;
    }
    
    if(capture) {
        [capture stopCapture];
        capture = nil;
    }
    
    if(factory) {
        factory = nil;
    }
}

- (void) byeFrom:(NSString *)room User:(NSString *)uid {
    NSLog(@"the user(%@) has leaved from room(%@) notify!", uid, room);
    [self addLogToScreen: @"the user(%@) has leaved from room(%@) notify!", uid, room];
    myState = @"joined_unbind";
    
    [peerConnection close];
    peerConnection = nil;
    
}

- (void)answer:(NSString *)room message:(NSDictionary *)dict {
    NSLog(@"have received a answer message %@", dict);
    
    NSString *remoteAnswerSdp = dict[@"sdp"];
    RTCSessionDescription *remoteSdp = [[RTCSessionDescription alloc]
                                           initWithType:RTCSdpTypeAnswer
                                                    sdp: remoteAnswerSdp];
    [peerConnection setRemoteDescription:remoteSdp
                       completionHandler:^(NSError * _Nullable error) {
        if(!error){
            NSLog(@"Success to set remote Answer SDP");
        }else{
            NSLog(@"Failure to set remote Answer SDP, err=%@", error);
        }
    }];
}

- (void) setLocalAnswer: (RTCPeerConnection*)pc withSdp: (RTCSessionDescription*)sdp {
    
    [pc setLocalDescription:sdp completionHandler:^(NSError * _Nullable error) {
        if(!error){
            NSLog(@"Successed to set local answer!");
        }else {
            NSLog(@"Failed to set local answer, err=%@", error);
        }
    }];
    
    __weak NSString* weakMyRoom = myRoom;
    dispatch_async(dispatch_get_main_queue(), ^{
        
        //send answer sdp
        NSDictionary* dict = [[NSDictionary alloc] initWithObjects:@[@"answer", sdp.sdp]
                                                           forKeys: @[@"type", @"sdp"]];
        
        [[SignalClient getInstance] sendMessage: weakMyRoom withMsg:dict];
    });
}

- (void) getAnswer:(RTCPeerConnection*) pc {
    
    NSLog(@"Success to set remote offer SDP");
    
    [pc answerForConstraints:[self defaultPeerConnContraints]
                           completionHandler:^(RTCSessionDescription * _Nullable sdp, NSError * _Nullable error) {
                               if(!error){
                                   NSLog(@"Success to create local answer sdp!");
                                   __weak RTCPeerConnection* weakPeerConn = pc;
                                   [self setLocalAnswer:weakPeerConn withSdp:sdp];
                                   
                               }else{
                                   NSLog(@"Failure to create local answer sdp!");
                               }
                           }];
}

- (void) offer:(NSString *)room message:(NSDictionary *)dict {
    NSLog(@"have received a offer message %@", dict);
    
    NSString* remoteOfferSdp = dict[@"sdp"];
    RTCSessionDescription* remoteSdp = [[RTCSessionDescription alloc]
                                        initWithType:RTCSdpTypeOffer
                                        sdp: remoteOfferSdp];
    if(!peerConnection){
        peerConnection = [self createPeerConnection];
    }
    
     __weak RTCPeerConnection* weakPeerConnection = peerConnection;
    [weakPeerConnection setRemoteDescription:remoteSdp completionHandler:^(NSError * _Nullable error) {
        if(!error){
            [self getAnswer: weakPeerConnection];
        }else{
            NSLog(@"Failure to set remote offer SDP, err=%@", error);
        }
    }];
}

- (void)candidate:(NSString *)room message:(NSDictionary *)dict {
    NSLog(@"have received a message %@", dict);
    
    NSString* desc = dict[@"sdp"];
    NSString* sdpMLineIndex = dict[@"label"];
    int index = [sdpMLineIndex intValue];
    NSString* sdpMid = dict[@"id"];
    
    
    RTCIceCandidate *candidate = [[RTCIceCandidate alloc] initWithSdp:desc
                                                        sdpMLineIndex:index
                                                               sdpMid:sdpMid];;
    [peerConnection addIceCandidate:candidate];
}

- (void)connected {
    [[SignalClient getInstance]  joinRoom: myRoom];
    [self addLogToScreen: @"socket connect success!"];
    [self addLogToScreen: @"joinRoom: %@", myRoom];
    

}

- (void)connect_error {
    //todo: notfiy UI
    [self addLogToScreen: @"socket connect_error!"];
}

- (void)connect_timeout {
    //todo: notfiy UI
    [self addLogToScreen: @"socket connect_timeout!"];
}

- (void) reconnectAttempt{
    [self addLogToScreen: @"socket reconnectAttempt!"];
}
#pragma mark RTCPeerConnectionDelegate

/** Called when the SignalingState changed. */
- (void)peerConnection:(RTCPeerConnection *)peerConnection
didChangeSignalingState:(RTCSignalingState)stateChanged{
    NSLog(@"%s",__func__);
}

/** Called when media is received on a new stream from remote peer. */
- (void)peerConnection:(RTCPeerConnection *)peerConnection didAddStream:(RTCMediaStream *)stream{
    NSLog(@"%s",__func__);
}

/** Called when a remote peer closes a stream.
 *  This is not called when RTCSdpSemanticsUnifiedPlan is specified.
 */
- (void)peerConnection:(RTCPeerConnection *)peerConnection didRemoveStream:(RTCMediaStream *)stream{
     NSLog(@"%s",__func__);
}

/** Called when negotiation is needed, for example ICE has restarted. */
- (void)peerConnectionShouldNegotiate:(RTCPeerConnection *)peerConnection {
    NSLog(@"%s",__func__);
}

/** Called any time the IceConnectionState changes. */
- (void)peerConnection:(RTCPeerConnection *)peerConnection
didChangeIceConnectionState:(RTCIceConnectionState)newState{
    NSLog(@"%s",__func__);
}

/** Called any time the IceGatheringState changes. */
- (void)peerConnection:(RTCPeerConnection *)peerConnection
didChangeIceGatheringState:(RTCIceGatheringState)newState{
    NSLog(@"%s",__func__);
}

/** New ice candidate has been found. */
- (void)peerConnection:(RTCPeerConnection *)peerConnection
didGenerateIceCandidate:(RTCIceCandidate *)candidate{
    NSLog(@"%s",__func__);
    
    NSString* weakMyRoom = myRoom;
    dispatch_async(dispatch_get_main_queue(), ^{
    
        NSDictionary* dict = [[NSDictionary alloc] initWithObjects:@[@"candidate",
                                                                [NSString stringWithFormat:@"%d", candidate.sdpMLineIndex],
                                                                candidate.sdpMid,
                                                                candidate.sdp]
                                                           forKeys:@[@"type", @"label", @"id", @"candidate"]];
    
        [[SignalClient getInstance] sendMessage: weakMyRoom
                                    withMsg:dict];
    });
}

/** Called when a group of local Ice candidates have been removed. */
- (void)peerConnection:(RTCPeerConnection *)peerConnection
didRemoveIceCandidates:(NSArray<RTCIceCandidate *> *)candidates {
    NSLog(@"%s",__func__);
}

/** New data channel has been opened. */
- (void)peerConnection:(RTCPeerConnection *)peerConnection
    didOpenDataChannel:(RTCDataChannel *)dataChannel {
    NSLog(@"%s",__func__);
}

/** Called when a receiver and its track are created. */
- (void)peerConnection:(RTCPeerConnection *)peerConnection
        didAddReceiver:(RTCRtpReceiver *)rtpReceiver
               streams:(NSArray<RTCMediaStream *> *)mediaStreams {
    NSLog(@"%s",__func__);
    
    RTCMediaStreamTrack* track = rtpReceiver.track;
    if([track.kind isEqualToString:kRTCMediaStreamTrackKindVideo]){
        if(!self.remoteVideoView){
            NSLog(@"error:remoteVideoView have not been created!");
            return;
        }
        
        remoteVideoTrack = (RTCVideoTrack*)track;
        
        //dispatch_async(dispatch_get_main_queue(), ^{

            [remoteVideoTrack addRenderer: self.remoteVideoView];
        //});
        //[remoteVideoTrack setIsEnabled:true];
        
        //[self.view addSubview:self.remoteVideoView];
    }
    
}

#pragma mark webrtc


- (RTCMediaConstraints*) defaultPeerConnContraints {
    RTCMediaConstraints* mediaConstraints =
    [[RTCMediaConstraints alloc] initWithMandatoryConstraints:@{
                                                                kRTCMediaConstraintsOfferToReceiveAudio:kRTCMediaConstraintsValueTrue,
                                                                kRTCMediaConstraintsOfferToReceiveVideo:kRTCMediaConstraintsValueTrue
                                                                }
                                          optionalConstraints:@{ @"DtlsSrtpKeyAgreement" : @"true" }];
    return mediaConstraints;
}


- (void)captureLocalMedia {
    NSDictionary* mandatoryConstraints = @{};
    RTCMediaConstraints* constraints =
    [[RTCMediaConstraints alloc] initWithMandatoryConstraints:mandatoryConstraints
                                          optionalConstraints:nil];
    
    RTCAudioSource* audioSource = [factory audioSourceWithConstraints: constraints];
    //self.audioTrack = [factory audioTrackWithTrackId:@"ARDAMSa0"];
    audioTrack = [factory audioTrackWithSource:audioSource trackId:@"ADRAMSa0"];
    
    NSArray<AVCaptureDevice* >* captureDevices = [RTCCameraVideoCapturer captureDevices];
    AVCaptureDevicePosition position = AVCaptureDevicePositionFront;
    AVCaptureDevice* device = captureDevices[0];
    for (AVCaptureDevice* obj in captureDevices) {
        if (obj.position == position) {
            device = obj;
            break;
        }
    }
    
    //检测摄像头权限
    AVAuthorizationStatus authStatus = [AVCaptureDevice authorizationStatusForMediaType:AVMediaTypeVideo];
    if(authStatus == AVAuthorizationStatusRestricted || authStatus == AVAuthorizationStatusDenied) {
        NSLog(@"相机访问受限");
        return;
    }
    
    if (device) {
        RTCVideoSource* videoSource = [factory videoSource];
        capture = [[RTCCameraVideoCapturer alloc] initWithDelegate:videoSource];
        AVCaptureDeviceFormat* format = [[RTCCameraVideoCapturer supportedFormatsForDevice:device] lastObject];
        CGFloat fps = [[format videoSupportedFrameRateRanges] firstObject].maxFrameRate;
        videoTrack = [factory videoTrackWithSource:videoSource trackId:@"ARDAMSv0"];
        self.localVideoView.captureSession = capture.captureSession;
        [capture startCaptureWithDevice:device
                                     format:format
                                        fps:fps];
        
    }
}

//初始化STUN Server （ICE Server）
- (RTCIceServer *)defaultSTUNServer {
    return [[RTCIceServer alloc] initWithURLStrings:@[RTCSTUNServerURL]
                                           username:@"carrot"
                                         credential:@"123456"];
}

- (void)createPeerConnectionFactory {
    //设置SSL传输
    [RTCPeerConnectionFactory initialize];
    
    //如果点对点工厂为空
    if (!factory)
    {
        RTCDefaultVideoDecoderFactory* decoderFactory = [[RTCDefaultVideoDecoderFactory alloc] init];
        RTCDefaultVideoEncoderFactory* encoderFactory = [[RTCDefaultVideoEncoderFactory alloc] init];
        NSArray* codecs = [encoderFactory supportedCodecs];
        [encoderFactory setPreferredCodec:codecs[2]];
        
        factory = [[RTCPeerConnectionFactory alloc] initWithEncoderFactory: encoderFactory
                                                            decoderFactory: decoderFactory];
    }
}

- (RTCPeerConnection *)createPeerConnection {
    //得到ICEServer
    if (!ICEServers) {
        ICEServers = [NSMutableArray array];
        [ICEServers addObject:[self defaultSTUNServer]];
    }

    //用工厂来创建连接
    RTCConfiguration* configuration = [[RTCConfiguration alloc] init];
    [configuration setIceServers:ICEServers];
    RTCPeerConnection* conn = [factory
                                     peerConnectionWithConfiguration:configuration
                                                         constraints:[self defaultPeerConnContraints]
                                                            delegate:self];
    
    NSArray<NSString*>* mediaStreamLabels = @[@"ARDAMS"];
    [conn addTrack:videoTrack streamIds:mediaStreamLabels];
    [conn addTrack:audioTrack streamIds:mediaStreamLabels];

    return conn;
}

- (void)setLocalOffer:(RTCPeerConnection*)pc withSdp:(RTCSessionDescription*) sdp{
    
    [pc setLocalDescription:sdp completionHandler:^(NSError * _Nullable error) {
        if (!error) {
            NSLog(@"Successed to set local offer sdp!");
        }else{
            NSLog(@"Failed to set local offer sdp, err=%@", error);
        }
    }];
    
    __weak NSString* weakMyRoom = myRoom;
    dispatch_async(dispatch_get_main_queue(), ^{
        
        NSDictionary* dict = [[NSDictionary alloc] initWithObjects:@[@"offer", sdp.sdp]
                                                           forKeys: @[@"type", @"sdp"]];
        
        [[SignalClient getInstance] sendMessage: weakMyRoom
                                        withMsg: dict];
    });
}

- (void)doStartCall {
    NSLog(@"Start Call, Wait ...");
    [self addLogToScreen: @"Start Call, Wait ..."];
    if (!peerConnection) {
        peerConnection = [self createPeerConnection];
    }
    
    [peerConnection offerForConstraints:[self defaultPeerConnContraints]
                      completionHandler:^(RTCSessionDescription * _Nullable sdp, NSError * _Nullable error) {
                          if(error){
                              NSLog(@"Failed to create offer SDP, err=%@", error);
                          } else {
                              __weak RTCPeerConnection* weakPeerConnction = self->peerConnection;
                              [self setLocalOffer: weakPeerConnction withSdp: sdp];
                          }
                      }];
}

#pragma mark - RTCEAGLVideoViewDelegate
- (void)videoView:(RTCEAGLVideoView*)videoView didChangeVideoSize:(CGSize)size {
    if (videoView == self.remoteVideoView) {
        remoteVideoSize = size;
    }
    [self layoutSubviews];
}

@end

```

