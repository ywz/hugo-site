---
title: "Win10 + Hyper-V + Ubuntu 20.04 安装"
date: 2020-11-16T21:22:38+08:00
draft: true
---

# Win10 + Hyper-V + Ubuntu 20.04 安装
## Home 版 Win10 安装 Hyper-V
创建文件 `hyper.cmd`
``` cmd
pushd "%~dp0"
dir /b %SystemRoot%\servicing\Packages\*Hyper-V*.mum >hyper-v.txt
for /f %%i in ('findstr /i . hyper-v.txt 2^>nul') do dism /online /norestart /add-package:"%SystemRoot%\servicing\Packages\%%i"
del hyper-v.txt
Dism /online /enable-feature /featurename:Microsoft-Hyper-V-All /LimitAccess /ALL
```
以管理员身份运行，重启后验证是否安装成功。

<!--more-->
## 安装 Ubuntu 20.04
唯一需要注意的在`指定代数`中选择`第一代(1)`。  

## 网络配置（访问外网）
安装完成后 Hyper-V 会自动创建默认的 switch `vEthernet (Default Switch)`。  
`控制面板\网络和 Internet\网络连接` -> WLAN（联网的那个网卡）右键 -> 属性 -> 共享

![属性](images\2020-11-16_213210.png)

配置完成后，`vEthernet (Default Switch)`会被设置为固定 ip `192.168.137.1`。  

在 Ubuntu 服务器中`ping google.com`验证网络是否生效。

## 静态 IP
``` text
sudo vim /etc/netplan/00-installer-config.yaml
---
network:
  ethernets:
    eth0:
      dhcp4: no
      addresses: [192.168.137.3/24]
      gateway4: 192.168.137.1
      optional: true
      nameservers:
        addresses: [192.168.137.1]
  version: 2
---
sudo netplan apply
```