---
title: "OpenWrt 使用 WireGuard 组网及局域网文件共享设置"
date: 2023-12-12T15:00:29+08:00
categories:
    - 杂记
tags:
    - openwrt
    - wireguard
    - dns
---

一直以来都是依靠群友/同事的帮助来配置当甩手掌柜，最近路由升级 OpenClash 又炸了，加上之前神秘的 DNS 转发失败的问题，决定从零开始自己重新配置一遍软路由，实践出真知。

## 前期准备

硬件上用的是两台 FriendlyARM NanoPi R2S，闲鱼现在 200 出头就可以收到，主要是体积小巧，本来的需求也只是跑个 OpenWrt + WireGuard 组网，性能完全够用。家里的网络也没有千兆，不担心跑不满带宽的问题。

因为是要打算跟公司里的设备组网，显然公司是不会给你 IP / 端口转发的，所以家里的宽带需要申请公网 IP + 桥接。之前买了台小米的 Redmi AX6S 用作家里的软路由，但刷机不如 R2S 方便，就刷回原厂用作 AP 了。

考虑到公网 IP 拨号可能重新分配，最好还是绑定个域名做 DDNS，便于之后 WireGuard 通信，避免意外断电重启找不到 IP。
<!-- more -->
## OpenWrt 配置

因为 R2S 已经在 OpenWrt 的支持列表里了，这里简单从[官网](https://firmware-selector.openwrt.org/)下一个镜像，目前的版本是 23.05.2。

镜像选择了 SQUASHFS 格式，主要是为了方便重置配置备份升级。但缺点是默认的分区大小可能有些局促，需要进行一下扩容。

刷写完毕启动后，SSH 进入 OpenWrt，执行以下命令：

```bash
# 更换为国内镜像源加快下载
sed -i 's/downloads.openwrt.org/mirrors.ustc.edu.cn\/openwrt/g' /etc/opkg/distfeeds.conf
opkg update
opkg install cfdisk losetup f2fs-tools

# https://github.com/anaelorlinski/OpenWrt-NanoPi-R2S-R4S-Builds/blob/main/docs/resize-f2fs.md
cfdisk /dev/mmcblk0
# 选择第二个分区，扩容，写入，退出
LOOP="$(losetup -n -O NAME | sort | sed -n -e "1p")"
ROOT="$(losetup -n -O BACK-FILE ${LOOP} | sed -e "s|^|/dev|")"
OFFS="$(losetup -n -O OFFSET ${LOOP})"
LOOP="$(losetup -f)"
losetup -o ${OFFS} ${LOOP} ${ROOT}
fsck.f2fs -f ${LOOP}
mount ${LOOP} /mnt
umount ${LOOP}
resize.f2fs ${LOOP}
reboot
```

重启后扩容完成，可以继续 `opkg install` 需要的包了。

```bash
# 因为还是有后期加上 OpenClash 的打算，先替换 dnsmasq
opkg remove dnsmasq && opkg install dnsmasq-full

# OpenClash 的依赖包
opkg install coreutils-nohup bash curl ca-certificates ipset ip-full libcap libcap-bin ruby ruby-yaml kmod-tun kmod-inet-diag unzip kmod-nft-tproxy luci-compat luci luci-base

# WireGuard + 自己需要的包
opkg install luci-app-wireguard nano-full bind-dig
```

两边的 OpenWrt 要分配不同的网段，访问 OpenWrt 页面（默认地址 192.168.1.1）：

Network -> Interfaces -> Lan -> Edit -> IPv4 address

这里举例家里设备为 `192.168.10.1/24` 公司为 `192.168.20.1/24`。

## WireGuard

前述已经安装好了 WireGuard，配置时最好能够同时访问两台设备，操作都是相同的。

### 添加新接口
Network -> Interfaces -> Add new interface...

```text
Name: WireGuard
Protocol: WireGuard VPN
```

### 设置新接口
Network -> Interfaces -> WireGuard -> Edit -> General Settings

```text
Listen Port: 50251
IP Addresses: 192.168.192.10
```
这里的 IP 地址是 WireGuard 识别自身设备的地址，给两个设备设置在同一个子网 (192.168.192.0) 里就行，比如另一台设备设置为 `192.168.192.20`，末位数和 LAN 的地址数字相同便于自己识别。

点击 `Generate new key pair`，复制生成的 `Public Key` 记录下来，之后要用。

转到 `Firewall Settings`，选择 `LAN`。

然后再转到 `Peers` 点击 `Add peer` 添加对端设备。

在 `192.168.10.1` 上配置
```text
Public Key: 192.168.20.1 的 `Public Key`
Allowed IPs: 192.168.192.20/24, 192.168.20.1/24
Route Allowed IPs: ☑
Endpoint Host: 留空
Endpoint Port: 留空
Persistent Keep Alive: 25
```

在 `192.168.20.1` 上配置
```text
Public Key: 192.168.10.1 的 `Public Key`
Allowed IPs: 192.168.192.10/24, 192.168.10.1/24
Route Allowed IPs: ☑
Endpoint Host: ddns.router.home（绑定了公网 IP 的 DDNS 域名）
Endpoint Port: 50251
Persistent Keep Alive: 25
```

两边配置完成后重启 WireGuard 接口，如无意外应该能看到握手数据，互相 ping 对面设备的 LAN 地址，正常返回数据则 WireGuard 已经正常工作。

最后配置 watchdog，意外断电重启公网 IP 变动后 WireGuard 可以及时重新连接：

> echo '* * * * * /usr/bin/wireguard_watchdog' >> /etc/crontabs/root

## 配置 DNS 解析和转发

有了 WireGuard 后最方便的事情就是可以在家访问公司内网服务，只要通过位于内网路由的 DNS 来解析内网地址就行。

在内网也可以访问家里的设备，两边设置不同的域即可，对应的域转发给对应的 DNS 来解析。

Network -> DHCP and DNS -> General Settings

在 `192.168.10.1` 上配置
```text
Local server: /home/
Local domain: home
DNS forwardings: /office/192.168.192.20
Rebind protection: ☐
```

在 `192.168.20.1` 上配置
```text
Local server: /office/
Local domain: office
DNS forwardings: /home/192.168.192.10
Rebind protection: ☐
```

需要在家里（`192.168.10.1`）访问内网时，按两步走：

1. 将对应的内网域名以 `/abc.xyz/192.168.192.20` 的格式添加到 `DNS forwardings` 中
2. 将对应的解析 IP/IP段 添加到对端 Peer 的 `Allowed IPs` 中
3. 也可以直接将内网 DNS 地址添加到 `Allowed IPs` 中，`1` 中的 LAN IP 直接换成内网 DNS 地址。

这样就能用正确的 DNS 结果（通过 WireGuard 转发给公司上游 DNS 解析），访问到内网 IP 上的服务。

通过 dig 命令可以查看是否正确转发了 DNS 请求。

设置完成后可以通过 hostname.domain (比如 raspberry.office, router.home) 的方式访问两边的设备，不用再记难记的 IP 地址了。

## 主机防火墙策略

通过上述方式建立了局域网之后，理所应当就可以在局域网里共享文件进行传输了，由于两边子网不同，需要一些简单的设置。

首先打开当前连接网络的网络配置文件，看一下网络状态是`公用`还是`专用`网络。然后打开：

- Windows 10：控制面板\网络和 Internet\网络和共享中心\高级共享设置
- Windows 11：设置\网络和 Internet\高级网络设置\高级共享设置

在对应的配置文件里启用`网络发现`和`文件和打印机共享`，在`所有网络`的配置文件里启用`有密码保护的共享`。

打开防火墙设置：

- Windows 10：控制面板\系统和安全\Windows Defender 防火墙\高级设置
- Windows 11：设置\网络和 Internet\高级网络设置\Windows 防火墙\高级设置

在`入站规则`中找到`文件和打印机共享(SMB-In)`（有多个，选择对应网络配置的那一个），双击打开属性，选择`作用域`，在`远程IP地址`处选择`任何IP地址`，或者添加对应子网，比如访问家里设备的共享，就添加公司的子网地址`192.168.20.0/24`。

## 局域网文件传输

上述设置完成后，比如我们在家里电脑上共享一个文件夹`Share`：

> \\\\computer.home\\Share

我就可以直接通过这个网络地址打开共享，输入账号密码认证后像本地文件夹一样传输文件了。

至此，我终于不用受限于公司 VPN 的各种问题，也不需要靠 WSL 做桥梁，用最习惯的方式同步文件了。

而且此前设置的 OpenSSH 也能互相正常访问，爽到。

## 一些额外的折腾

完全是为了用 robocopy 而用 robocopy 的折腾，记录一下。

原因还是 SCP 不够灵活，想要类似 rsync 的体验。其实就是需要一个差异同步的功能减少传输数据量节约时间，结果还得是 robocopy，而且 robocopy 支持形如 `\\computer.home\Share` 的网络地址，完美。

考虑到 robocopy 的 `/compass` 参数需要[条件支持](https://learn.microsoft.com/zh-cn/windows-server/storage/file-server/smb-compression)，批量小文件传输速度本身也会比较慢，而后者的影响比较大，打算利用 robocopy 导出差异列表，然后打包小文件变成一个大文件来传输提高效率。

> RoboCopy [source] [dist] *.* /E /Z /COPY:DAT /DCOPY:T /XD \_\_pycache__ node_modules venv .git .github /XO /IT /XJ /R:3 /W:3 /REG /NC /NS /NDL /NJH /NJS /LOG:diff.txt

> 7z a -t7z diff.7z -i@"diff.txt" -scsWIN -spf

## EOF

这下折腾爽了。