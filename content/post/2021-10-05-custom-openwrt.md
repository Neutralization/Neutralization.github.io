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
    > **sudo** apt full-upgrade -y

4. [准备编译工具](https://openwrt.org/docs/guide-developer/build-system/install-buildsystem#debianubuntu)

    > **sudo** apt install -y ack antlr3 asciidoc autoconf automake \\\
    autopoint binutils bison build-essential bzip2 ccache cmake cpio \\\
    curl device-tree-compiler fastjar flex gawk gettext gcc-multilib \\\
    g++-multilib git gperf haveged help2man intltool libc6-dev-i386 \\\
    libelf-dev libglib2.0-dev libgmp3-dev libltdl-dev libmpc-dev libmpfr-dev \\\
    libncurses5-dev libncursesw5-dev libreadline-dev libssl-dev libtool \\\
    lrzsz mkisofs msmtp nano ninja-build p7zip p7zip-full patch pkgconf \\\
    python2.7 python3 python3-pip qemu-utils rsync scons squashfs-tools \
    subversion swig texinfo uglifyjs upx-ucl unzip vim wget xmlto xxd zlib1g-dev

<!-- more -->

## 克隆仓库

1. 下载 OpenWrt 源码（更改为[LEDE](https://github.com/coolsnowwolf/lede)）

    ```bash
    cd ~
    git clone https://github.com/coolsnowwolf/lede openwrt
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

3. 修改路由IP地址

    ```bash
   > sed -i 's/192.168.1.1/192.168.2.1/g' package/base-files/files/bin/config_generate
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
    Kernel modules  --->
        Cryptographic API modules  --->
            ALL
    LuCI  --->
            Translations  --->
                <*> Simplified Chinese (zh-cn)
            <*> luci-compat
        3. Applications  --->
            <*> luci-app-openclash
            <*> luci-app-turboacc
            <*> luci-app-unblockmusic
            [*] UnblockNeteaseMusic NodeJS Version
            <*> luci-app-wireguard
            <*> luci-app-zerotier
    Utilities  --->
        Editors  --->
            <*> nano-full
    ```

3. 执行编译

    > **cd** ~/openwrt/  
    > **make** -j8 download  
    > **make** -j8

编译生成的镜像文件在 `~/openwrt/bin/targets/rockchip/armv8/` 路径下

## 收尾工作

opkg 安装部分官方包的时候会提示 Kernel 版本不对`kernel is not compatible`，这是因为自己编译的 kernel 指纹和官方不一致造成的，[替换成官方指纹即可](https://github.com/iyuangang/openwrt/issues/8#issuecomment-605431578)，比如目前 kernel 版本是`5.4.143-1-fb881fbbae69f30da18e7c6eb01310c1`

> **sed** -i s/`编译的kernel hash`/fb881fbbae69f30da18e7c6eb01310c1/g /usr/lib/opkg/status

## EOF

编译 run 了一万年，再不写点什么记下来，下次编译根本记不住了.jpg

后来参考学习了 P3TERX 的 [这篇文章](https://p3terx.com/archives/build-openwrt-with-github-actions.html) 利用 Github Action 进行编译，并且按照 [LEDE的这个PR](https://github.com/coolsnowwolf/lede/pull/7796) 打开了缓存工具链，再也不用面对奇怪的环境问题和该参数重编又是四、五个小时的窘境了。
