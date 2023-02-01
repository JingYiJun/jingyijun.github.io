---
title: "Linux Debian 部署远程局域网打印机"
description: "使用cups打印服务将USB打印机共享到局域网"
date: 2022-07-04T18:18:00+08:00
lastmod: 2023-02-01T14:00:00+08:00
math: false
license: 
hidden: false
comments: true
draft: false
categories:
    - 折腾
tags:
    - Linux
    - Debian
    - 打印机
---

## 需求

家里需要一个局域网共享打印机，但是 EPSON L211 打印机不支持网络共享、仅支持 USB 连接，如果连 Windows 系统，开启共享打印服务很方便，但是台式机不经常开启。恰好前段时间装了一个 PVE 系统的 NAS，并且 24 小时开机，可以用来部署远程局域网打印机服务。

## 设备

- 自组 x86_64 NAS ，基于 Debian 11 的 PVE
- EPSON L210 Series 打印机

## 实施流程

### 安装 CUPS

CUPS（Common UNIX Printing System）是一个由苹果公司（Apple Inc.）开发的开源打印程序系统，支持 IPP（网络打印协议）、Web 接口管理、扫描仪管理等功能，兼容 MacOS、IOS、IpadOS 和 Linux 操作系统，各大 Linux 发行版的软件包里也都集成了 CUPS 软件，因此可以很方便地做打印服务器。

Debian 桌面发行版是自带了 CUPS，用于桌面打印服务，而服务器版本没有，需要自己安装。

```bash
> sudo apt-get install cups cups-pdf
```

系统默认安装 cups 服务，默认只监听本机的 631 端口（如果有防火墙的话，要放行 631 端口）。于是研究了一下 cups 的配置文件 `/etc/cups/cupsd.conf`，修改一下内容，默认是本机访问.

```shell
Listen localhost:631
```

改为

```shell
Listen 0.0.0.0:631 # 监听所有端口，由于是局域网打印机，没有暴露风险
```

同时修改放行条件和管理员条件：

```vim
# Restrict access to the server...
<Location />
  Order allow,deny
  Allow From 192.168.0.0/16
</Location>

# Restrict access to the admin pages...
<Location /admin>
  Order allow,deny
  Allow From 192.168.0.0/16
</Location>
```
然后重启 cupsd 服务
```shell
> sudo service cups restart
# 或者
> sudo systemctl restart cups
```

### 安装打印机驱动

检查打印机是否已经连接。

```shell
> lsusb # usb 查询打印机，软件是 usbutils，没有的话要下载
...
Bus 005 Device 002: ID 04b8:08a1 Seiko Epson Corp. EPSON L210 Series
...
```

EPSON 打印机的驱动在官网提供下载
[EPSON Linux 打印机驱动官网](http://download.ebz.epson.net/dsc/search/01/search/?OSC=LX)，搜索打印机型号，找到 `ESC/P Driver` （非 `Printer Utilities`），（用 wget 或者其他方式）下载对应的驱动软件包（Debian 系统对应 deb 包）到系统的非 root 目录下。然后执行

```shell
> sudo apt install ./epson-inkjet-printer-201207w_1.0.0-1lsb3.2_i386.deb
```

显示缺少依赖 `lsb3.2`。

尝试用 `apt` 安装 `lsb` 或者 `lsb-compat`，发现没有该软件包，原因是 Debian 从 2015 年起抛弃了 `lsb` 软件包，仅在 debian 9 保留了一个 `lsb-compat` 包，最新的系统把 `lsb-compat` 都抛弃了，所以需要去 debian 官网，找到 debian 9 (stretch) 的 `lsb-compat` 包下载后，手动安装。

```bash
> sudo apt install ./lsb-compat_9.20161125_amd64.deb
> sudo apt install ./epson-inkjet-printer-201207w_1.0.0-1lsb3.2_i386.deb
```

会自动安装以下依赖 `lib32z1 libc6-i386 libcupsimage2 lsb-compat`。

** 注意 **：下载包的时候在非 root 用户文件夹下安装，不然会报错：_apt 权限不足无法访问。

至此驱动安装完成。

### 安装打印机到 cups

用浏览器访问 `https://[ip]:631`，[ip] 是 NAS 的 ip 地址，进入 cups 的管理界面。如果访问不到的话，考虑关闭防火墙。选择 Add printer 添加打印机。cups 会自动检测到 EPSON L211 的打印机，然后一路往下点。选择型号驱动，在下拉菜单里面找到 EPSON L210 系列的 model，就好了，可以打印测试页。

### Windows 11 连接到远程打印机

打开 “设置——蓝牙和其他设备——打印机和扫描仪——添加设备”，会找到同一局域网内的 cups 打印设备，连接即可。

## 拓展

### ipp Internet Printing Protocol 网络打印协议

IPP 协议全称 Internet Printing Protocol，默认使用 631 端口，该协议基于 HTTP 实现，新标准里亦有基于 HTTPS 的实现。

CUPS 实现了 IPP 协议，并且提供了很方便的打印解决方案和管理方案，所以是 Linux、MaxOS、BSD 等操作系统的默认和常用打印服务器

## 后记

不折腾了，买了一台网络打印机 EPSON L4268，既能实现网络打印、远程打印，也不用费劲折腾。还挺好的。

## 参考资料：
1. [HP Gen8 + Epson L211 + Ubuntu 搭建内部服务器](https://zhuanlan.zhihu.com/p/23103582)
2. [Debian: unable to install lsb package](https://unix.stackexchange.com/questions/545540/debian-unable-to-install-lsb-package)
3. [在 Debian 上设置 USB 网络打印机和扫描仪服务器](https://linux.cn/article-4139-1.html)
4. [Epson L211 Linux 打印机驱动下载地址]((http://download.ebz.epson.net/dsc/f/01/00/01/87/87/b90c68a7844fff22a3f4b4c273fcf3407699f89c/epson-inkjet-printer-201207w_1.0.0-1lsb3.2_amd64.deb))
5. [debian 9 (stretch) 的 lsb-compat 软件包下载地址](https://mirrors.tuna.tsinghua.edu.cn/debian/pool/main/l/lsb/lsb-compat_9.20161125_amd64.deb)