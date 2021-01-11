---
title: Windows 10 使用 OpenSSH + Rsync/SCP 同步文件
date: 2020-05-03 15:22:10
s: win10-ssh-rsync-scp
categories:
- 爬虫
tags:
- windows
- ssh
- wsl
- rsync
- scp
---

疫情期间在家工作一直使用公司的VPN，于是在公司电脑开了共享用来传输文件。但不知道为什么，公司->家方向传文件很顺畅，反方向传文件时就经常会出现进度条走到末尾就停住，要多等一段时间才能传输结束的问题。

虽然可以考虑用OneDrive这类网盘来进行同步，但这样的同步不够实时，而且毕竟要从三方服务器绕一圈，上传速度波动也很大。

想到既然Windows 10支持了OpenSSH，家里和公司电脑又都装了WSL，也许可以用rsync/scp来解决这个问题。
<!-- more -->
## 安装OpenSSH Server

Windows 10(1809)虽然自带了OpenSSH的Server和Client，但默认并没有安装。

参照[Microsoft的文档](https://docs.microsoft.com/en-us/windows-server/administration/openssh/openssh_install_firstuse)给两台电脑都安装OpenSSH：

```PowerShell
Get-WindowsCapability -Online | ? Name -like 'OpenSSH*'

# This should return the following output:
Name  : OpenSSH.Client~~~~0.0.1.0
State : NotPresent
Name  : OpenSSH.Server~~~~0.0.1.0
State : NotPresent

# Install the OpenSSH Client
Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0

# Install the OpenSSH Server
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0

# Both of these should return the following output:
Path          :
Online        : True
RestartNeeded : False

Start-Service sshd
# OPTIONAL but recommended:
Set-Service -Name sshd -StartupType 'Automatic'
# Confirm the Firewall rule is configured. It should be created automatically by setup. 
Get-NetFirewallRule -Name *ssh*
# There should be a firewall rule named "OpenSSH-Server-In-TCP", which should be enabled
# If the firewall does not exist, create one
New-NetFirewallRule -Name sshd -DisplayName 'OpenSSH Server (sshd)' -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22
```

这时候就可以从其中一台电脑向另一台发起SSH连接了。

## 生成SSH密钥并部署

考虑到之后计划使用rsync传输，与Linux类似，我们可以使用密钥登陆的方式来避免每次都要输入密码。

先设定作为Client的电脑：

```PowerShell
# 提示输入生成密钥的文件名，这里假设是 client_rsa
ssh-keygen

# 启动ssh-agent服务
Get-Service -Name ssh-agent | Set-Service -StartupType 'Automatic'
Start-Service ssh-agent

# 导入密钥
ssh-add client_rsa
```

先通过密码登录的方式SSH连接到Server的电脑上：（注意替换成自己的server_user和client_user）

```PowerShell
ssh server_user@Server_ip

mkdir C:\Users\server_user\.ssh\

scp C:\Users\client_user\.ssh\client_rsa.pub server_user@Server_ip:C:\Users\server_user\.ssh\authorized_keys
```

退出SSH会话，然后重新连接，这时应该已经不需要再输入密码了。

设置到这里的时候踩到一个坑：如果SSH登录的Server用户是管理员身份的话，`authorized_keys`需要存到`C:\ProgramData\ssh\administrators_authorized_keys`这个位置。

不然的话此时SSH登录仍然会要求密码。

```PowerShell
ssh server_user@Server_ip

scp C:\Users\client_user\.ssh\client_rsa.pub server_user@Server_ip:C:\ProgramData\ssh\administrators_authorized_keys

# 修正ACL权限
$acl = Get-Acl C:\ProgramData\ssh\administrators_authorized_keys
$acl.SetAccessRuleProtection($true, $false)
$administratorsRule = New-Object system.security.accesscontrol.filesystemaccessrule("Administrators","FullControl","Allow")
$systemRule = New-Object system.security.accesscontrol.filesystemaccessrule("SYSTEM","FullControl","Allow")
$acl.SetAccessRule($administratorsRule)
$acl.SetAccessRule($systemRule)
$acl | Set-Acl
```

之后对调Client和Server的身份再做一次同样的事情就可以了。

## 进入WSL部署rsync

只用scp存在着一定的局限性，若要使用rsync则需要依靠WSL了。

依旧从Client开始设置：

```bash
cd ~/.ssh

cp /mnt/c/Users/client_user/.ssh/client_rsa ./client_rsa

cp /mnt/c/Users/client_user/.ssh/client_rsa.pub ./client_rsa.pub

chmod 600 client_rsa
```

转到Server上：

```bash
cd ~/.ssh

cp /mnt/c/Users/server_user/.ssh/authorized_keys ./authorized_keys
```

假设Client的D盘有个testfile的文件，测试rsync：

```bash
# --rsync-path="wsl rsync" 用来指示Server通过wsl启动rsync环境
rsync -rtuv -e "ssh -i ~/.ssh/client_rsa" /mnt/d/testfile server_user@Server_ip:/mnt/d/ --rsync-path="wsl rsync"
```

如果一切正常，应该不会提示输入密码，然后你能在Server的D盘看到这个文件。

同样，对调Client和Server的身份再设定一次就可以双向操作了。

## EOF
