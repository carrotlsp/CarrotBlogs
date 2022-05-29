---
title: day08-音频AAC编码实战(三)
categories:
  - 音视频
tags:
  - 音视频
  - ffmpeg
date: 2022-05-29 13:30:38
---



## 一、AAC编码代码实战

### 1、AVPacket、AVFrame这两个重要结构体一般存放什么数据？

- <font color="blue">AVFrame</font>：一般存放编码前的数据，对于音频数据来说，大部分情况下是PCM数据。

- <font color="blue">AVPacket</font>：一般存放编码后的数据，对于音频数据来说，大部分情况是压缩数据。



### 2、那为什么从麦克风读取的PCM数据，却是放在AVPacket呢？

- 因为ffmpeg认为麦克风是个外部多媒体文件（类似mp4），ffmpeg统一认为是编码后的AVPacket数据。



### 3、AAC编码的关键步骤

- 找到编码器、创建上下文、打开编码器

```C
    //找到libfdk编码器
    const AVCodec *codec = avcodec_find_encoder_by_name("libfdk_aac");
    if (codec == NULL) {
        printf("avcodec_find_encoder_by_name error");
        return;
    }
    
    //创建编码上下文
    AVCodecContext *codec_ctx = avcodec_alloc_context3(codec);
    codec_ctx->sample_fmt = AV_SAMPLE_FMT_S16;          //输入音频的采样大小
    codec_ctx->channel_layout = AV_CH_LAYOUT_STEREO;    //输入音频的channel layout
    codec_ctx->channels = 2;                            //输入音频 channel 个数
    codec_ctx->sample_rate = 44100;                     //输入音频的采样率
//    codec_ctx->bit_rate = 0; //AAC_LC:128K, AAC HE:64K, AAC HE V2:32K
    codec_ctx->profile = FF_PROFILE_AAC_HE; //阅读 ffmpeg 代码，可知bit_rate 和 profile之间的设计关系
    
    //打开编码器
    if (avcodec_open2(codec_ctx, codec, NULL) < 0) {
        printf("avcodec_open2 error");
        return;
    }
```

- 初始化输入和数据格式

```C
    //初始化AAC编码前的数据载体
    AVFrame *aac_frame = av_frame_alloc();
    if (!aac_frame) {
        printf("av_frame_alloc 失败");
        return;
    }
    aac_frame->nb_samples           = 512;                  //单通道一个音频帧的采样数
    aac_frame->format               = AV_SAMPLE_FMT_S16;    //每个采样大小
    aac_frame->channel_layout       = AV_CH_LAYOUT_STEREO;  //channel layout
    av_frame_get_buffer(aac_frame, 0);                      //512 * 2 * 2 = 2048 AVFrame的大小
    if (!aac_frame->buf[0]) {
        printf("av_frame_get_buffer 失败");
        return;
    }
    
    //初始化AAC编码后的数据载体
    AVPacket *aac_packet = av_packet_alloc(); //分配编码后的数据空间
    if (!aac_packet) {
        printf("av_packet_alloc 失败");
        return;
    }
```

- 双重循环，从编码器中去除编码后的数据

```C
    //read data from device
    while ((ret = av_read_frame(fmt_ctx, &pkt)) == 0 || count++ < 50000) {
        usleep(100);
        printf("ret %d", ret);
        if (pkt.size > 0) {
            //进行内存拷贝
            memcpy(src_data[0], pkt.data, pkt.size);
            
            //重采样
            swr_convert(swr_ctx, dst_data, 512, (const uint8_t **)src_data, 512);
            
            //将重采样的数据拷贝到frame中去
            memcpy(aac_frame->data[0], dst_data[0], dst_linesize);
            
            //将数据送编码器
            ret = avcodec_send_frame(codec_ctx, aac_frame);
            
            //如果ret >= 0 说明数据设置成功
            while (ret >= 0) {
                //获取编码后的音频数据，如果成功，则需要重复获取，直到失败为止
                ret = avcodec_receive_packet(codec_ctx, aac_packet);
                if (ret < 0) {
                    if (ret == AVERROR(EAGAIN) || ret == AVERROR_EOF) {
                        break;
                    } else {
                        printf("avcodec_receive_packet error");
                        exit(-1);
                    }
                }
                
                fwrite(aac_packet->data, aac_packet->size, 1, outFile);
                fflush(outFile);
            }
            
            printf("packet size is %d(%p), count=%d \n", pkt.size, pkt.data, count);
            av_packet_unref(&pkt);
        }
    }
```



### 4、AAC编码的完整代码(还有bug)

```C
#include "test.h"
#include <unistd.h>
#include "include/libavutil/avutil.h"
#include "include/libavdevice/avdevice.h"
#include "include/libavcodec/avcodec.h"
#include "include/libswresample/swresample.h"

//录音
void record_audio(void) {
    int ret = 0;
    char errors[1024];
    
    //ctx
    AVFormatContext *fmt_ctx = NULL;
    AVDictionary *options = NULL;
    SwrContext *swr_ctx = NULL;
    
    swr_ctx = swr_alloc_set_opts(NULL,
                                 AV_CH_LAYOUT_STEREO, AV_SAMPLE_FMT_S16, 44100,  // 重采样输出音频三要素
                                 AV_CH_LAYOUT_STEREO, AV_SAMPLE_FMT_FLT, 44100,  // 重采样输入音频三要数
                                 0, NULL);
    if (swr_ctx == NULL) {
        printf("swr_alloc_set_opts error");
        return;
    }
    
    ret = swr_init(swr_ctx);
    if (ret != 0) {
        printf("swr_init error");
        return;
    }
    
    uint8_t **src_data = NULL;
    int src_linesize = 0;
    uint8_t **dst_data = NULL;
    int dst_linesize = 0;
    
    av_samples_alloc_array_and_samples(&src_data, &src_linesize, 2, 512, AV_SAMPLE_FMT_FLT, 0);
    av_samples_alloc_array_and_samples(&dst_data, &dst_linesize, 2, 512, AV_SAMPLE_FMT_S16, 0);
    
    //packet
    int count = 0;
    AVPacket pkt;
    
    // [video device]:[aduio device]
    char *devicename = ":0";
    
    //register audio device
    avdevice_register_all();
    
    //get format
    const AVInputFormat *iformat = av_find_input_format("avfoundation");
    
    //open device
    ret = avformat_open_input(&fmt_ctx, devicename, iformat, &options);
    if (ret < 0) {
        av_strerror(ret, errors, 1024);
        printf("avformat_open_input error");
        return;
    }
    
    //crate file
    char *outPath = "/Users/carrot/Desktop/MyCode/he_audio.aac";
    FILE *outFile = fopen(outPath, "wb+");
    if (outFile == NULL) {
        printf("outFile fopen failed");
        return;
    }
    
    //找到libfdk编码器
    const AVCodec *codec = avcodec_find_encoder_by_name("libfdk_aac");
    if (codec == NULL) {
        printf("avcodec_find_encoder_by_name error");
        return;
    }
    
    //创建编码上下文
    AVCodecContext *codec_ctx = avcodec_alloc_context3(codec);
    codec_ctx->sample_fmt = AV_SAMPLE_FMT_S16;          //输入音频的采样大小
    codec_ctx->channel_layout = AV_CH_LAYOUT_STEREO;    //输入音频的channel layout
    codec_ctx->channels = 2;                            //输入音频 channel 个数
    codec_ctx->sample_rate = 44100;                     //输入音频的采样率
//    codec_ctx->bit_rate = 0; //AAC_LC:128K, AAC HE:64K, AAC HE V2:32K
    codec_ctx->profile = FF_PROFILE_AAC_HE; //阅读 ffmpeg 代码，可知bit_rate 和 profile之间的设计关系
    
    //打开编码器
    if (avcodec_open2(codec_ctx, codec, NULL) < 0) {
        printf("avcodec_open2 error");
        return;
    }
    
    //初始化AAC编码前的数据载体
    AVFrame *aac_frame = av_frame_alloc();
    if (!aac_frame) {
        printf("av_frame_alloc 失败");
        return;
    }
    aac_frame->nb_samples           = 512;                  //单通道一个音频帧的采样数
    aac_frame->format               = AV_SAMPLE_FMT_S16;    //每个采样大小
    aac_frame->channel_layout       = AV_CH_LAYOUT_STEREO;  //channel layout
    av_frame_get_buffer(aac_frame, 0);                      //512 * 2 * 2 = 2048 AVFrame的大小
    if (!aac_frame->buf[0]) {
        printf("av_frame_get_buffer 失败");
        return;
    }
    
    //初始化AAC编码后的数据载体
    AVPacket *aac_packet = av_packet_alloc(); //分配编码后的数据空间
    if (!aac_packet) {
        printf("av_packet_alloc 失败");
        return;
    }
    
    
    //read data from device
    while ((ret = av_read_frame(fmt_ctx, &pkt)) == 0 || count++ < 50000) {
        usleep(100);
        printf("ret %d", ret);
        if (pkt.size > 0) {
            //进行内存拷贝
            memcpy(src_data[0], pkt.data, pkt.size);
            
            //重采样
            swr_convert(swr_ctx, dst_data, 512, (const uint8_t **)src_data, 512);
            
            //将重采样的数据拷贝到frame中去
            memcpy(aac_frame->data[0], dst_data[0], dst_linesize);
            
            //将数据送编码器
            ret = avcodec_send_frame(codec_ctx, aac_frame);
            
            //如果ret >= 0 说明数据设置成功
            while (ret >= 0) {
                //获取编码后的音频数据，如果成功，则需要重复获取，直到失败为止
                ret = avcodec_receive_packet(codec_ctx, aac_packet);
                if (ret < 0) {
                    if (ret == AVERROR(EAGAIN) || ret == AVERROR_EOF) {
                        break;
                    } else {
                        printf("avcodec_receive_packet error");
                        exit(-1);
                    }
                }
                
                fwrite(aac_packet->data, aac_packet->size, 1, outFile);
                fflush(outFile);
            }
            
            printf("packet size is %d(%p), count=%d \n", pkt.size, pkt.data, count);
            av_packet_unref(&pkt);
        }
    }
    
    //close device and release ctx
    avformat_close_input(&fmt_ctx);
    
    printf("运行结束\n");
}
```



### 5、借助ffplay播放编码后的aac文件

```sh
ffplay he_audio.aac	
```



## 二、录音、重采样、aac编码代码抽取优化

