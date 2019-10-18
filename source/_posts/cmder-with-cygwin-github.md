---
title: 配置Cmder整合Cygwin与Github
s: cmder-with-cygwin-github
date: 2019-10-19 05:08:09
categories:
- 杂记
tags:
- cmder
- bash
- cygwin
---

记录下折腾Cmder的过程。

虽然群友安利了许久的WSL和Powershell，但对于前两者的GUI我实在看不下去。也许他们真的很棒，但好用的工具似乎注定了不能拥有好看的UI。

而我，还是倾向于UI舒服的。毕竟只要它真的不是功能上缺胳膊少腿儿，剩下的事情，大部分都可以克服。

再说了，不多折腾折腾，不多踩几个坑，多Google和Stackoverflow一下，你怎么会知道原来世界上还能有这种问题呢.jpg
<!--more-->
## 初始化

我们可以从[Cmder的主页](https://cmder.net/)下载它的最新版（当前是v1.3.12）。为了避免之后和Github以及Cygwin附带的命令出现不同版本并存的问题，选择`Download Mini`。

下载完毕后解压，先把Cmder添加到右键菜单，方便之后随时调用。在解压目录打开一个管理员权限的Powershell：

> Cmder.exe /REGISTER ALL

接下来先放着Cmder不管，继续装Git for windows。

从[Git for windows的主页](https://gitforwindows.org/)下载最新版，安装基本都是默认选项，需要注意的是：

1. `Adjusting your PATH enviroment`选择`Use Git from the Windows Command Prompt`
2. `Configuring the line ending conversions`选择`Checkout Windows-style, commit Unix-style line endings`
3. `Configuing the terminal emulator to use with Git Bash`选择`Use MinTTY`

额外选项可以默认也可以不选，安装完成后在任意目录右键，应该能看见`Git Bash Here`和`Git GUI Here`的选项。

接着继续，安装Cygwin。

在[Cygwin官网](https://www.cygwin.com/)下载对应版本（64-bit/32-bit），双击执行，默认选项即可。

在`Choose A Download Site`这步，我们可以添加[清华大学的镜像源](https://mirror.tuna.tsinghua.edu.cn/help/cygwin/)`https://mirrors.tuna.tsinghua.edu.cn/cygwin/`来提升软件包的下载速度，然后点击下一步。

在Search中搜索wget，下拉菜单将Skip替换成其中一个版本安装，这是之后安装apt-cyg需要的依赖。另外检查gwak、tar、bzip2这几个包是否安装（不是Skip就对了）。

有需要的话可以自己选择还要安装哪些命令，不过这些都可以之后再改，先接着继续下一步默认安装就可以了。

## 折腾

现在我们有了Cmder+Github+Cygwin，接下来把他们整一起。

git比较方便，打开控制面板，选择`系统和安全`->`系统`->`高级系统设置`->`环境变量`，在`用户变量`里选择`Path`->`编辑`，如果是默认安装的话，应该会有一条`C:\Program Files\Git\usr\bin`的记录，如果没有可以手动添加。

然后打开Cmder，输入git并回车，检查是否能够调用git。

Cygwin按照同样的方式加入系统环境变量，默认路径是`C:\cygwin64\bin`，如果修改了安装路径需要对应修改。

接着安装apt-cyg，有了它就可以像Ubuntu管理软件包一样随意install需要的命令了。

apt-cyg的项目主页是 https://github.com/transcode-open/apt-cyg ，在Release页面下载最新版，解压将`apt-cyg`文件移动到`C:\cygwin64\bin`，打开cygwin终端：

> apt-cyg install nano

测试apt-cyg是否正常工作。

为了解决中文编码问题，在Cygwin终端窗口右键选择`Options`，选择`Text`，更改`locale`为`zh_CN`，`Character set`为`UTF-8`。

然后nano或者vi编辑`~/.bashrc`文件，在文件最后添加：

```~/.bashrc
export LC_ALL=zh_CN.UTF-8
export LC_CTYPE=zh_CN.UTF-8
export LANG=zh_CN.UTF-8
```

为了Cygwin和Cmder整合后，能够识别当前工作目录，通过apt-cyg安装chere命令：

> apt-cyg install chere

然后还是编辑`~/.bashrc`，追加内容：[参考1]

```~/.bashrc
if [ -n "${ConEmuWorkDir}" ]; then
  cd "$ConEmuWorkDir"
fi
```

Cygwin中C盘的路径映射为`/cygdrive/c`，如果觉得太长的话，可以在cygwin终端里修改`/etc/fstab`或者直接修改`C:\cygwin64\etc\fstab`:

```/etc/fstab
# /etc/fstab
#
#    This file is read once by the first process in a Cygwin process tree.
#    To pick up changes, restart all Cygwin processes.  For a description
#    see https://cygwin.com/cygwin-ug-net/using.html#mount-table

# This is default anyway:
# none /cygdrive cygdrive binary,posix=0,user 0 0
none / cygdrive binary 0 0
```

之后C盘映射到`/c`，D盘映射到`/d`，以此类推。

以上设置完可以算是告一段落了，接着把Cygwin整进Cmder里。

打开Cmder，右键选择`Settings`，选择`Startup`->`Tasks`。

点击'+'号添加新的Task，`Task Name`填一个能区分出是Cygwin的，比如`Cygwin::bash`，`Task parameters`填写`/icon C:\cygwin64\Cygwin-Terminal.ico`，在`Commands`中填写`set CHERE_INVOKING=1 & C:\cygwin64\Cygwin.bat -c "/bin/xhere /bin/bash.exe --login -i '%V'"`[参考2]

然后勾选上`Default task for new console`和`Taskbar jump lists`。回到`Startup`，选择`Specified named task`->`Cygwin::bash`。

这样一来Cmder的默认终端就是Cygwin了，git命令和windows本身的支持也没有问题。

## 进一步调整

### `General`->`Fonts`

先解决编码问题，选择`General`->`Fonts`->`Unicode ranges`->`CJK: 2E80-9FC3;AC00-D7A3;F900-FAFF;FE30-FE4F;FF01-FF60;FFE0-FFE6;`->`Apply`。

`Font charset`还是保持ANSI，否则Cmder会报错`Failed to create font`然后fail back回缺省字体。

然后选择`Startup`->`Environment`，添加如下内容：

```
set PATH=%ConEmuBaseDir%\Scripts;%PATH%
set LANG=zh_CN.UTF8
```

接着是字体，中文字体真的太少了，好看的就更少了。（有一说一，我觉得，确实，是这样的）

选了Powerline Fonts：

> git clone https://github.com/powerline/fonts.git

我单独装了`Noto Mono for Powerline`，也可以选择执行`install.ps1`直接安装全部的字体。

在`Main console font`选了`Noto Mono for Powerline`后，中文还是会fail back成宋体，这里勾上`Alternative font`，选择`微软雅黑 Light`，个人感觉不违和，能看。

其他勾选上`Monospace`和`Compress long strings to fit space`。

### `General`->`Size & Pos`

将`Width`改为80%，`Height`改为70%，这样Cmder启动会自动根据显示器大小调整窗口大小。

### `General`->`Background`

设置`Background Image`和`Darkening`可以给终端添加图片背景并调整图片透明度，我就敬谢不敏了。

### `General`->`Confirm`

去除`Confirm creating new console/tab`和`Confirm tab duplicating`的勾选，这两个太烦人了。

### `Features`->`Transparency`

`Alpha transparency`可以调整终端窗口本身的透明度，我这里直接拖到了最右边不透明。屡次截图终端映出了背后的内容总是让我心有余悸。

### `Keys & Macro`->`Paste`

确保两个`Paste mode`都是`Multi lines`，避免行为不一致。

## 其他

* Cmder有自己的`user_alias`("Cmder/config/user_alias.cmd")可以很方便的设定一些常用命令，可以在cmd或者powershell终端里使用，但是不能和Cygwin通用。Cygwin需要Alias的话还是得老老实实编辑`~/.bashrc`。

* Cmder可以作为Sublime Text的终端来使用，Sublime安装Terminal插件，设置终端路径为Cmder安装路径即可。默认呼出终端的快捷键是Ctrl+Shift+T。

* Cygwin有个已知问题，Ctrl+方向键没有绑定操作，需要手动添加，方法是编辑`~/.inputrc`添加两行内容`"\e[1;5C": forward-word`和`"\e[1;5D": backward-word`。[参考3]

* 如果之前已经生成了SSH KEY的话，需要手动复制到`C:\cygwin64\<user name>\home`或者直接制定ssh key才能让git识别到。

## EOF

[参考1]:https://conemu.github.io/en/CygwinStartDir.html "cygwin, mingw, ConEmu and start up directory"

[参考2]:https://github.com/cmderdev/cmder/wiki/Integrating-Cygwin "Integrating Cygwin"

[参考3]:http://trumaze.blogspot.com/2011/04/how-to-configure-cygwin-to-use-ctrl.html "How to configure cygwin to use ctrl + arrow to move cursor forward / backward"

