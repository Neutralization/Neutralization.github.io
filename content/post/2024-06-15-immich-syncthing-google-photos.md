---
title: "自建 Immich 相册服务同步白嫖 Google Photos"
date: 2024-06-15T15:56:00+08:00
categories:
    - DIY
tags:
    - immich
    - pixel
    - syncthing
    - google photos
---

Google Photos 有着 15GB 的免费空间上限，前几个月自己搞了 NAS 之后，看上了 Immich 相册，算是最贴近 Google Photos 界面的开源自建相册服务了，自己手头也正好还留着当初买的初代 Google Pixel，能够以原画质白嫖无限相册空间，相当于能够在本地备份的同时也拥有稳定的云备份。

考虑到稳定性，NAS 上并没有安装 APP 或者其他容器服务，单纯作为一个存储设备。Docker 服务都放在另一台小主机上运行。

计划是 TrueNAS 存储只提供 SMB 服务挂载，小主机上通过多个 Docker 分别部署 Immich 相册，以及用来同步照片到 Pixel 的 Syncthing 服务。另外手机肯定是7*24供电的状态，为了不用频繁物理操作，考虑部署 RustDesk 服务远程控制手机，同样采用 Docker 的方式部署在小主机上。

<!-- more -->

# 软硬件配置

TrueNAS 服务器 CPU 是 N100， 16GB 内存， 16TB*2 HDD（Raid 1）

小主机 CPU 同样是 N100，24GB 内存，256GB SSD，Debian 系统

初代 Pixel，未 Root

## TrueNAS 配置

![TrueNAS_SMB](/pic/2024-06-15-immich-syncthing-google-photos/TrueNAS_SMB.png)

非常简单，TrueNAS 上的配置就结束了，继续配置小主机。

## 服务器配置

首先为了让之后 Immich 从 NAS 上读写，需要把 SMB 开机自动挂载路径。SSH 登陆后执行：

> sudo apt install cifs-utils psmisc  
> nano ~/.credentials  

对应修改 SMB 共享的用户名和密码

```conf
username=smbusername
password=smbpassword
```

> sudo chown username .credentials  
> sudo chmod 600 .credentials  

测试挂载

> sudo mkdir /mnt/Immich  
> sudo mount -t cifs -o credentials=~/.credentials //truenas.ipaddress/Immich /mnt/Immich

没有问题则暂时卸载

> sudo umount -t cifs /mnt/Immich

修改 `/etc/fstab`

> sudo nano /etc/fstab

添加如下内容

```conf
# Immich Storage
//truenas.ipaddress/Immich    /mnt/Immich    cifs  credentials=/home/username/.credentials    0    0
```

应用修改完成挂载

> sudo systemctl daemon-reload  
> sudo mount -av

# Docker 配置

## [安装 Docker](https://docs.docker.com/engine/install/debian/)

```bash
sudo apt-get update  
sudo apt-get install ca-certificates curl  
sudo install -m 0755 -d /etc/apt/keyrings  
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc  
sudo chmod a+r /etc/apt/keyrings/docker.asc  

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## 部署 Immich

官方推荐采用 [Docker Compose](https://v1.106.4.archive.immich.app/docs/install/docker-compose) 的方式安装，跟着教程走并没有什么难度。
```bash
mkdir ~/immich-app
cd ~/immich-app

wget -O docker-compose.yml https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
wget -O .env https://github.com/immich-app/immich/releases/latest/download/example.env
wget -O hwaccel.transcoding.yml https://github.com/immich-app/immich/releases/latest/download/hwaccel.transcoding.yml
wget -O hwaccel.ml.yml https://github.com/immich-app/immich/releases/latest/download/hwaccel.ml.yml
```

编辑 `.env` 文件，修改照片的上传路径

```conf
TZ=Asis/Shanghai
UPLOAD_LOCATION=/mnt/Immich
DB_DATA_LOCATION=./postgres
```
数据库路径不变，之所以这样是因为数据库有频繁的读写，像 16T 容量的机械盘基本都是充气盘，会频繁的发出炒豆子的声音。  
换言之只要不是充气盘或者距离远听不见声音就无所谓了，可以改成 `/mnt/Immich/postgres`

> docker compose up -d

启动容器后，访问 http://server.ipaddress:2283/ 登录，在`设置`/`机器学习设置`/`智能搜索`/`CLIP模型`，修改为支持多语言的`XLM-Roberta-Large-Vit-B-16Plus`。

为了便于管理，启用`设置`/`存储模板`功能。

在`设置`/`视频转码设置`中，`支持的音频编解码器`勾选 AAC/MP3/Opus，`支持的视频编解码器`勾选 H.264/HEVC/VP9/AV1，因为我现在的主力机都支持这些编码，没有必要转码。

## 部署 Syncthing

先建立需要的目录
```bash
sudo mkdir -p /var/syncthing/Immich
mkdir ~/syncthing
cd ~/syncthing
nano docker-compose.yml
```

按照[官方指导](https://github.com/syncthing/syncthing/blob/main/README-Docker.md)编写 Dockerfile

docker-compose.yml
---
```dockerfile
---
services:
  syncthing:
    image: syncthing/syncthing
    container_name: syncthing
    hostname: Dorothy
    environment:
      - PUID=1000
      - PGID=1000
    volumes:
      - /var/syncthing:/var/syncthing
      - /mnt/Immich/library/admin:/var/syncthing/Immich
    network_mode: host
    restart: unless-stopped
```

> docker compose up -d

启动容器后，访问 http://server.ipaddress:8384/ 登录，添加文件夹

![Syncthing_Immich](/pic/2024-06-15-immich-syncthing-google-photos/Syncthing_Immich.png)

在`高级`书签中选择`文件夹类型`为`仅发送`

![Syncthing_Immich_extra](/pic/2024-06-15-immich-syncthing-google-photos/Syncthing_Immich_extra.png)

这样 Syncthing 的初步配置就结束了

## 部署 RustDesk

同样按照[官方指导](https://rustdesk.com/docs/en/self-host/rustdesk-server-oss/docker/)建立 Dockerfile

docker-compose.yml
---
```dockerfile
---
services:
  hbbs:
    container_name: hbbs
    image: rustdesk/rustdesk-server:latest
    command: hbbs
    volumes:
      - ./data:/root
    network_mode: "host"

    depends_on:
      - hbbr
    restart: always

  hbbr:
    container_name: hbbr
    image: rustdesk/rustdesk-server:latest
    command: hbbr
    volumes:
      - ./data:/root
    network_mode: "host"
    restart: always
```
启动服务

> docker compose up -d

获取服务端 API KEY

> docker container logs hbbs | grep Key

# 配置 Pixel 自动任务

首先安装 [Syncthing](https://f-droid.org/packages/com.nutomic.syncthingandroid/) 和 [RustDesk](https://f-droid.org/packages/com.carriez.flutter_hbb/) 的客户端

然后是用来实现自动任务的 [Tasker](https://play.google.com/store/apps/details?id=net.dinglisch.android.taskerm) 和 [TouchTask](https://play.google.com/store/apps/details?id=com.balda.touchtask)

## 连接 RustDesk

修改对应的服务器 IP 地址和 KEY

![RustDesk](/pic/2024-06-15-immich-syncthing-google-photos/RustDesk.png)

允许 IP 地址访问，保持后台服务，允许自启动

![RustDesk_Setting](/pic/2024-06-15-immich-syncthing-google-photos/RustDesk_Setting.png)

设置固定密码，使用固定密码

![RustDesk_passwd](/pic/2024-06-15-immich-syncthing-google-photos/RustDesk_passwd.png)

勾选全部权限后启动服务

![RustDesk_Service](/pic/2024-06-15-immich-syncthing-google-photos/RustDesk_Service.png)

在自己的主机上也装好 Rustdesk，修改对应的服务器 IP 地址和 KEY 信息后，现在就可以接上充电线放在一边，通过电脑远程操作 Pixel 了。

## 连接 Syncthing

选择 Syncthing 的`设备`然后点击右上角`+`，添加设备

![Synthing_device](/pic/2024-06-15-immich-syncthing-google-photos/Synthing_device.png)

回到 http://server.ipaddress:8384/ 登录，点击识别的设备标识，通过手机扫码或者直接输入完整设备标识字符串进行配对

![Synthing_id](/pic/2024-06-15-immich-syncthing-google-photos/Synthing_id.png)

然后网页端会出现配对许可的提示，同意即可。

接着编辑之前共享的目录，共享给配对的设备。

![Synthing_share](/pic/2024-06-15-immich-syncthing-google-photos/Synthing_share.png)

回到手机，同意接收共享，将目录选为照相机的默认目录 `/storage/emulated/0/DCIM/Camera`，目录种类更改为仅接收。

这样就实现了 Immich 从 NAS 往 Pixel 的单向同步，之后在 Pixel 上清除本地缓存时也不会导致 NAS 上的文件被同步删除，Pixel 也会自动将同步收到的照片上传至 Google Photos。

## 配置 Tasker

为了避免手机存储满了之后无法继续同步文件，需要定期运行相册 APP 释放空间，这里就可以使用 Tasker 来完成。

主要的操作一共三种：

1. `+`/`任务`/`等待`
2. `+`/`程序`/`启动应用/结束应用`
3. `+`/`插件`/`TouchTask`/`Actions`

![Tasker_add_1](/pic/2024-06-15-immich-syncthing-google-photos/Tasker_add_1.png)![Tasker_add_2](/pic/2024-06-15-immich-syncthing-google-photos/Tasker_add_2.png)

![Tasker_add_3](/pic/2024-06-15-immich-syncthing-google-photos/Tasker_add_3.png)![Tasker_add_4](/pic/2024-06-15-immich-syncthing-google-photos/Tasker_add_4.png)

屏幕点击操作需要的坐标数据可以通过打开`开发者选项`/`输入`/`指针位置`来获取

然后按照

等待/结束相册/启动相册/等待/点击头像/释放空间/确定释放/等待/启动Tasker（回前台）

的逻辑添加操作步骤即可

![Tasker](/pic/2024-06-15-immich-syncthing-google-photos/Tasker.png)

之后添加配置文件，让任务定时执行

![Tasker_config](/pic/2024-06-15-immich-syncthing-google-photos/Tasker_config.png)![Tasker_timer](/pic/2024-06-15-immich-syncthing-google-photos/Tasker_timer.png)

## FakeStandby

为了不需要 Root 权限，Pixel 设置了不锁屏，但保持亮屏一是容易造成烧屏，二十造成无意义的耗电，为此可以安装 [FakeStandby](https://f-droid.org/packages/android.jonas.fakestandby/) 这款应用，通过显示黑屏实现伪锁屏待机。

FakeStandby
![FakeStandby](/pic/2024-06-15-immich-syncthing-google-photos/FakeStandby.png)

# 导入 Google Photos

最后，对于已经在 Google 相册中的照片，可以通过 Google 导出服务和 immich-go 工具便捷的导入 Immich。

访问 https://takeout.google.com/settings/takeout/custom/photos 在要包含的数据中选择 `Google 相册`，继续，然后再目标位置选择`添加到 OneDrive`，文件类型选择`.zip`，大小选择`50GB`。

视具体网络情况，选择通过 OneDrive 可以直连下载，节约大量的代理流量。

网页端登录 Immich，点击头像/`账号设定`/`API Keys`/`新API Key`，复制下 Key 字符串。

Google 导出结束后，下载[immich-go](https://github.com/simulot/immich-go)，将解压的 exe 文件和 Google 导出的 zip 放在同一目录中，打开命令行 cd 到此目录并执行

> .\immich-go.exe -server=http://immich.ipaddress:2283 -key=API_KEY_STRING upload -google-photos takeout-*.zip

即可将过去的相册内容导入至 Immich。

## EOF

Google 目前的政策只有初代 Pixel 能够上传原始画质享有无限空间，后代的 Pixel 也只有最早的几代能够上传省空间画质且享有无限空间。

目前闲鱼 2~300 收一个初代 Pixel 还是合算的，不过哪天 Pixel 电池顶不住了，是换电池还是换机，亦或者直接抛弃 Google 相册，就见仁见智了。