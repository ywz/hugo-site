---
title: "配置 ubuntu 20.04"
date: 2020-06-12T22:40:23+08:00
tags: [ubuntu,linux,proxy,java]
---

Win10 下的测试环境，通过 VMware 安装 [Ubuntu Server](https://ubuntu.com/download/server)，安装过程略。

![配置 ubuntu](/posts/images/2020-09-23_220019.png)

<!--more-->
### 静态IP
``` shell
$ cd /etc/netplan
$ sudo vim 00-installer-config.yaml
--- 
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens33:
      dhcp4: no
      addresses: [192.168.217.133/24]
      optional: true
      gateway4: 192.168.217.2
      nameservers:
              addresses: [192.168.217.2]
  version: 2
---
$ sudo netplan apply
```

### ssh
``` shell
$ sudo systemctl enable ssh
$ sudo systemctl start ssh
```

### 配置 apt
https://mirror.tuna.tsinghua.edu.cn/help/ubuntu/

### 升级 
``` shell
$ sudo apt update
$ sudo apt upgrade
$ sudo apt clean
```

### 安装软件
``` shell
$ sudo apt install vim zsh tmux mlocate git htop
```

### 代理
``` shell
$ sudo apt install proxychains-ng
$ sudo vim /etc/proxychains4.conf
---
+ http    192.168.1.3 8001
---
$ vim ~/.zshrc
---
+ alias pr='proxychains4'
---
$ pr curl google.com
```

### oh-my-zsh
``` shell
$ vim ~/.zshrc
---
+ ZSH_THEME="eastwood"
+ alias zshconfig="vim ~/.zshrc"
+ alias ll='ls -alh'
+ alias pr='proxychains4'
```

### vim
``` shell
$ vim ~/.vimrc
---
set encoding=utf-8
set tabstop=4
set softtabstop=4
set shiftwidth=4
"set laststatus=2
set showmatch
set autoindent
set incsearch
set ignorecase
set hlsearch
set number
set ruler

syntax on
filetype on

" leader
let mapleader = ' '

" edit vimrc
:nnoremap <leader>rc :e ~/.vimrc<cr>
```

### java
``` shell
$ sudo dpkg -i jdk-11.0.7_linux-x64_bin.deb
$ vim ~/.zshrc
$ sudo vim /etc/profile
---
export JAVA_HOME=/usr/lib/jvm/jdk-11.0.7/
export PATH=$JAVA_HOME/bin:$PATH
export CLASSPATH=$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export JAVA_HOME PATH CLASSPATH
---
$ source ~/.zshrc
$ java -version
```

最后，创建快照。
