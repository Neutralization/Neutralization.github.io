---
title: 升级3D打印机固件及远程打印方案
date: 2021-08-28 14:13:33
categories:
    - DIY
tags:
    - 3D print
    - Arduino
    - 树莓派
---

Ender 3 不愧是创想三维家的经典款 3D 打印机，一直没关注产品消息，不知道什么时候已经出了 V2 款了。

买当然是不会买的，原来的版本还是挺好用，购买的时候正好京东打折，当时 1k 出头的价格还是很香的。

买来时间也有点久了，考虑升级一下固件，但是官方并没有提供这一款的后续固件升级，也没有相关教程。搜索了一下发现还需要自行刷入 bootloader 后才能刷写固件，看来是当时就没有考虑用户侧的后续升级。

升级固件的想法主要是因为看到了 OctoPrint，一个利用树莓派控制和实时监控 3D 打印机的项目。

## 刷写 bootloader

主要是跟着[Bootloader Flashing Guide](https://www.th3dstudio.com/hc/guides/bootloader/bootloader-flashing-guide-cr-10-ender-2-3-5-wanhao-i3-anet-1284p-boards/)这篇文章中的指导。因为主板芯片是 1284P，和其他打印机的刷写存在区别，建议直接下载链接中的刷写包来操作。

概括的来讲，需要 Arduino 开发板作为中介，与打印机主板的刷写接口连接，再通过 Arduino IDE 将 bootloader 引导程序由 Arduino 刷写到打印机主板芯片上。

找同事借了一块 Arduino MEGA 2560 和几根杜邦线，因为 Arduino 的不同版本引脚定义几乎没有区别，也不是一定要用 Arduino Uno。

完成这一步后，打印机有了 bootloader，就可以方便的进行固件升级操作了。

## 编译固件并上传至打印机

[TH3D](https://www.th3dstudio.com/hc/downloads/unified-2-firmware/creality/creality-ender-3-firmware-melzi-board/) 给出了 Ender 3 的完整固件升级流程。

真的是个不错的网站，除了 Creality 还有其他很多常见 3D 打印机厂家不同型号的操作教程。根据教程描述，安装 VSCode，编译固件上传即可，实际操作中我也完全没有遇到阻碍，非常顺利的编译上传完成了，见到了启动画面的 TH3D 标志 logo。除了界面是全英文，但这也并不影响机上操作。

## OctoPrint 远程打印

又是一个利用树莓派的项目，于是我的树莓派又经历了一次 SD 卡格式化重写映像。

同样也是完全跟随[教程](https://howchoo.com/g/y2rhnzm3odz/control-your-3d-printer-with-octoprint-and-raspberry-pi)操作，在树莓派上使用 OctoPrint 映像，连接 WiFi，USB 连接打印机。

教程中还提供了一套用于固定树莓派摄像机的 3D 打印文件，可以固定在打印机 X 轴上用来监测打印状态。也可以选择使用普通的 USB 摄像头，但是架设摄像头的方式就要靠自己另外实现了。

正好很早以前买过一个树莓派专用摄像头，这就给它用上了。

## Finish

一切设置好，https 访问树莓派 IP 或者 https://octopi.local/ ，如果摄像头正常的话，就能看到打印状态了。

界面里可以一览无余的看到打印机当前的喷嘴和热床温度，打印状态，Gcode 图示等等。也可以直接通过网页上串 Gcode 文件直接打印，由于是通过 USB 连接到打印机，免去了反复插拔 SD 卡传输文件的步骤，真的是方便了很多。

OctoPrint 还提供视频录制，开启后可以录制打印过程中的摄像头画面，生成延时视频。

<center>{% asset_img "Snipaste_2021-08-28_14-55-44.png" "Timelapse" %}</center>
