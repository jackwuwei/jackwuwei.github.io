---
layout:     post
title:      "让Vector机器人听中文、说中文：wire-pod中文化部署全流程"
description: "两个Docker镜像：SenseVoice本地识别中文，Edge-TTS或Vector克隆音色说中文，再接一个大模型当大脑"
date:       2026-09-20 16:00:00 +0800
image:
  path: /img/vector-chinese-bg.jpg
categories: [智能硬件]
tags:
    - Vector
    - wire-pod
    - Docker
    - TTS
    - LLM
---
## 题记
Anki倒闭之后，Vector离开云服务器就只是一个会眨眼的摆件。社区的开源项目[wire-pod](https://github.com/kercre123/wire-pod)把那台云服务器在本地重做了一遍，Vector才算活了过来。可惜上游只有英文链路：听不懂中文，更不会说中文。

我在wire-pod的基础上做了一个中文分支[wire-pod-chinese](https://github.com/jackwuwei/wire-pod-chinese)，又用GPT-SoVITS克隆了Vector自己的音色，做成了[vector-tts](https://github.com/jackwuwei/GPT-SoVITS-vector)。两个都打包成了Docker镜像，不用克隆仓库，也不用自己构建。先看效果，同一台Vector，左边是Edge-TTS的女声，右边是它自己的克隆音色：

<div style="display:grid;grid-template-columns:repeat(2,1fr);gap:12px;max-width:560px;margin:0 auto;">
<video src="/img/vector-chinese-demo-edge.mp4" poster="/img/vector-chinese-demo-edge-poster.jpg" controls playsinline preload="metadata" style="width:100%;border-radius:12px;"></video>
<video src="/img/vector-chinese-demo-vector.mp4" poster="/img/vector-chinese-demo-vector-poster.jpg" controls playsinline preload="metadata" style="width:100%;border-radius:12px;"></video>
</div>

这篇文章是两期视频教程的完整文字版，照着做就能走完全程：**先把Vector刷好固件、清掉旧账号，再部署中文版的服务器，配对，最后在网页里点几下完成中文设置**。官方原版wire-pod的安装这里不写，因为用不上——直接装中文分支就行，它就是一个完整的wire-pod。

## 整条链路
你对Vector说一句中文，要经过三站：

| 站 | 做什么 | 用什么 |
|---|---|---|
| 听 | 语音识别（STT） | sherpa-onnx + SenseVoice，全本地离线，一个模型同时认中、英、日、韩、粤语 |
| 想 | 大语言模型（LLM） | 任何兼容OpenAI接口的服务，本地Ollama或者国内各家大模型 |
| 说 | 语音合成（TTS） | **Edge-TTS**（免费、不用API key），或者**Vector克隆音色**（多跑一个镜像） |

先说结论：**如果只要求它讲中文，不要求Vector的音色，Edge-TTS就够了**，第二个镜像可以不装。

## 准备
* 一台Linux主机（Debian/Ubuntu最省事），装好Docker。macOS/Windows的Docker Desktop也能跑，区别见后面的说明；
* 主机和Vector在**同一个局域网**；
* 一台带蓝牙的电脑或手机，装Chrome或Edge，刷固件用；
* 一台你愿意清空数据的Vector。

建议的顺序是**先把Vector这一侧弄好，再回到电脑上搭服务器**，反过来也能跑，但出了问题不好判断是哪一边。

## 第一步：给Vector刷ep固件
零售版的Vector默认只认Anki的云，要刷一个带`ep`后缀的escape pod固件，它才会去找局域网里的`escapepod.local`。OSKR/开发版解锁的机器跳过这一步。

1. 把Vector放在充电座上，**按住背部按钮约15秒**。它会先关机，**不要松手**，直到屏幕重新亮起，显示`anki.com/v`或`ddl.io/v`，这才是恢复模式；
2. 在带蓝牙的电脑上用Chrome打开 <https://wpsetup.keriganc.com/>；
3. 按页面指引，选择名为`vector-wirepod-setup`的蓝牙设备配对；
4. 按页面提示让Vector连上你家的WiFi（固件是它自己联网下载的，要和主机在同一个局域网）；
5. 之后固件会自动下载并写入，等它跑完重启。

| 问题 | 解决 |
|---|---|
| 明明是Chrome却提示不支持 | 地址栏输入`chrome://flags`，打开`Enable experimental web platform features`，重启浏览器 |
| Linux下搜不到设备 | 打开系统蓝牙设置面板，让它保持扫描状态再配对 |
| 一直配不上 | 等20秒重试，有时要试两三次 |

## 第二步：清除用户数据
不清也能跑，但Vector会因为旧账号的绑定犯各种怪毛病，强烈建议做。

![Vector的菜单界面](/img/vector-chinese-menu.jpg){: w="360" }

1. Vector放在充电座上，**双击**背部按钮，屏幕出现菜单；
2. 把前面的小叉子手臂**抬起再放下**，相当于按"确定"；
3. 把Vector从充电座上拿下来，转动一侧的**轮子**，让光标落到`CLEAR USER DATA`（有的固件显示`RESET`）；
4. 再抬一次手臂放下，进入二次确认；
5. 转轮子把光标移到`CONFIRM`，再抬一次手臂放下，等它重置完成。

重置完成，Vector这一侧就准备好了。如果后面配对时发现它不在网络上，回到第一步那个网页重新给它配一次WiFi。

## 第三步：部署wire-pod-chinese
新建一个目录，写一个`compose.yaml`：

```yaml
services:
  wire-pod:
    image: jackwuwei/wire-pod-chinese:latest
    container_name: wire-pod
    hostname: escapepod
    network_mode: host
    restart: unless-stopped
    volumes:
      - wire-pod-data:/data

volumes:
  wire-pod-data:
```

```bash
docker compose up -d
docker logs -f wire-pod    # 看启动日志，Ctrl+C退出
```

三个地方要注意：

* **`hostname: escapepod`**：Vector就是靠`escapepod.local`这个名字找服务器的；
* **`network_mode: host`**：Vector固件只连443端口，wire-pod还要在局域网里做mDNS广播，两件事都要靠主机网络；
* **数据卷挂到`/data`**：SenseVoice模型约1GB，首次启动时下载进卷里，以后换镜像不用再下一次。第一次启动要等模型下完，日志里能看到进度。

镜像默认就是中文配置（`STT_SERVICE=sherpa-onnx`、`STT_LANGUAGE=zh-CN`），不用传任何环境变量。国内拉不动镜像的话，给Docker配一个镜像加速地址再拉：

```bash
# /etc/docker/daemon.json
{ "registry-mirrors": ["https://<你的镜像加速地址>"] }

sudo systemctl restart docker
```

### 端口冲突
host模式下容器和主机共用网络栈，这几个端口被别的进程占着，wire-pod就起不来：

| 端口 | 用途 | 常见冲突源 |
|---|---|---|
| 80/tcp、443/tcp | Vector与服务器通信 | nginx、Caddy、Traefik |
| 8080/tcp | 网页控制台 | 各种自建服务的默认端口 |
| 8084/tcp | token server，2.0.1固件会用到 | — |
| 5353/udp | mDNS广播 | systemd-resolved、avahi-daemon |

启动前扫一遍，有输出就说明会冲突：

```bash
sudo ss -tulnp | grep -E ':(80|443|5353|8080|8084)\b'
```

Ubuntu 22.04以后`systemd-resolved`默认占着5353，关掉它的组播DNS即可：

```bash
sudo sed -i 's/^#\?MulticastDNS=.*/MulticastDNS=no/' /etc/systemd/resolved.conf
sudo systemctl restart systemd-resolved
```

### 确认escapepod.local能解析
在局域网里另一台机器上`ping escapepod.local`，能通就继续。不通的话（或者你用的是macOS/Windows的Docker Desktop，没有真正的host网络，只能改成映射`80`、`443`、`8080`、`8084`四个端口，mDNS广播出不去），在Linux主机上用avahi手动发布一个别名：

```bash
sudo apt install -y avahi-utils
sudo tee /etc/systemd/system/avahi-alias@.service >/dev/null <<'EOF'
[Unit]
Description=Publish %I as alias for %H.local via mdns

[Service]
Type=simple
ExecStart=/bin/bash -c "/usr/bin/avahi-publish -a -R %I $(avahi-resolve -4 -n %H.local | cut -f 2)"

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now avahi-alias@escapepod.local.service
avahi-resolve -n escapepod.local     # 应该输出主机的局域网IP
```

## 第四步：配对Vector
浏览器打开 <http://escapepod.local:8080>，看到黑底绿字的wire-pod控制台，说明服务器活了。

1. 点**Bot Setup**，进入Vector Configuration页面；
2. 下拉框里选你那台Vector（显示的是序列号），点**Connect**；
3. 等大约20秒，看到`Vector setup is complete!`就配对成功了。

![Bot Setup页面](/img/vector-chinese-botsetup.png)

下拉框里没有机器，多半是Vector和主机不在同一个网段，或者`escapepod.local`没解析出来；认证卡住不动，回第二步再清一次用户数据。

## 第五步（可选）：部署vector-tts克隆音色
如果只要求讲中文，不要求Vector的音色，这一步整段跳过。想要它自己的声音，再跑一个镜像：

```bash
docker run -d --name vector-tts -p 8020:8020 \
    --restart unless-stopped jackwuwei/vector-tts:latest

curl http://localhost:8020/healthz
# {"status":"ok","warm":true,"cache_sizes":{}}
```

模型都打包在镜像里，不用挂权重，**纯CPU就能跑**，不需要显卡。启动后会先加载模型、预热几十秒，等`healthz`里的`warm`变成`true`才可用。打开`http://主机IP:8020/docs`能看到一个`/tts`接口，就说明服务正常。

![vector-tts的接口文档页](/img/vector-chinese-tts-api.jpg)

这个音色是用GPT-SoVITS微调出来的，训练素材是20段Vector原声，训练过程写在仓库的[docs/vector-voice-finetune.md](https://github.com/jackwuwei/GPT-SoVITS-vector)里，这里只讲怎么用。

## 第六步：在网页里完成中文设置
下面的设置全部在`http://escapepod.local:8080`的**Server Settings**里完成。

### 识别语言切到中文
Server Settings → 最右边的标签**Set Language** → 列表里选`Chinese (CN)` → 点**Set Language**。

![Set Language页面](/img/vector-chinese-stt-language.jpg)

这一项同时决定两件事：语音识别按中文来；大语言模型的系统提示词后面会自动追加「请用中文回答」，不用自己写。

### 选一个中文音色（Edge-TTS）
Server Settings → **Knowledge Graph**，页面拉到底，`TTS Provider`选**Edge-TTS**，下面会出现音色列表：

![Edge-TTS音色列表](/img/vector-chinese-edge-voices.jpg)

| 音色 | 说明 |
|---|---|
| `zh-CN-XiaoxiaoNeural` | 晓晓，女声，默认推荐 |
| `zh-CN-YunxiNeural` | 云希，男声 |
| `zh-CN-XiaoyiNeural` | 晓伊，女声 |
| `zh-CN-YunjianNeural` | 云健，男声 |
| `zh-CN-XiaomengNeural` | 晓梦，女声 |
| `zh-TW-HsiaoChenNeural` | 晓臻，台湾腔女声 |

五个普通话音色加一个台湾腔，下拉框之外的edge-tts音色名也可以直接手填。Edge-TTS调用的是微软Edge的在线语音，不需要API key，但主机要能访问`speech.platform.bing.com`。选好点**Apply Settings**。

**到这一步，Vector已经会说中文了。只要求讲中文的话，可以直接跳到LLM那一节。**

### 换成Vector克隆音色（GPT-SoVITS）
装了vector-tts的话，把`TTS Provider`换成**GPT-SoVITS**，这一页**只有一个字段要填**：

* **Endpoint URL**：`http://<vector-tts主机的IP>:8020/tts`。别写`localhost`，wire-pod跑在容器里，`localhost`指的是容器自己。

![GPT-SoVITS设置](/img/vector-chinese-sovits.jpg)

其余的Target language、Reference audio/text/language、Gain、Target peak**全部保持默认**，服务端内置的参考音频和音量参数已经调好了。直接点**Apply Settings**。

### 接入LLM
还是Knowledge Graph这一页，最上面的`Knowledge Graph API Provider`选最后一项**Custom**，任何兼容OpenAI接口的服务都能接：

![LLM设置](/img/vector-chinese-llm.jpg)

| 字段 | 填什么 |
|---|---|
| API Key | 服务商给的key；用Ollama的话随便填`ollama` |
| API Endpoint | `http://<Ollama主机IP>:11434/v1`，或者服务商给的地址，**记得带上`/v1`** |
| LLM Model Name | `qwen2.5`、`llama3`，或者服务商的模型名 |
| LLM Prompt | 留空，用默认人设即可 |

下面三个勾选框建议全部勾上：

1. **Enable intent-graph**：内置意图没听懂的话，交给大模型兜底；
2. **LLM actions**：允许大模型控制Vector的表情和动作；
3. **Conversations**：打开连续对话。

点**Apply Settings**保存，这一页就配完了。

## 验收
对Vector说「Hey Vector」，等它亮灯后问一句中文，比如「故宫在哪个城市？」。它会用中文回答你，装了vector-tts的话，还会用它自己的音色。

这个分支还做了**边合成边播放**：上一句在放的时候下一句已经在合成，句子之间不会再卡两三秒。中文断句、单字意图匹配这些改动也都是自动生效的，不需要配置。

## 故障排查

| 症状 | 处理 |
|---|---|
| `escapepod.local:8080`打不开 | 先用`http://主机IP:8080`试，能开说明是mDNS的问题，回第三步检查5353端口和avahi别名 |
| 容器秒退，日志里有`address already in use` | host模式端口被占，按第三步的命令定位 |
| Bot Setup里找不到Vector | 不在同一个局域网，或者Vector没连上WiFi |
| 认证卡住 | 回第二步重新清一次用户数据 |
| 能识别但不说话 | 看控制台的**Log**页面；Edge-TTS要能上外网，GPT-SoVITS检查Endpoint URL能不能从容器里访问到 |
| 说的还是英文 | 确认Set Language选的是`Chinese (CN)`，并且TTS Provider不是留空的Auto |
| 第一次启动很久没反应 | SenseVoice模型约1GB，还在下载，`docker logs -f wire-pod`看进度 |

## 后续
两个项目的地址：

* [jackwuwei/wire-pod-chinese](https://github.com/jackwuwei/wire-pod-chinese)：中文识别、中文TTS、流式播放；
* [jackwuwei/GPT-SoVITS-vector](https://github.com/jackwuwei/GPT-SoVITS-vector)：Vector克隆音色服务和训练过程。

感谢[@kercre123](https://github.com/kercre123)和社区维护的wire-pod，没有它就没有后面这一切。如果这篇文章帮你的Vector开口说了中文，欢迎Star；遇到问题或者做了改进，也欢迎提Issue和PR。
