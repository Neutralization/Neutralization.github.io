---
title: 没有 KMSCON 的第一天，想他
date: 2019-01-10 00:24:24
categories:
    - 杂记
tags:
    - 树莓派
    - crontab
---

因为树莓派用了官方系统，原本在 Arch 上利用[KMSCON](https://wiki.archlinux.org/index.php/KMSCON)来回显中文字符的方式不可行了，沉痛悼念。

不会编译是罪魁祸首，而且包依赖关系看得头大，实在不想自己解决。

<!-- more -->

## 前情提要

因为 tty 设备不支持 CJK 字符，原本 crontab 任务只要重定向输出到`KMSCON`模拟的 pts 设备上就能正常显示中文，现在不得不回归起点。

选择只有两种：

1. 研究怎么在 debian 上把`KMSCON`编译过去
2. 研究怎么在图形界面输出工作用脚本的日志

然后`KMSCON`的编译依赖实在有点多，在 debian 内的包名又都不一样。本来就只会按照教程 `./configure`、`make`、`make install`三步走的我实在是应付不来。

结果只能当`KMSCON`从来不存在，在图形界面寻找解决办法。

## 变通法

搜索之后发现的确有从终端弹图形界面的方式，只需要指定显示设备就行了：

> DISPLAY=:0.0 command

因为我的目的只是需要一个终端窗口，所以：

```bash
> crontab -e
---
@reboot sleep 30 && DISPLAY=:0.0 /usr/bin/lxterminal
```

这样便会在系统启动之后自动启动一个终端窗口，分配一个 pts 的设备，因为是第一个终端窗口所以一定是 pts/0

再把剩下的定时脚本输出全部重定向到 `/dev/pts/0` 就行了

## EOF

感觉还是会有回到 Arch 的一天，Arch 真香。
