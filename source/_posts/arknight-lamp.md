---
title: 试做了一个明日方舟的小夜灯
date: 2021-01-12 00:32:09
s: arknight-lamp
categories:
- DIY
tags:
- 3D print
- openscad
---

之前也有在尝试用3D打印做奇怪的东西，建模一直在用的是AutoDesk家的线上工具[Tinkercad](https://www.tinkercad.com/)，属于那种拖拖拽拽就能拼出差不多模型的工具，还挺好用。

这次尝试做小夜灯，原本是打算做成游戏关卡结算时的三星标志，后来想了想，为了更显而易见的突出方舟元素，用了游戏里医疗干员的标志。主要因为医疗干员的标志几何对称性很好，估摸着可以偷懒。

软件用了同事推荐的[OpenSCAD](https://www.openscad.org/)，通过代码的方式来进行建模。我基本上是一边查文档一边写，很容易就上手找到需要的功能写出来了。

除了没有代码自动格式化这一点略有不习惯，代码逻辑支持实体的交并补操作，渲染后可以意见导出stl格式文档进行后续打印工作，实在是方便的很。
<!-- more -->
放两张成果图，使用的是创想三维的Ender-3打印机。冬天材料热涨冷缩比较严重结果留下了两道印子，有点可惜。

电池盒是圣诞节的时候那种细绳装饰灯的电池，考虑到亮度和分散光源，接了一个COB灯板。

<center>{% asset_img "battery.jpg" "lamp" %}</center>

<center>{% asset_img "lamp.jpg" "lamp" %}</center>

工程文件放在[github](https://github.com/Neutralization/Arknights-lamp)上了。