---
title: "Git 检测到存储库可能不安全的解决方法"
date: 2024-03-12T12:02:31+08:00
categories:
    - 杂记
tags:
    - git
---

因为经常折腾东西导致重新安装系统，也因此经常遇到类似如下问题：

```powershell
git status

fatal: detected dubious ownership in repository at 'D:/Neutralization.github.io'
'D:/Neutralization.github.io/.git' is owned by:
        (inconvertible) (S-1-5-21-111-111-111-1001)
but the current user is:
        WORKGROUP/Neutralized (S-1-5-21-222-222-222-1002)
To add an exception for this directory, call:

        git config --global --add safe.directory D:/Neutralization.github.io
```

网上的建议一般是按照 git 的提示照做添加 `safe.directory` ，或者就是文件夹右键 属性/安全/高级/更改所有者/☑替换子容器和对象的所有者 这样的一套操作。

其实完全可以简化为：

> sudo takeown /R /F .  
> sudo icacls . /reset /q /t /c  

留作记录。