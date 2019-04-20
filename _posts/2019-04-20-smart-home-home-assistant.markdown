---
layout:     post
title:      "智能家居折腾记"
subtitle:   "Home Assistant大法好！"
date:       2019-04-20 19:12:00
author:     "Jack"
header-img: "img/home-assistant-bg.png"
tags:
    - IOT
    - Home Assistant
---
## 题记
做为一个懒癌晚期，家里的灯泡和开关懒得起身关，特别是躺在床上，所以装了小米生态的各种智能家居设备，为了能通过Siri和Google Assistant语音控制，用吃灰已久的树莓派3装上了Home Assistant，通过一顿猛如虎的骚操作之后，整个系统完美运行。

## 小米全家桶的优势
1. 价格便宜，比起其他品牌动则上千，实惠太多了；
2. 种类多，有ZigBee、Wifi和蓝牙设备，网关、开关、插座、人体传感器、灯泡、红外遥控等；
3. 配置简单，使用米家App快速完成设备的添加和设置。

## Home Assistant的作用
* Home Assistant是是一款基于 Python 的智能家居开源系统，支持众多品牌的智能家居设备，可以轻松实现设备的语音控制、自动化等；
* Home Assistant能接入海量的设备，支持的[设备及组件](https://www.home-assistant.io/components/)。

## 需求
* iPhone的Siri和Google Home Mini用语音控制分布在各房间的灯；
* 人体传感器、光线传感器与灯联动，实现夜灯的场景。

## 硬件
* [Raspberry Pi 3 Model B](https://item.taobao.com/item.htm?spm=0.7095261.0.0.29491debwhELXl&id=565703159178&src=raspberrypi)，一张至少8G的TF卡，安装Home Assistant，也可以安装在各种PC和系统上；
* [米家多功能网关](https://www.xiaomiyoupin.com/detail?gid=103292)，绿米(Aqara)有一个支持HomeKit的网关，但贵一倍，有了Home Assistant，就没有必要了；
* [Aqara智能墙壁开关（单火版）](https://www.xiaomiyoupin.com/detail?gid=284)，零火版需要有零线，家里的是单火线的，单火版比较通用；
* [米家智能插座(ZigBee版)](https://www.xiaomiyoupin.com/detail?gid=103074)，ZigBee版本比较简单，WiFi版需要额外取token，比较麻烦；
* [米家床头灯](https://www.xiaomiyoupin.com/detail?gid=101471)，这个只有WiFi版本，开局域网控制就能被Home Assistant控制；
* [米家门窗传感器](https://www.xiaomiyoupin.com/detail?gid=101464)，贴在门的两边，随时了解门的开关状态，也可以联动夜灯；
* [米家温湿度传感器](https://www.xiaomiyoupin.com/detail?gid=103153)，检测温度和湿度，可以跟空调和抽湿机联动；
* [Aqara人体传感器](https://www.xiaomiyoupin.com/detail?gid=730)，红外感应人体移动，光照度检测；
* [Aqara无线开关](https://www.xiaomiyoupin.com/detail?gid=101295)，可当门铃、报警器，支持四种控制模式，内置高精度加速度传感器；
* [Aqara无线开关（贴墙式）](https://www.xiaomiyoupin.com/detail?gid=102856)，无线开关，可以贴在任何地方，自定义控制各种设备。

## 软件
* Hassbian
* 米家App

## 安装步骤
* 建议安装Hassbian，难度适中，可操作性较强，按照[官方的安装指南](https://www.home-assistant.io/docs/installation/hassbian/)一步步来，没有什么难度；
* 重点是修改配置，基本按照官方的[组件介绍](https://www.home-assistant.io/components/)一步步操作；
    * 米家WiFi系列设备除Yeelight外，接入Home Assistant需从米家App里[获取token](https://www.home-assistant.io/components/vacuum.xiaomi_miio/#retrieving-the-access-token)；
    * 接入Google Assistant比较麻烦，需要https URL才能连接，从DDNS服务提供商获得一个域名，并取得[Let's Encrypt免费证书](https://certbot.eff.org/lets-encrypt/centos6-apache)，[官方教程](https://www.home-assistant.io/components/google_assistant/)。
* HomeKit已经内置支持，无需安装Homebridge。

## 截图
* Home Assistant后台![Home Assistant](/img/ha-front-end.png)
* Apple Home Kit![Home Kit](/img/home-kit-app.PNG)
* Google Home![Google Home](/img/google-home-app.PNG)