---
title: "树莓派搭建网络摄像头"
date: 2024-03-15T16:35:00+08:00
categories:
    - 杂记
tags:
    - 树莓派
    - nginx
    - hls
---

年前迭代了 3D 打印机，换了拓竹的 A1 mini。感慨技术迭代是真的快，Ender 3 真的是完全落后时代了，打算（已经）拆了改造成 PET 回收装置。

对应着原本担当[远程控制作用的树莓派](https://naizi.ch/2021/08/28/升级-3d-打印机固件及远程打印方案/)又双叒叕闲置了，再次折腾点东西玩。

树莓派是 Raspberry Pi 3B+ ，现在已经算是老东西了。摄像头放弃了专用的排线摄像头，排线的物理距离太短还容易伤，换了另一个闲置的 USB 摄像头，型号是 Logitech C920。

主要参考了 [Setting up HLS live streaming server using NGINX](https://medium.com/@peer5/setting-up-hls-live-streaming-server-using-nginx-67f6b71758db) 的设置过程。
<!-- more -->

## 配置过程

### 安装软件包

> sudo apt-get update  
> sudo apt install nginx libnginx-mod-rtmp ffmpeg  

### nginx 配置

> sudo nano /etc/nginx/nginx.conf  

```conf
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections  1024;
}

rtmp {
    server {
        listen 1935;
        chunk_size 4000;

        application show {
            live on;
            hls on;
            hls_path /tmp/hls/;
            hls_fragment 3;
            hls_playlist_length 60;
        }
    }
}

http {
    sendfile off;
    tcp_nopush on;
    directio 512;
    default_type application/octet-stream;

    server {
        listen 8080;

        location / {
            add_header 'Cache-Control' 'no-cache';

            add_header 'Access-Control-Allow-Origin' '*' always;
            add_header 'Access-Control-Expose-Headers' 'Content-Length';

            if ($request_method = 'OPTIONS') {
                add_header 'Access-Control-Allow-Origin' '*';
                add_header 'Access-Control-Max-Age' 1728000;
                add_header 'Content-Type' 'text/plain charset=UTF-8';
                add_header 'Content-Length' 0;
                return 204;
            }

            types {
                application/dash+xml mpd;
                application/vnd.apple.mpegurl m3u8;
                video/mp2t ts;
            }

            root /tmp/;
        }
    }

    server {
        listen 80;
        location / {
            root /var/www/html/;
        }
    }
}
```

> sudo systemctl enable nginx  
> sudo nginx -t  
> sudo nginx -s reload  

### ffmpeg 推流配置

> sudo nano /etc/systemd/system/ffmpeg_stream.service  

```conf
[Unit]
Description=FFmpeg Streamer
After=network.target

[Service]
ExecStart=/bin/bash -c 'while true; do ffmpeg -hide_banner -thread_queue_size 512 -s 1280x720 -i /dev/video0 -thread_queue_size 1024 -f lavfi -i anullsrc -codec:v h264_v4l2m2m -b:v 8096k -c:a aac -b:a 128k -r 24 -f flv rtmp://localhost/show/stream || continue; done'
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

> sudo systemctl daemon-reload  
> sudo service ffmpeg_stream enable  
> sudo service ffmpeg_stream start  

### html 播放器页面

> sudo nano /var/www/html/index.html

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <title>Live Streaming</title>
    <link href="//vjs.zencdn.net/8.3.0/video-js.min.css" rel="stylesheet">
    <script src="//vjs.zencdn.net/8.3.0/video.min.js"></script>
</head>

<body>
    <div style="margin: auto;width: 100%;padding: 0px;">
        <video id="my-player" class="video-js vjs-default-skin" controls preload="auto" poster="poster.jpg"
            data-setup='{"fluid": true}'>
            <source src="http://raspberry.home:8080/hls/stream.m3u8" type="application/x-mpegURL">
            </source>
            <p class="vjs-no-js">
                To view this video please enable JavaScript, and consider upgrading to a
                web browser that
                <a href="https://videojs.com/html5-video-support/" target="_blank">
                    supports HTML5 video
                </a>
            </p>
        </video>
    </div>
    <script>
        var player = videojs('my-player');
    </script>
</body>

</html>
```

## EOF

至此摄像头架设完成。