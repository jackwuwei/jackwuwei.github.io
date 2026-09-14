---
layout:     post
title:      "墨水屏Homelab看板"
subtitle:   "Kindle、4.2寸墨水屏和共用的FastAPI后端"
date:       2026-09-12 13:13:14 +0800
author:     "Jack"
header-img: "img/eink-dashboard-bg.jpg"
tags:
    - IOT
    - Homelab
    - E-ink
---
## 题记
家里的机房越来越热闹：绿联NAS、飞牛NAS、树莓派、华硕路由器，再加上Claude订阅的用量额度，想看一眼状态就得打开好几个网页。墨水屏不发光、不刺眼、断电也能保留画面，最适合做一块常亮的看板。于是做了两块屏：一台越狱的Kindle Paperwhite 2，和一块4.2寸的ESP8266墨水屏，背后共用同一个FastAPI后端，代码都开源在Github上：

* [kindle-dashboard](https://github.com/jackwuwei/kindle-dashboard)：越狱Kindle的看板客户端 + 3D打印底座；
* [eink-dashboard](https://github.com/jackwuwei/eink-dashboard)：4.2寸墨水屏的ESP8266固件 + 3D打印外壳；
* [dashboard-backend](https://github.com/jackwuwei/dashboard-backend)：两块屏共用的服务端，以git submodule的形式挂在两个客户端仓库的```server/```目录下，改一次两边都生效。

![4.2寸墨水屏实拍](/img/eink-dashboard-photo.jpg)

## 演示视频
<video src="/img/eink-dashboard-promo.mp4" poster="/img/eink-dashboard-promo-poster.jpg" controls playsinline preload="metadata" style="display:block;width:100%;max-width:360px;margin:0 auto;"></video>

## 整体架构
```
[Glances]  [Home Assistant]  [Claude Usage]  [Open-Meteo]   数据源
    |             |                |              |
    +-------------+-------+--------+--------------+
                          |
                          v
                 [dashboard-backend]                  NAS上的Docker，定时采集并缓存
                          |
          +---------------+---------------+
          |                               |
  GET /dashboard.png              GET /api/eink.json
          |                               |
          v                               v
      [Kindle]                        [ESP8266]
```

两块屏的思路完全不同：
* **Kindle**性能够、屏幕大，但越狱后能跑的东西有限，所以**服务端把整张图画好**，Kindle只负责下载图片并刷到屏幕上；
* **ESP8266**只有几十KB内存、靠电池供电，下载和解码一张图太费电，所以**服务端只下发精简JSON**，由设备自己用位图字库画出来。

## 后端：dashboard-backend

### 数据采集
* **服务器状态**：每台机器装一个[Glances](https://github.com/nicolargo/glances)，通过REST API取CPU、内存、温度和运行时间，存活探测支持```ping```/```tcp```/```http```/```exec```四种方式；
* **路由器**：路由器装不了Glances，就走Home Assistant中转，HACS里装AsusRouter集成，拿到CPU、内存和温度实体；
* **环境读数**：温湿度、机柜风扇等来自Home Assistant，用长期访问令牌读取；
* **Claude订阅用量**：显示5小时窗口、本周全部、按模型的周限额和重置时间，服务端自持OAuth凭据，浏览器打开```/claude-auth```授权一次，之后自动续期；
* **天气**：默认用免Key的Open-Meteo，按服务端公网IP自动定位，也可以手动指定城市或改用心知天气，30分钟拉一次。

### 设计要点
* **预采集 + 缓存**：后台线程每```refresh_seconds```采集一次，HTTP接口只返回已经采好的结果，客户端请求永远秒回，不用等上游接口；
* **不用无头浏览器**：Kindle的看板直接用Pillow画成8位灰度PNG，镜像小、纯黑白高对比，对墨水屏很友好；
* **配置热生效**：```config.yaml```每个采集周期都会重读，改完不用重启容器；
* **密钥不入库**：```HA_TOKEN```放在```.env```里，通过环境变量注入。

### 墨水屏的JSON接口
给ESP8266的```/api/eink.json```字段名都尽量短，整个响应不到2KB：
```json
{
  "rev": "内容哈希，不含时间戳",
  "next_poll": 300,
  "alert": 0,
  "servers": [{"n": "绿联 NAS", "ok": 1, "cpu": 6, "mem": 63, "temp": 51, "up": "20.9天"}],
  "claude": [{"l": "5 小时窗口", "pct": 36, "r": "02:19 重置"}],
  "env": [{"i": "temp", "l": "室内温度", "v": "28.9°C"}],
  "w": {"...": "天气"}
}
```
* 设备请求时带上上一次的```rev```，内容没变化服务端直接返回```304```，设备连屏幕都不用碰；
* ```next_poll```由服务端决定，白天5分钟，夜间30分钟，有设备掉线时可以加快轮询；
* 设备顺便在请求参数里上报电池电压和温湿度，可以在```/api/eink/device```查看。

## Kindle看板：kindle-dashboard

<p align="center"><img src="/img/kindle-dashboard-screen.png" width="420" alt="Kindle看板"></p>

### 硬件
* Kindle Paperwhite 2（758×1024，固件5.12.2.2），已越狱，装好KUAL和USBNetwork；
* 布局按1072×1448的设计稿绘制，输出时按配置缩放，PW3/PW4的宽高比一样，换设备只需要改两个数字。

### 工作方式
* ```dash.sh```是主循环，每5分钟用```curl```从服务端拉一次PNG，用```eips -g```刷到屏幕上，每5次做一次全刷清残影；
* 启动时先停掉Kindle的系统界面，不然右上角的状态栏每分钟会叠画到看板上，顺便关掉前光省电；
* ```touch.py```监听触摸事件，点屏幕右下角的「退出」会弹出确认框，确认后恢复Kindle的系统界面；
* 离线的设备整张卡片黑底反白，一眼就能看出谁挂了。

### 踩过的坑
* **KUAL启动后白屏**：KUAL启动的进程属于framework的进程树，停掉framework会把自己也一起杀掉，```start.sh```必须用```setsid```起一个独立会话；
* **停framework也会白屏**：framework停止时会擦屏，所以必须**先停系统界面，再画第一屏**；
* **状态栏叠画**：5.12固件上```disableEnablePillow```无效，只能```initctl stop framework```；
* **℃显示空白**：文泉驿字体里U+2103是空字形，渲染时探测到空字形就自动退回```°C```；
* **退出后白屏2~3分钟**：这是正常现象，framework冷启动就是这么慢，所以退出时先画一张"正在恢复"的提示图。

### 3D打印底座
顺手用[build123d](https://github.com/gumyr/build123d)画了一个参数化的充电底座：15°后倾，直头micro-USB从槽底的直通孔捅进充电口，线从底座内部的暗道走出去，免支撑就能打印。充电口位置、电源键避让槽都是用卡尺实测的，换设备改参数重新生成即可。

![Kindle底座](/img/kindle-dashboard-dock.png)

## 4.2寸墨水屏：eink-dashboard

### 硬件
硬件用的是开源项目[weather-ink-screen](https://gitee.com/Lichengjiez/weather-ink-screen)的4.2寸板（revB-230621，Z96屏），原本是一块天气墨水屏：
* ESP8266（4MB Flash），深睡靠GPIO16接RST唤醒；
* 400×300的4.2寸黑白墨水屏；
* 时钟芯片、SHT30温湿度传感器、电池电压检测；
* 660mAh锂电池，TC4056充电。

硬件一点没改，在这块板子上重写了一套独立固件，屏幕驱动、时钟芯片、温湿度和电压检测的逻辑从原固件移植过来，原固件的整块Flash也做了备份，随时可以刷回去。

### 页面
一共6个页面，醒着的时候按键翻页：看板 → Claude用量 → 服务器详情 → 天气 → 万年历 → 设备信息。

<div style="display:grid;grid-template-columns:repeat(2,1fr);gap:12px;">
<img src="/img/eink-dashboard-dashboard.png" alt="看板" style="width:100%;margin:0;">
<img src="/img/eink-dashboard-claude.png" alt="Claude用量" style="width:100%;margin:0;">
<img src="/img/eink-dashboard-servers.png" alt="服务器详情" style="width:100%;margin:0;">
<img src="/img/eink-dashboard-weather.png" alt="天气" style="width:100%;margin:0;">
<img src="/img/eink-dashboard-calendar.png" alt="万年历" style="width:100%;margin:0;">
<img src="/img/eink-dashboard-device.png" alt="设备信息" style="width:100%;margin:0;">
</div>

* 顶栏的时间、温湿度、WiFi和电量都是设备本地的数据，服务端挂了也照常显示；
* 万年历的农历、二十四节气和节日全部在设备端查表，覆盖2019~2100年，和lunar_python逐日对比零差异，不依赖网络；
* 支持中英文界面，在配网页里切换。

### 省电策略
电池只有660mAh，省电是这个固件的核心，每分钟的唤醒流程大致如下：
1. 深睡到下一个整分钟，被RTC唤醒；
2. 读时钟芯片、SHT30和电池电压，这一步不开WiFi；
3. 如果到了轮询时间（默认5分钟，夜间30分钟），才连WiFi请求JSON，拿完马上关WiFi；
4. 顶栏每分钟局刷一次，内容区只有```rev```变化时才局刷，每小时整点全刷一次清残影；
5. 关闭屏幕电源，继续深睡。

按设计文档里的估算，每天大约耗电17mAh（HTTP）到20mAh（HTTPS），660mAh的电池能用**4~5周**：

| 项目 | 每天次数 | 每天耗电 |
|---|---|---|
| 分钟唤醒（不联网，局刷顶栏） | 1440 | ~4mAh |
| 联网轮询（快连 + JSON） | 288 | ~11mAh |
| HTTPS额外开销（会话复用） | 288 | ~3mAh |
| 内容区局刷 + 整点全刷 | ≤312 | ~0.8mAh |
| 深睡（0.035mA） | 全天 | ~0.8mAh |

### 配网
* 首次上电自动进入配网模式，开一个```ESP8266 E-Paper```热点，手机连上会自动弹出配置页（Captive Portal）；
* 填好WiFi和服务端地址，点「测试连接」看到```ok```再保存；
* 支持HTTP和HTTPS，HTTPS可以用内置的Let's Encrypt根证书，也可以填自签证书的指纹；
* 连公司、酒店这种要网页登录的WiFi，还支持门户自动登录，常见的Aruba门户已经内置了预设。

### 踩过的坑
* **必须用```ssl=all```编译**：```ssl=basic```的BearSSL没有ECDHE密钥交换，现代服务器一律握手失败；
* **华硕WiFi 6/7路由器连不上**：ESP8266默认802.11n模式关联后会被踢，固件连接前强制切到```WIFI_PHY_MODE_11G```；
* **开放网络不能带密码**：ESP8266的```WiFi.begin()```只要密码非空就把认证方式设成WPA，开放的访客WiFi明明扫得到却连不上；
* **按键和屏幕共用GPIO0**：屏幕初始化后GPIO0是输出，读按键时要临时切回输入，读完再恢复；
* **ADC读数偏高0.3V**：ESP8266在关闭射频的唤醒里ADC基准不准，所以电压只在联网那次唤醒里采样；
* **U8g2字库超内存**：原版库在ESP8266上没有把字库放进Flash，链接直接超RAM，需要打补丁。

### 3D打印外壳
仓库的```stl/```目录里有4.2寸板的外壳，就是上面实拍图里的那个，打印出来把板子和电池装进去就是一个完整的桌面摆件。

## 后续
两块屏摆在桌上，服务器的负载、温度，Claude还剩多少额度，抬头就能看到，比打开网页方便多了。欢迎到Github上Star和提Issue：[kindle-dashboard](https://github.com/jackwuwei/kindle-dashboard)、[eink-dashboard](https://github.com/jackwuwei/eink-dashboard)、[dashboard-backend](https://github.com/jackwuwei/dashboard-backend)。
