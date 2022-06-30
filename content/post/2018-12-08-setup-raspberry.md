---
title: 从零开始 Raspberry Pi
date: 2018-12-08 00:20:00
categories:
    - 杂记
tags:
    - 树莓派
    - debian
---

为了让吃灰的树莓派重新开始工作，这回不瞎折腾了，直接用官方系统。

<!-- more -->

## 安装

按照[官方教程](https://www.raspberrypi.org/downloads/noobs/)采用 NOOBS 安装，当然也可以选择用[镜像安装](https://www.raspberrypi.org/downloads/raspbian/)，不论哪一种，选择 ↓

`Raspbian Stretch with desktop and recommended software`

安装完成后，首次进入图形界面，选择首选项中的 `Raspberry Pi Configuration` ，在 `Interfaces` 书签页启用 `SSH`, `VNC`，之后就可以不用另接键鼠了。

记得在 `Localisation` 书签页设定好 `WiFi Country`。

## 连接 WiFi

因为公司 WiFi 使用了 WPA-EAP 协议，树莓派不显示热点，让我一度以为树莓派不能连接此类 WiFi。后来在[这篇文章](https://eparon.me/2016/09/09/rpi3-enterprise-wifi.html)的帮助下，通过修改 `wpa_supplicant.conf` 连接成功，但是 WiFi 列表中对应 SSID 选项为灰色不可点击，迷。

> 修改 /etc/wpa_supplicant/wpa_supplicant.conf

```textile
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=CN

network={
    ssid="OfficeWiFiSSID"
    key_mgmt=WPA-EAP
    identity="username"
    password="password"
}
```

## 软件更新

更换软件源，这个不多说了，国内推荐 [中国科学技术大学](https://mirrors.ustc.edu.cn/) 、 [清华大学](https://mirrors.tuna.tsinghua.edu.cn/) 、 [网易开源镜像站](https://mirrors.163.com/) 的镜像。

在 apt 更新前，先删除不必要的软件。反正周也是要卸载的，没必要让他们下载安装更新。

```shell
sudo apt-get remove --purge python3-pygame python-pygame scratch nuscratch claws-mail smartsim sonic-pi minecraft-pi python-minecraftpi wolfram-engine scratch bluej  nodered greenfoot scratch2 libreoffice libreoffice-base libreoffice-core chromium-browser  thonny python3-thonny python-sense-emu python3-sense-emu python-sense-emu-doc sense-emu-tools

sudo apt-get autoremove
sudo apt-get update
sudo apt-get dist-upgrade
```

卸载这部分应用后，会有一部分文件残留，通过 VNC 远程进去打开以下三个目录

> /usr/share/applications/
>
> /usr/share/raspi-ui-overrides/applications/
>
> /usr/share/mimelnk/application/

在三个目录里找到图标显示失效的文件，删掉它们。

## 自动获取树莓派 IP

接下来遇到的问题是，可能是因为联网设备实在太多，公司 WiFi 分配的 IP 地址短期内是和 MAC 绑定不变的，但假如长时间未连接，重新连接就会发分配不同的 IP。

导致 SSH 记住 IP 并没有什么用，只能每次用 VNC 或者 nmap 扫描网段找到树莓派的地址。

Google 后参考 [这篇文章](https://ariandy1.wordpress.com/2014/04/08/linux-send-email-when-ip-address-changes/) 设置，让树莓派每次 IP 变动后自动发送邮件通知自己。

> sudo apt-get install ssmtp mailutils
>
> nano /etc/ssmtp/ssmtp.conf

```textile
root=youremail@gmail.com
mailhub=smtp.gmail.com:587
AuthUser=youremail@gmail.com
AuthPass=yourpassword
UseTLS=YES
UseSTARTTLS=YES
AuthMethod=LOGIN
```

> nano /home/pi/checkip.sh

```shell
#!/bin/bash

MYIP=`ifconfig | grep -Eo 'inet (addr:)?([0-9]*\.){3}[0-9]*' | grep -Eo '([0-9]*\.){3}[0-9]*' | grep -v '127.0.0.1'`;
TIME=`date`;

LASTIPFILE='/home/pi/.last_ip_addr';
LASTIP=`cat ${LASTIPFILE}`;

if [[ ${MYIP} != ${LASTIP} ]]
then
        if [[ ${MYIP} = '' ]]
        then
            echo "LOST"
        else
            echo "Sending E-mail.."
            echo -e "Hello\n\nTimestamp = ${TIME}\nIP = ${MYIP}\n\nBye" | \
                    /usr/bin/mail -s "[INFO] Raspberrypi New IP" youremail@gmail.com;
            echo ${MYIP} > ${LASTIPFILE};
        fi
else
        echo "No IP change!"
fi
```

> chmod +x checkip.sh
>
> crontab -e

```textile
*/30 * * * * /home/pi/checkip.sh
```

## EOF

于是树莓派终于可以一边通电一边吃灰了。