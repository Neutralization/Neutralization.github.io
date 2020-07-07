---
title: Python 使用 USB 热敏打印机
date: 2020-07-07 21:17:37
s: python-escpos-on-windows
categories:
- 杂记
tags:
- Windows
- python
---

通过使用 python-escpos 库，就可以和常见的USB热敏打印机通信来进行打印，不再赘述具体方法。

但是python-escpos库并不是为Windows写的，实际在使用的时候就会出现许多的问题。

一个常见的错误是

> ValueError: No backend available

根据[Stackoverflow](https://stackoverflow.com/questions/13773132/pyusb-on-windows-no-backend-available)上的解答，应该是缺少dll文件，按解答中提供的链接地址下载安装就行了。

但安装后会出现新的错误：

> NotImplementedError: Operation not supported or unimplemented on this platform

所幸在python-escpos的Github [issue316](https://github.com/python-escpos/python-escpos/issues/316)中有人提供了解决方式。直接修改源代码，删去对应的异常捕捉，就能够正常使用了。

此处留档以备不时之需。