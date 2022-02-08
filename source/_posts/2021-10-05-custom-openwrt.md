---
title: OpenWrt 折腾笔记
date: 2021-10-05 16:58:21
categories:
    - DIY
tags:
    - openwrt
    - wsl
---

## 准备工作

1. 安装 WSL，不再赘述。

2. [移除 WSL 环境变量中包含的 Windows 路径](https://openwrt.org/docs/guide-developer/build-system/wsl#setting_up_path)，为编译做准备。

    打开 WSL 终端，新建`/etc/wsl.conf`文件，输入如下内容并保存。

    ```conf
    [interop]
    appendWindowsPath = false
    ```

    然后执行`wsl --shutdown`，再重新启动 wsl，执行`echo ${PATH}`检查是否含有 Windows 路径。

3. 更新系统

    > **sudo** apt update  
    > **sudo** apt dist-upgrade -y

4. [准备编译工具](https://openwrt.org/docs/guide-developer/build-system/install-buildsystem#debianubuntu)

    > **sudo** apt install build-essential ccache ecj fastjar file g++ gawk \\\
    > gettext git java-propose-classpath libelf-dev libncurses5-dev \\\
    > libncursesw5-dev libssl-dev python python2.7-dev python3 unzip wget \\\
    > python3-distutils python3-setuptools python3-dev rsync subversion \\\
    > swig time xsltproc zlib1g-dev

<!-- more -->

## 克隆仓库

1. 下载 [OpenWrt](https://openwrt.org/docs/guide-developer/build-system/use-buildsystem) 源码

    ```bash
    cd ~
    git clone https://git.openwrt.org/openwrt/openwrt.git openwrt
    cd openwrt
    git pull
    git checkout v21.02.0
    ```

2. 下载 [OpenClash](https://github.com/vernesong/OpenClash) 源码

    ```bash
    cd ~/openwrt/
    mkdir package/luci-app-openclash
    cd package/luci-app-openclash
    git init
    git remote add origin https://github.com/vernesong/OpenClash.git
    git config core.sparsecheckout true
    echo "luci-app-openclash" >> .git/info/sparse-checkout
    git pull --depth 1 origin master
    git branch --set-upstream-to=origin/master master
    ```

3. 下载 [UnblockNeteaseMusic](https://github.com/UnblockNeteaseMusic/luci-app-unblockneteasemusic) 源码

    ```bash
    cd ~/openwrt/package/
    git clone https://github.com/UnblockNeteaseMusic/luci-app-unblockneteasemusic.git
    ```

    ~~UnblockNeteaseMusic 依赖 `libustream-openssl`，但是 OpenWrt v21.02 版本的 NanoPi R2S 架构依赖 `libustream-wolfssl`，直接编译会发生冲突，需要替换 `Makefile` 中的依赖。~~

    仓库已经更新会自动解决依赖问题了。

4. [一些个人配置](https://openwrt.org/docs/guide-developer/uci-defaults#integrating_custom_settings) 参考[The UCI system](https://openwrt.org/docs/guide-user/base-system/uci)

    > **nano** ~/openwrt/package/base-files/files/etc/uci-defaults/99_custom

    ```bash
    uci -q batch << EOI
    set network.lan.ipaddr='192.168.2.1'
    set network.@device[2].macaddr='b8:27:eb:48:f8:30'
    set network.@device[2].ipv6='0'
    del network.wan6
    commit network
    set dhcp.lan.force='1'
    set dhcp.lan.ra_flags='none'
    del dhcp.lan.ra
    del dhcp.lan.ra_slaac
    del dhcp.lan.dhcpv6
    commit dhcp
    set system.@system[0]=system
    set system.@system[0].hostname='WTF'
    set system.@system[0].timezone='CST-8'
    set system.@system[0].zonename='Asia/Shanghai'
    set system.@system[0].ttylogin='0'
    set system.@system[0].log_size='64'
    set system.@system[0].log_proto='udp'
    set system.@system[0].urandom_seed='0'
    commit system
    ```

## 开始编译

1. 更新 feeds

    > **cd** ~/openwrt/  
    > ./scripts/feeds update -a  
    > ./scripts/feeds install -a

2. 选择编译内容

    > **cd** ~/openwrt/  
    > **make defconfig**  
    > **make menuconfig**  
    > ./scripts/diffconfig.sh > diffconfig

    diffconfig 后期可用于更新镜像时导入

    > **cp** diffconfig ~/openwrt/.config  
    > **make defconfig**  
    > **make menuconfig**

    OpenClash 依赖 `dnsmasq-full`，要取消 `dnsmasq` 后勾选 `dnsmasq-full`

    ```menuconfig
    Target System  --->
        (X) Rockchip
    Subtarget  --->
        (X) RK33xx boards (64 bit)
    Target Profile  --->
        (X) FriendlyARM NanoPi R2S
    Target Images  --->
        [*] ext4
        [*] squashfs
        [*] GZip images
        (128) Kernel partition size (in MB)
        (2048) Root filesystem partition size (in MB)
    Base system  --->
        < > dnsmasq
        <*> dnsmasq-full
    Languages  --->
        Python  --->
            <*> python-pip-conf
            <*> python3
            <*> python3-aiohttp
            <*> python3-lxml
            <*> python3-pip
            <*> python3-pyotp
            <*> python3-requests
    LuCI  --->
        1. Collections  --->
            <*> luci-ssl
        2. Modules  --->
            [*] Minify Lua sources
            Translations  --->
                <*> Chinese Simplified (zh_Hans)
            <*> luci-compat
        3. Applications  --->
            <*> luci-app-openclash
            <*> luci-app-unblockneteasemusic
            <*> luci-app-wireguard
    Multimedia  --->
        <*> ffmpeg
    Network  --->
        Version Control Systems  --->
            <*> git-http
    Utilities  --->
        Compression  --->
            <*> unzip
        Editors  --->
            <*> nano
        <*> gawk
        <*> grep
        <*> sed
        <*> whereis
        <*> which
    ```

3. 执行编译

    > **cd** ~/openwrt/  
    > **make** -j8 download  
    > **make** -j8

编译生成的镜像文件路径为 `~/openwrt/bin/targets/rockchip/armv8/openwrt-rockchip-armv8-friendlyarm_nanopi-r2s-squashfs-sysupgrade.img.gz`

## 收尾工作

opkg 安装部分官方包的时候会提示 Kernel 版本不对`kernel is not compatible`，这是因为自己编译的 kernel 指纹和官方不一致造成的，[替换成官方指纹即可](https://github.com/iyuangang/openwrt/issues/8#issuecomment-605431578)，目前 kernel 版本是`5.4.143-1-fb881fbbae69f30da18e7c6eb01310c1`

> **sed** -i s/`编译的kernel hash`/fb881fbbae69f30da18e7c6eb01310c1/g /usr/lib/opkg/status

编译 python 的主要目的还是为了利用 crontab 跑一些简单的脚本，比如[原神签到小助手](https://www.yindan.me/tutorial/genshin-impact-helper.html)

编译 ffmpeg 则是为了跑 youtube 下载工具 [yt-dlp](https://github.com/yt-dlp/yt-dlp)，结合 [Rclone](https://rclone.org/) 定时存档订阅的 youtube 频道更新。

## EOF

编译 run 了一万年，再不写点什么记下来，下次编译根本记不住了.jpg

后来参考学习了 [使用 GitHub Actions 云编译 OpenWrt](https://p3terx.com/archives/build-openwrt-with-github-actions.html) 利用 Github Action 进行编译，再也不用面对奇怪的环境问题了。
