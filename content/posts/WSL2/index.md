---
title: "Windows如何安装WSL2"
date: 2026-11-25
draft: false
tags: ["WSL2", "操作系统", "WSL"]
categories: ["日志"]
summary: "WSL"
---

## 安装WSL

去 GitHub Release 页下最新 .msixbundle（需要梯子加速下载）：[Github](https://github.com/microsoft/WSL/releases/latest)

管理员 PowerShell 安装下载的程序包：
```
Add-AppxPackage "D:\Downloads\Microsoft.WSL_xxx_x64_ARM64.msixbundle"
```

装完验证：
```
wsl --version
```

若能正常输出，则重启电脑。

## 安装对应的Ubuntu

重启后，可以直接install，下载默认版本，也可以下载指定版本：
```
WSL2一键安装命令（自动启用功能、下载发行版）
wsl --install

# 查看可安装的 WSL 分发版（可选）
wsl --list --online
# 简写
wsl -l -o
# 安装指定分发版
wsl --install -d Ubuntu-22.04
```

首次启动会设置用户名和密码，完成这一步后更换国内源，Ubuntu默认源是国外服务器，国内访问速度慢，替换为阿里源/清华源，更新和下载软件会快很多（以下以阿里源为例）。

1. 打开Ubuntu终端，先备份默认源文件（避免修改错误无法恢复），输入以下命令，回车执行（需要输入之前设置的Ubuntu密码）：

```
# 备份默认源文件（备份到/etc/apt/sources.list.bak）
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
```

2. 编辑源文件，输入以下命令，打开源文件编辑器
```
sudo nano /etc/apt/sources.list
```

复制以下阿里源内容，粘贴到编辑器中（在Ubuntu终端中，右键即可粘贴）：
```
# 阿里源（Ubuntu 22.04 LTS专用）
deb http://mirrors.aliyun.com/ubuntu/ jammy main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ jammy main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ jammy-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ jammy-security main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ jammy-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ jammy-updates main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ jammy-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ jammy-backports main restricted universe multiverse
```

粘贴完成后，按下「Ctrl+O」保存文件，再按下「Enter」确认保存，最后按下「Ctrl+X」退出编辑器；

3. 更新源列表，使国内源生效，输入以下命令，回车执行（耐心等待更新完成，约1-2分钟）：

```
sudo apt update
```


