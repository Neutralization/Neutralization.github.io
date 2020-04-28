---
title: Windows使用WSL配置crontab定时任务
s: wsl-crontab
date: 2020-01-25 02:01:30
categories:
- 杂记
tags:
- WSL
- crontab
---

有些python脚本任务对于树莓派来说还是负担太重了。毕竟性能上限摆在那里，也不能靠一个小小的pi承担太多。

借此机会正好尝试一下WSL(Windows Subsystem for Linux).
<!-- more -->
## 安装WSL

安装WSL并没有想象中的复杂。进入控制面板->程序和功能->启用或关闭Windows功能，勾选`适用于Linux的Windows子系统`，等待安装完毕，然后重启系统。

<center>{% asset_img "Snipaste_2020-01-25_19-14-26.png" "适用于Linux的Windows子系统" %}</center>

重启完毕，开始菜单选择MicrosoftStore，搜索linux就能够找到可供使用的linux发行版，我这里选择了Ubuntu18.04LTS。

<center>{% asset_img "Snipaste_2020-01-25_02-08-03.png" "MicrosoftStore" %}</center>

应用安装完毕后，启动应用会要求设定基础的用户名和密码，接着，一个可供使用的WSL就准备就绪了。

## 确保crontab服务

由于WSL并不会默认启动，而且当运行中的WSL的最后一个bash关闭后，就相当于关闭了整个子系统，那我们的crontab也就直接去世了。

为了保持crontab常驻，需要一些措施。这里参考了[Schedule Tasks Using Crontab on Windows 10 with WSL](https://blog.snowme34.com/post/schedule-tasks-using-crontab-on-windows-10-with-wsl/index.html)的方法，建立一个crontab服务的Windows启动项，每次系统启动后在后台常驻。

首先进入WSL，修改sudoer，避免每次启动crontab服务时需要输入密码确认：

> sudo visudo

在文件末尾加入

> %sudo ALL=NOPASSWD: /etc/init.d/cron start

然后保存文件。回到Windows，Win+R打开运行，输入`shell:startup`打开启动文件夹，右键新建快捷方式，目标输入：

> C:\Windows\System32\wsl.exe sudo /etc/init.d/cron start

这样我们就能够保证crontab在每次Windows启动后常驻后台了。一切正常的话重启Windows，在任务管理器能看到一个名称为cron的进程。

<center>{% asset_img "Snipaste_2020-01-25_19-16-06.png" "任务管理器" %}</center>

## 使用crontab

剩下的操作和在linux里使用crontab没有区别，唯一需要注意的是文件的路径。

在WSL中，Windows的C盘被挂载在`/mnt/c`目录下，D\E\F以此类推，需要注意对应修改。

## EOF
