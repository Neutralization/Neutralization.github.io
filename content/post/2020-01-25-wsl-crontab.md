---
title: Windows使用WSL配置crontab定时任务
date: 2020-01-25 02:01:30
categories:
    - 杂记
tags:
    - crontab
    - wsl
---

有些 python 脚本任务对于树莓派来说还是负担太重了。毕竟性能上限摆在那里，也不能靠一个小小的 pi 承担太多。

借此机会正好尝试一下 WSL(Windows Subsystem for Linux).

<!-- more -->

## 安装 WSL

安装 WSL 并没有想象中的复杂。进入控制面板->程序和功能->启用或关闭 Windows 功能，勾选`适用于Linux的Windows子系统`，等待安装完毕，然后重启系统。

![适用于Linux的Windows子系统](/2020-01-25-wsl-crontab/Snipaste_2020-01-25_19-14-26.png)

重启完毕，开始菜单选择 MicrosoftStore，搜索 linux 就能够找到可供使用的 linux 发行版，我这里选择了 Ubuntu18.04LTS。

![MicrosoftStore](/2020-01-25-wsl-crontab/Snipaste_2020-01-25_02-08-03.png)

应用安装完毕后，启动应用会要求设定基础的用户名和密码，接着，一个可供使用的 WSL 就准备就绪了。

## 确保 crontab 服务

由于 WSL 并不会默认启动，而且当运行中的 WSL 的最后一个 bash 关闭后，就相当于关闭了整个子系统，那我们的 crontab 也就直接去世了。

为了保持 crontab 常驻，需要一些措施。这里参考了[Schedule Tasks Using Crontab on Windows 10 with WSL](https://blog.snowme34.com/post/schedule-tasks-using-crontab-on-windows-10-with-wsl/index.html)的方法，建立一个 crontab 服务的 Windows 启动项，每次系统启动后在后台常驻。

首先进入 WSL，修改 sudoer，避免每次启动 crontab 服务时需要输入密码确认：

> sudo visudo

在文件末尾加入

> %sudo ALL=NOPASSWD: /etc/init.d/cron start

然后保存文件。回到 Windows，Win+R 打开运行，输入`shell:startup`打开启动文件夹，右键新建快捷方式，目标输入：

> C:\Windows\System32\wsl.exe sudo /etc/init.d/cron start

这样我们就能够保证 crontab 在每次 Windows 启动后常驻后台了。一切正常的话重启 Windows，在任务管理器能看到一个名称为 cron 的进程。

![任务管理器](/2020-01-25-wsl-crontab/Snipaste_2020-01-25_19-16-06.png)

## 使用 crontab

剩下的操作和在 linux 里使用 crontab 没有区别，唯一需要注意的是文件的路径。

在 WSL 中，Windows 的 C 盘被挂载在`/mnt/c`目录下，D\E\F 以此类推，需要注意对应修改。

## EOF
