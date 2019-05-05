---
layout:     post
title:      "Android音视频处理技术"
subtitle:   "Android MediaCodec及ffmpeg"
date:       2019-05-05 1:23:00
author:     "Jack"
header-img: "img/ffmpeg_banner.png"
tags:
    - Codec
---
## 编解码器的处理流程
* 解码器的工作流程如下图![decoder](/img/decoder.JPG)
* 编码器的工作流程如下图![encoder](/img/encoder.JPG)

## 关于MediaCodec
Android 4.1以后引入了[MediaCodec](http://developer.android.com/reference/android/media/MediaCodec.html)，终于在Android SDK层面支持了音视频编解码。

## 相关资源
官方SDK文档中没有过多描述，网上系统介绍的文章比较少，找到一些比较有用的参考代码。
* [Android MediaCodec stuff](http://bigflake.com/mediacodec/)，这个网站上提供了一些调用MediaCodec的CTS代码，是从Android的系统源代码里抽出来到，能大致了解到MediaCodec的使用方法。
* [grafika](https://github.com/google/grafika)，Google维护的一套使用MediaCodec的Sample Code，基于Android 4.3以上的系统，较系统的介绍了如何使用SurfaceTexture、TextureView，从Camera里的画面通过纹理绘制到OpenGL ES，使用Shader还能实现滤镜叠加效果，以及OpenGL ES渲染的视频直接输出到MediaCodec进行编码。![MediaCodec](/img/continuous_capture_activity.png)
详细内容请参考[Android官方文档](https://source.android.com/devices/graphics/architecture.html#continuous-capture)。

## 实际开发建议
* 由于MediaCodec本身对容器的支持比较少，兼容性不好，不建议使用其带的MediaExtractor，建议直接使用ffmpeg的libavformat；
* 同样MediaMuxer也有限制，而且只在Android 4.3以上的版本才有，干脆直接使用ffmpeg代替，能合成更多的容器；
* Demuxer和Muxer只是对压缩后的音视频数据进行分离和打包，采用ffmpeg不会有性能的问题，但如果对音视频进行解码和编码，优先使用MediaCodec。![MediaCodec Workflow](/img/media-codec-workflow.jpg)

## 其他
在Android NDK r10里，终于加入对MediaCodec的支持，include目录下多了一个media文件夹，里面包含了NdkMediaCodec.h, NdkMediaMuxer.h等文件。
希望MediaCodec越来越给力，结合ffmpeg中的demuxer和muxer使用，能搞定绝大部分主流视频的解码和编码。