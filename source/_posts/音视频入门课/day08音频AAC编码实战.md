---
title: day08-音频AAC编码实战(三)
categories:
  - 音视频入门课
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



## 二、录音、重采样、aac编码代码抽取优化、完整代码



```C
#include "test.h"
#include <unistd.h>
#include "include/libavutil/avutil.h"
#include "include/libavdevice/avdevice.h"
#include "include/libavcodec/avcodec.h"
#include "include/libswresample/swresample.h"

#include <string.h>

static int rec_status = 0;

void set_status(int status){
    rec_status = status;
}

//[in]
//[out]
//ret
//@brief encode audio data
static void encode(AVCodecContext *ctx,
            AVFrame *frame,
            AVPacket *pkt,
            FILE *output){
    
    int ret = 0;
    
    //将数据送编码器
    ret = avcodec_send_frame(ctx, frame);
    
    //如果ret>=0说明数据设置成功
    while(ret >= 0){
        //获取编码后的音频数据,如果成功，需要重复获取，直到失败为止
        ret = avcodec_receive_packet(ctx, pkt);
        
        if(ret == AVERROR(EAGAIN) || ret == AVERROR_EOF){
            return;
        }else if( ret < 0){
            printf("Error, encoding audio frame\n");
            exit(-1);
        }
        
        //write file
        fwrite(pkt->data, 1, pkt->size, output);
        fflush(output);
    }
    
    return;
}

//[in]
//[out]
//
static AVCodecContext* open_coder(){
    
    //打开编码器
    //avcodec_find_encoder(AV_CODEC_ID_AAC);
    AVCodec *codec = avcodec_find_encoder_by_name("libfdk_aac");
    
    //创建 codec 上下文
    AVCodecContext *codec_ctx = avcodec_alloc_context3(codec);
    
    codec_ctx->sample_fmt = AV_SAMPLE_FMT_S16;          //输入音频的采样大小
    codec_ctx->channel_layout = AV_CH_LAYOUT_STEREO;    //输入音频的channel layout
    codec_ctx->channels = 2;                            //输入音频 channel 个数
    codec_ctx->sample_rate = 44100;                     //输入音频的采样率
    codec_ctx->bit_rate = 0; //AAC_LC: 128K, AAC HE: 64K, AAC HE V2: 32K
    codec_ctx->profile = FF_PROFILE_AAC_HE; //阅读 ffmpeg 代码
    
    //打开编码器
    if(avcodec_open2(codec_ctx, codec, NULL)<0){
        //
        
        return NULL;
    }
    
    return codec_ctx;
}

static
SwrContext* init_swr(){
    
    SwrContext *swr_ctx = NULL;
    
    //channel, number/
    swr_ctx = swr_alloc_set_opts(NULL,                //ctx
                                 AV_CH_LAYOUT_STEREO, //输出channel布局
                                 AV_SAMPLE_FMT_S16,   //输出的采样格式
                                 44100,               //采样率
                                 AV_CH_LAYOUT_STEREO, //输入channel布局
                                 AV_SAMPLE_FMT_FLT,   //输入的采样格式
                                 44100,               //输入的采样率
                                 0, NULL);
    
    if(!swr_ctx){
        
    }
    
    if(swr_init(swr_ctx) < 0){
        
    }
    
    return swr_ctx;
}

/**
  * @brief open audio device
  * @return succ: AVFormatContext*, fail: NULL
  */
static
AVFormatContext* open_dev(){
    
    int ret = 0;
    char errors[1024] = {0, };
    
    //ctx
    AVFormatContext *fmt_ctx = NULL;
    AVDictionary *options = NULL;
    
    //[[video device]:[audio device]]
    char *devicename = ":0";
    
    //get format
    AVInputFormat *iformat = av_find_input_format("avfoundation");
    
    //open device
    if((ret = avformat_open_input(&fmt_ctx, devicename, iformat, &options)) < 0 ){
        av_strerror(ret, errors, 1024);
        fprintf(stderr, "Failed to open audio device, [%d]%s\n", ret, errors);
        return NULL;
    }
    
    return fmt_ctx;
}

/**
 * @brief xxxx
 * @return xxx
 */
static
AVFrame* create_frame(){
    
    AVFrame *frame = NULL;
    
    //音频输入数据
    frame = av_frame_alloc();
    if(!frame){
        printf("Error, No Memory!\n");
        goto __ERROR;
    }
    
    //set parameters
    frame->nb_samples     = 512;                //单通道一个音频帧的采样数
    frame->format         = AV_SAMPLE_FMT_S16;  //每个采样的大小
    frame->channel_layout = AV_CH_LAYOUT_STEREO; //channel layout
    
    //alloc inner memory
    av_frame_get_buffer(frame, 0); // 512 * 2 * = 2048
    if(!frame->data[0]){
        printf("Error, Failed to alloc buf in frame!\n");
        //内存泄漏
        goto __ERROR;
    }
    
    return frame;
    
__ERROR:
    if(frame){
        av_frame_free(&frame);
    }
    
    return NULL;
}

static
void alloc_data_4_resample(uint8_t ***src_data,
                           int *src_linesize,
                           uint8_t *** dst_data,
                           int *dst_linesize){
    //4096/4=1024/2=512
    //创建输入缓冲区
    av_samples_alloc_array_and_samples(src_data,         //输出缓冲区地址
                                       src_linesize,     //缓冲区的大小
                                       2,                 //通道个数
                                       512,               //单通道采样个数
                                       AV_SAMPLE_FMT_FLT, //采样格式
                                       0);
    
    //创建输出缓冲区
    av_samples_alloc_array_and_samples(dst_data,         //输出缓冲区地址
                                       dst_linesize,     //缓冲区的大小
                                       2,                 //通道个数
                                       512,               //单通道采样个数
                                       AV_SAMPLE_FMT_S16, //采样格式
                                       0);
}

/**
 */
static
void free_data_4_resample(uint8_t **src_data, uint8_t **dst_data){
    //释放输入输出缓冲区
    if(src_data){
        av_freep(&src_data[0]);
    }
    av_freep(&src_data);
    
    if(dst_data){
        av_freep(&dst_data[0]);
    }
    av_freep(&dst_data);
}

/**
 */
static
void read_data_and_encode(AVFormatContext *fmt_ctx, //
                          AVCodecContext *c_ctx,
                          SwrContext* swr_ctx,
                          FILE *outfile){
    
    int ret = 0;
    
    //pakcet
    AVPacket pkt;
    AVFrame *frame = NULL;
    AVPacket *newpkt = NULL;
    
    //重采样缓冲区
    uint8_t **src_data = NULL;
    int src_linesize = 0;
    
    uint8_t **dst_data = NULL;
    int dst_linesize = 0;

    frame = create_frame();
    if(!frame){
        //printf(...)
        goto __ERROR;
    }
    
    newpkt = av_packet_alloc(); //分配编码后的数据空间
    if(!newpkt){
        printf("Error, Failed to alloc buf in frame!\n");
        goto __ERROR;
    }
    
    //分配重采样输入/输出缓冲区
    alloc_data_4_resample(&src_data, &src_linesize, &dst_data, &dst_linesize);
    
    //read data from device
    while(rec_status) {
        ret = av_read_frame(fmt_ctx, &pkt);
        
        if (pkt.size <= 0) {
            usleep(300);
            continue;
        }
        //进行内存拷贝，按字节拷贝的
        memcpy((void*)src_data[0], (void*)pkt.data, pkt.size);
        
        //重采样
        swr_convert(swr_ctx,                    //重采样的上下文
                    dst_data,                   //输出结果缓冲区
                    512,                        //每个通道的采样数
                    (const uint8_t **)src_data, //输入缓冲区
                    512);                       //输入单个通道的采样数
        
        //将重采样的数据拷贝到 frame 中
        memcpy((void *)frame->data[0], dst_data[0], dst_linesize);
        
        //encode
        encode(c_ctx, frame, newpkt, outfile);
        
        //
        av_packet_unref(&pkt); //release pkt
    }
    
    //强制将编码器缓冲区中的音频进行编码输出
    encode(c_ctx, NULL, newpkt, outfile);

__ERROR:
    //释放 AVFrame 和 AVPacket
    if(frame){
        av_frame_free(&frame);
    }
    
    if(newpkt){
        av_packet_free(&newpkt);
    }
    
    //释放重采样缓冲区
    free_data_4_resample(src_data, dst_data);
}

void rec_audio() {
  
    //context
    AVFormatContext *fmt_ctx = NULL;
    AVCodecContext *c_ctx = NULL;
    SwrContext* swr_ctx = NULL;

    //set log level
    av_log_set_level(AV_LOG_DEBUG);
    
    //register audio device
    avdevice_register_all();
    
    //start record
    rec_status = 1;
    
    //create file
    //char *out = "/Users/lichao/Downloads/av_base/audio.pcm";
    char *out = "/Users/carrot/Desktop/MyCode/audio.aac";
    FILE *outfile = fopen(out, "wb+");
    if(!outfile){
        printf("Error, Failed to open file!\n");
        goto __ERROR;
    }
    
    //打开设备
    fmt_ctx = open_dev();
    if(!fmt_ctx){
        printf("Error, Failed to open device!\n");
        goto __ERROR;
    }

    //打开编码器上下文
    c_ctx = open_coder();
    if(!c_ctx){
        printf("...");
        goto __ERROR;
    }

    //初始化重采样上下文
    swr_ctx = init_swr();
    if(!swr_ctx){
        printf("Error, Failed to alloc buf in frame!\n");
        goto __ERROR;
    }
    
    //encode
    read_data_and_encode(fmt_ctx, c_ctx, swr_ctx, outfile);

__ERROR:
    //释放重采样的上下文
    if(swr_ctx){
        swr_free(&swr_ctx);
    }

    if(c_ctx){
        avcodec_free_context(&c_ctx);
    }
    
    //close device and release ctx
    if(fmt_ctx) {
        avformat_close_input(&fmt_ctx);
    }
    
    if(outfile){
        //close file
        fclose(outfile);
    }

    av_log(NULL, AV_LOG_DEBUG, "finish!\n");
    
    return;
}

#if 0
int main(int argc, char *argv[])
{
    rec_audio();
    return 0;
}
#endif
```



- 播放上述录音

```sh
ffplay audio.aac
```

