---
layout:     post
title:      "ExoPlayer通过ffmpeg支持更多容器实现说明"
subtitle:   "FfmpegExtractor实现说明"
date:       2021-05-20 23:24:00
author:     "Jack"
header-img: "img/exoplayer.png"
tags:
    - ExoPlayer
    - ffmpeg
---
## 目的

ExoPlayer原生支持的容器有限，解复用和解码能力不足，需要通过ffmpeg来扩充解复用和解码能力，本文档描述用在ExoPlayer里用ffmpeg的libavformat进行解复用的实现方案，同时也会说明ExoPlayer的```Extractor```扩展方式。

## ExoPlayer版本

此文档基于当前最新的ExoPlayer 2.10.x版本，后续新版本可能会有较大变化。

## Extractor扩展说明

### Extractor的功能

### 1. 拆包

将```ExtractorInput```送过来的数据流解析后，拆包成未解码的音频和视频帧，并加上时间戳后，通过```TrackOuput.sampleData```输出。

### 2. 时间Seek

Seek到指定的时间，有两种seek方式：

1. 基于索引表，能通过文件头信息解析出时间与文件位置的关系，直接Seek到对应的文件位置，通过查找SeekMap实现，mp4文件采用此种方法，但很多文件并没有索引表，会采用方法2；
2. 二进制seek，先通过文件大小得出一个大致的文件位置，然后用二分查找法找到最接近目标时间的文件位置，通过```Extractor.read```来实现Seek的操作。

## 实现FfmpegExtractor要解决的问题

由于ExoPlayer对IO进行了封装，在Extractor层面只能通过```ExtractorInput```获取流数据，并没有文件常见的read和seek操作，而ffmpeg的IO已经通过```AVIOContext```封装，提供了```read_packet```和```seek```回调函数，由此可见，ExoPlayer和ffmpeg的IO是无法直接兼容的，需要解决一下两个问题：

### 1. 数据读取

由于```ExtractorInput```是动态的，当```ProgressiveMediaPeriod.ExtractingLoadable.load```函数的while循环退出时，就会从```dataSource```新创建```ExtractorInput```，所以要采用代理模式，通过```ExtractorInputWrapper```将动态的```ExtractorInput```进行封装，提供给```FfmpegParserJni```的read回调，给ffmpeg提供流数据。

### 2. 数据Seek

这里的数据seek是指IO Seek，而不是上面提到的时间Seek，是指ffmpeg在解析流信息和执行时间seek时，需要在流数据里前后随机访问指定位置的数据，而ExoPlayer提供给Extractor访问数据的方式，无法满足随机访问的方式，唯一的实现方法是在Extractor.read里返回RESULT_SEEK，并设置seekPosition.position，让上层的load循环重新创建ExtractorInput，但ffmpeg里的IO操作是不能被打断的，所以，必须要想办法解决这个问题。

## 解决问题的思路

![](/img/thread-model.png)
从上面的分析可以得出，libavformat的IO操作是不能被打断，且```av_read_frame```和```av_seek_frame```都是同步调用，所以调用必须在另外一个线程里，解决方案如下：

1. ```FfmpegExtrator```使用```HandlerThread```创建Seek线程，并创建发送message的主线程Handler；
2. 当read或seekTimeUs时，在Seek线程异步调用```FfmpegParserJni.readFrame```或```FfmpegParserJni.seekTo```，并阻塞load线程；
3. 最终通过native函数调用到libavformat的```av_read_frame```或```av_seek_frame```函数，此时进入了ffmpeg的IO模式；
4. libavformat将根据所解析的容器协议进行数据read和seek，read操作直接回调到```FfmpegParserJni.read```，从```ExtractorInputWrapper```里读取数据；
5. 也有可能调用IO Seek，seek操作回调到```FfmpegParserJni.seek```，此回调先唤醒Load线程，告知ffmpeg需要seek的位置；
6. ```FfmpegExtractor.read```返回RESULT_SEEK，并将要seek的位置放在```seekPosition.position```；
7. ```ProgressiveMediaPeriod.ExtractingLoadable.load```将会重新创建一个新的```ExtractorInput```，数据的位置就是```seekPosition.position```。

通过上面的步骤实现了，FfmpegExtractor和ffmpeg异步IO对接，实现更多容器的解复用。
