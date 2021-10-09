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

2. [移除 WSL 环境变量中包含的 Windows 路径](https://openwrt.org/docs/guide-developer/build-system/wsl)，为编译做准备。

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

3. 下载 [SmartDNS](https://github.com/pymumu/luci-app-smartdns) 源码

    ```bash
    cd ~/openwrt/
    WORKINGDIR="`pwd`/feeds/packages/net/smartdns"
    mkdir $WORKINGDIR -p
    rm $WORKINGDIR/* -fr
    wget https://github.com/pymumu/openwrt-smartdns/archive/master.zip -O $WORKINGDIR/master.zip
    unzip $WORKINGDIR/master.zip -d $WORKINGDIR
    mv $WORKINGDIR/openwrt-smartdns-master/* $WORKINGDIR/
    rmdir $WORKINGDIR/openwrt-smartdns-master
    rm $WORKINGDIR/master.zip

    LUCIBRANCH="master"
    WORKINGDIR="`pwd`/feeds/luci/applications/luci-app-smartdns"
    mkdir $WORKINGDIR -p
    rm $WORKINGDIR/* -fr
    wget https://github.com/pymumu/luci-app-smartdns/archive/${LUCIBRANCH}.zip -O $WORKINGDIR/${LUCIBRANCH}.zip
    unzip $WORKINGDIR/${LUCIBRANCH}.zip -d $WORKINGDIR
    mv $WORKINGDIR/luci-app-smartdns-${LUCIBRANCH}/* $WORKINGDIR/
    rmdir $WORKINGDIR/luci-app-smartdns-${LUCIBRANCH}
    rm $WORKINGDIR/${LUCIBRANCH}.zip
    ```

4. 下载 [UnblockNeteaseMusic](https://github.com/UnblockNeteaseMusic/luci-app-unblockneteasemusic) 源码

    ```bash
    cd ~/openwrt/package/
    git clone https://github.com/UnblockNeteaseMusic/luci-app-unblockneteasemusic.git
    ```

    UnblockNeteaseMusic 依赖 `libustream-openssl`，但是 OpenWrt v21.02 版本的 NanoPi R2S 架构依赖 `libustream-wolfssl`，直接编译会发生冲突，需要替换 `Makefile` 中的依赖。

    > **sed** -i s/libustream-openssl/libustream-wolfssl/g luci-app-unblockneteasemusic/Makefile

5. 一些个人配置, 利用[UCI](https://openwrt.org/docs/guide-user/base-system/uci)

    > **nano** ~/openwrt/package/base-files/files/etc/uci-defaults/99_custom

    ```bash
    set network.lan.ipaddr='192.168.2.1'
    set network.@device[2].macaddr='b8:27:eb:48:f8:30'
    set network.@device[2].ipv6='0'
    commit network
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

    OpenClash 依赖 `dnsmasq-full`，要取消 `dnsmasq` 后勾选 `dnsmasq-full`

    > **cd** ~/openwrt/  
    > **make defconfig**  
    > **make menuconfig**  
    > ./scripts/diffconfig.sh > diffconfig

    diffconfig 后期可用于更新镜像时导入

    > **cp** diffconfig ~/openwrt/.config  
    > **make defconfig**  
    > **make menuconfig**

    ```conf
    CONFIG_TARGET_rockchip=y
    CONFIG_TARGET_rockchip_armv8=y
    CONFIG_TARGET_rockchip_armv8_DEVICE_friendlyarm_nanopi-r2s=y
    CONFIG_LIBCURL_COOKIES=y
    CONFIG_LIBCURL_FILE=y
    CONFIG_LIBCURL_FTP=y
    CONFIG_LIBCURL_HTTP=y
    CONFIG_LIBCURL_NGHTTP2=y
    CONFIG_LIBCURL_NO_SMB="!"
    CONFIG_LIBCURL_PROXY=y
    CONFIG_LIBCURL_WOLFSSL=y
    CONFIG_LUCI_LANG_zh_Hans=y
    CONFIG_NODEJS_ICU_SMALL=y
    CONFIG_OPENSSL_ENGINE=y
    CONFIG_OPENSSL_WITH_ASM=y
    CONFIG_OPENSSL_WITH_CHACHA_POLY1305=y
    CONFIG_OPENSSL_WITH_CMS=y
    CONFIG_OPENSSL_WITH_DEPRECATED=y
    CONFIG_OPENSSL_WITH_ERROR_MESSAGES=y
    CONFIG_OPENSSL_WITH_PSK=y
    CONFIG_OPENSSL_WITH_SRP=y
    CONFIG_OPENSSL_WITH_TLS13=y
    CONFIG_PACKAGE_bash=y
    CONFIG_PACKAGE_ca-certificates=y
    CONFIG_PACKAGE_cgi-io=y
    CONFIG_PACKAGE_coreutils=y
    CONFIG_PACKAGE_coreutils-nohup=y
    CONFIG_PACKAGE_curl=y
    # CONFIG_PACKAGE_dnsmasq is not set
    CONFIG_PACKAGE_dnsmasq-full=y
    CONFIG_PACKAGE_dnsmasq_full_auth=y
    CONFIG_PACKAGE_dnsmasq_full_conntrack=y
    CONFIG_PACKAGE_dnsmasq_full_dhcp=y
    CONFIG_PACKAGE_dnsmasq_full_dhcpv6=y
    CONFIG_PACKAGE_dnsmasq_full_dnssec=y
    CONFIG_PACKAGE_dnsmasq_full_ipset=y
    CONFIG_PACKAGE_dnsmasq_full_noid=y
    CONFIG_PACKAGE_dnsmasq_full_tftp=y
    CONFIG_PACKAGE_gawk=y
    CONFIG_PACKAGE_ip-full=y
    CONFIG_PACKAGE_ipset=y
    CONFIG_PACKAGE_iptables-mod-extra=y
    CONFIG_PACKAGE_iptables-mod-tproxy=y
    CONFIG_PACKAGE_kmod-crypto-hash=y
    CONFIG_PACKAGE_kmod-crypto-kpp=y
    CONFIG_PACKAGE_kmod-crypto-lib-blake2s=y
    CONFIG_PACKAGE_kmod-crypto-lib-chacha20=y
    CONFIG_PACKAGE_kmod-crypto-lib-chacha20poly1305=y
    CONFIG_PACKAGE_kmod-crypto-lib-curve25519=y
    CONFIG_PACKAGE_kmod-crypto-lib-poly1305=y
    CONFIG_PACKAGE_kmod-ipt-extra=y
    CONFIG_PACKAGE_kmod-ipt-ipset=y
    CONFIG_PACKAGE_kmod-ipt-tproxy=y
    CONFIG_PACKAGE_kmod-nf-conntrack-netlink=y
    CONFIG_PACKAGE_kmod-nfnetlink=y
    CONFIG_PACKAGE_kmod-tun=y
    CONFIG_PACKAGE_kmod-udptunnel4=y
    CONFIG_PACKAGE_kmod-udptunnel6=y
    CONFIG_PACKAGE_kmod-wireguard=y
    CONFIG_PACKAGE_libatomic=y
    CONFIG_PACKAGE_libbpf=y
    CONFIG_PACKAGE_libbz2=y
    CONFIG_PACKAGE_libcap=y
    CONFIG_PACKAGE_libcap-bin=y
    CONFIG_PACKAGE_libcap-bin-capsh-shell="/bin/sh"
    CONFIG_PACKAGE_libcares=y
    CONFIG_PACKAGE_libcurl=y
    CONFIG_PACKAGE_libdb47=y
    CONFIG_PACKAGE_libelf=y
    CONFIG_PACKAGE_libffi=y
    CONFIG_PACKAGE_libgdbm=y
    CONFIG_PACKAGE_libgmp=y
    CONFIG_PACKAGE_libipset=y
    CONFIG_PACKAGE_libiwinfo=y
    CONFIG_PACKAGE_libiwinfo-data=y
    CONFIG_PACKAGE_libiwinfo-lua=y
    CONFIG_PACKAGE_liblua=y
    CONFIG_PACKAGE_liblucihttp=y
    CONFIG_PACKAGE_liblucihttp-lua=y
    CONFIG_PACKAGE_liblzma=y
    CONFIG_PACKAGE_libmnl=y
    CONFIG_PACKAGE_libncurses=y
    CONFIG_PACKAGE_libnetfilter-conntrack=y
    CONFIG_PACKAGE_libnettle=y
    CONFIG_PACKAGE_libnfnetlink=y
    CONFIG_PACKAGE_libnghttp2=y
    CONFIG_PACKAGE_libopenssl=y
    CONFIG_PACKAGE_libpcre=y
    CONFIG_PACKAGE_libpython3=y
    CONFIG_PACKAGE_libreadline=y
    CONFIG_PACKAGE_libruby=y
    CONFIG_PACKAGE_libsqlite3=y
    CONFIG_PACKAGE_libstdcpp=y
    CONFIG_PACKAGE_libubus-lua=y
    CONFIG_PACKAGE_libuv=y
    CONFIG_PACKAGE_libyaml=y
    CONFIG_PACKAGE_lua=y
    CONFIG_PACKAGE_luci=y
    CONFIG_PACKAGE_luci-app-firewall=y
    CONFIG_PACKAGE_luci-app-openclash=y
    CONFIG_PACKAGE_luci-app-opkg=y
    CONFIG_PACKAGE_luci-app-smartdns=y
    CONFIG_PACKAGE_luci-app-unblockneteasemusic=y
    CONFIG_PACKAGE_luci-app-wireguard=y
    CONFIG_PACKAGE_luci-base=y
    CONFIG_PACKAGE_luci-compat=y
    CONFIG_PACKAGE_luci-i18n-base-zh-cn=y
    CONFIG_PACKAGE_luci-i18n-firewall-zh-cn=y
    CONFIG_PACKAGE_luci-i18n-opkg-zh-cn=y
    CONFIG_PACKAGE_luci-i18n-wireguard-zh-cn=y
    CONFIG_PACKAGE_luci-lib-base=y
    CONFIG_PACKAGE_luci-lib-ip=y
    CONFIG_PACKAGE_luci-lib-jsonc=y
    CONFIG_PACKAGE_luci-lib-nixio=y
    CONFIG_PACKAGE_luci-mod-admin-full=y
    CONFIG_PACKAGE_luci-mod-network=y
    CONFIG_PACKAGE_luci-mod-status=y
    CONFIG_PACKAGE_luci-mod-system=y
    CONFIG_PACKAGE_luci-proto-ipv6=y
    CONFIG_PACKAGE_luci-proto-ppp=y
    CONFIG_PACKAGE_luci-proto-wireguard=y
    CONFIG_PACKAGE_luci-theme-bootstrap=y
    CONFIG_PACKAGE_nano=y
    CONFIG_PACKAGE_node=y
    CONFIG_PACKAGE_python-pip-conf=y
    CONFIG_PACKAGE_python3=y
    CONFIG_PACKAGE_python3-asyncio=y
    CONFIG_PACKAGE_python3-base=y
    CONFIG_PACKAGE_python3-cgi=y
    CONFIG_PACKAGE_python3-cgitb=y
    CONFIG_PACKAGE_python3-codecs=y
    CONFIG_PACKAGE_python3-ctypes=y
    CONFIG_PACKAGE_python3-dbm=y
    CONFIG_PACKAGE_python3-decimal=y
    CONFIG_PACKAGE_python3-distutils=y
    CONFIG_PACKAGE_python3-email=y
    CONFIG_PACKAGE_python3-gdbm=y
    CONFIG_PACKAGE_python3-light=y
    CONFIG_PACKAGE_python3-logging=y
    CONFIG_PACKAGE_python3-lzma=y
    CONFIG_PACKAGE_python3-multiprocessing=y
    CONFIG_PACKAGE_python3-ncurses=y
    CONFIG_PACKAGE_python3-openssl=y
    CONFIG_PACKAGE_python3-pip=y
    CONFIG_PACKAGE_python3-pkg-resources=y
    CONFIG_PACKAGE_python3-pydoc=y
    CONFIG_PACKAGE_python3-readline=y
    CONFIG_PACKAGE_python3-setuptools=y
    CONFIG_PACKAGE_python3-sqlite3=y
    CONFIG_PACKAGE_python3-unittest=y
    CONFIG_PACKAGE_python3-urllib=y
    CONFIG_PACKAGE_python3-xml=y
    CONFIG_PACKAGE_rpcd=y
    CONFIG_PACKAGE_rpcd-mod-file=y
    CONFIG_PACKAGE_rpcd-mod-iwinfo=y
    CONFIG_PACKAGE_rpcd-mod-luci=y
    CONFIG_PACKAGE_rpcd-mod-rrdns=y
    CONFIG_PACKAGE_ruby=y
    CONFIG_PACKAGE_ruby-bigdecimal=y
    CONFIG_PACKAGE_ruby-date=y
    CONFIG_PACKAGE_ruby-dbm=y
    CONFIG_PACKAGE_ruby-digest=y
    CONFIG_PACKAGE_ruby-enc=y
    CONFIG_PACKAGE_ruby-forwardable=y
    CONFIG_PACKAGE_ruby-pstore=y
    CONFIG_PACKAGE_ruby-psych=y
    CONFIG_PACKAGE_ruby-stringio=y
    CONFIG_PACKAGE_ruby-strscan=y
    CONFIG_PACKAGE_ruby-yaml=y
    CONFIG_PACKAGE_sed=y
    CONFIG_PACKAGE_smartdns=y
    CONFIG_PACKAGE_terminfo=y
    CONFIG_PACKAGE_uhttpd=y
    CONFIG_PACKAGE_uhttpd-mod-ubus=y
    CONFIG_PACKAGE_wget-ssl=y
    CONFIG_PACKAGE_wireguard-tools=y
    CONFIG_PACKAGE_zlib=y
    CONFIG_SQLITE3_DYNAMIC_EXTENSIONS=y
    CONFIG_SQLITE3_FTS3=y
    CONFIG_SQLITE3_FTS4=y
    CONFIG_SQLITE3_FTS5=y
    CONFIG_SQLITE3_JSON1=y
    CONFIG_SQLITE3_RTREE=y
    CONFIG_TARGET_KERNEL_PARTSIZE=64
    CONFIG_TARGET_ROOTFS_PARTSIZE=1024
    ```

3. 执行编译

    > cd ~/openwrt/  
    > make -j8 download  
    > make -j8

编译生成的镜像文件路径为 `~/openwrt/bin/targets/rockchip/armv8/openwrt-rockchip-armv8-friendlyarm_nanopi-r2s-squashfs-sysupgrade.img.gz`

## 收尾工作

opkg 安装部分官方包的时候会提示 Kernel 版本不对`kernel is not compatible`，这是因为自己编译的 kernel 指纹和官方不一致造成的，[替换成官方指纹即可](https://github.com/iyuangang/openwrt/issues/8#issuecomment-605431578)，目前 kernel 版本是`5.4.143-1-fb881fbbae69f30da18e7c6eb01310c1`

> **sed** -i s/`编译的kernel hash`/fb881fbbae69f30da18e7c6eb01310c1/g /usr/lib/opkg/status

编译 python 的主要目的还是为了利用 crontab 跑一些简单的脚本，比如[原神签到小助手](https://www.yindan.me/tutorial/genshin-impact-helper.html)

> **mkdir** /usr/share/genshin/  
> **ln** /usr/lib/python3.9/site-packages/genshinhelper/config/config.json /usr/share/genshin/config.json  
> **nano** /usr/share/genshin/genshin.sh

```bash
/usr/bin/python -m genshinhelper
```

> **crontab** -e

```conf
5 0 * * * /usr/share/genshin/genshin.sh
```

## EOF

编译 run 了一万年，再不写点什么记下来，下次编译根本记不住了.jpg
