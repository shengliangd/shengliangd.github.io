---
layout:     post
title:      "在 Debian 上安装向日葵远程控制"
subtitle:   ""
date:       2020-09-19 19:00:00
tags:
    - Misc
typora-root-url: ../
published: true
---

受疫情影响在家用 Teamviewer 远程连接实验室的机器有一阵子了，体验相当差，比如在大西北经常登录不上账号、偶尔被检测为商业用途等。今天发现了[向日葵远程控制](https://sunlogin.oray.com/)，对于 Linux，该软件提供了 Ubuntu/Deepin 和 CentOS 两种安装包，而我实验室的机器上装的是 Debian，原则上 Ubuntu/Deepin 的安装包是适用的，但实际安装过程中遇到了两个问题，在这里分享一下解决方案。

下载安装包后用 `dpkg -i xxx.deb` 安装，先报了下面的错误（只截取最相关的部分，下同）：

![image-20200919192528098](/img/2020-09-19-install-sun/missing package.png)

这是常见的问题，盲猜 Debian Stretch 的软件仓库里有，去 `/etc/apt/sources.list` 里把 buster 改成 stretch，然后 `apt update`，然后 `apt install -f`，过了这个坎儿。

下一个报错如图：

![image-20200919193204159](/img/2020-09-19-install-sun/unknown OS.png)

看来它还会检查系统发行版信息（啊这句奇怪的“unknown OS it not impl”）。既然官网说了是 Ubuntu/Deepin 安装包，那就伪造一下 Ubuntu 吧。发行版信息主要存在两个文件里：`/etc/issue` 和 `/etc/os-release`，google 一下找到了 Ubuntu 18.04 中这两个文件的内容，如下：

`/etc/issue`：

```
Ubuntu 18.04.1 LTS \n \l
```

`/etc/os-release`：

```
NAME="Ubuntu"
VERSION="18.04.1 LTS (Bionic Beaver)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 18.04.1 LTS"
VERSION_ID="18.04"
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
VERSION_CODENAME=bionic
```

把原有文件备份一下，然后改成上面这两个文件的内容，然后重新 `dpkg -i xxx.deb`，会报一个新的错误（别害怕）：

![yet another error](/img/2020-09-19-install-sun/yet another error.png)

看起来不是什么大问题，再 `dpkg -i xxx.deb` 一下，就装好了：

![image-20200919195418542](/img/2020-09-19-install-sun/verify.png)

滑稽的是，鉴于发行版信息是能通过 `/etc/os-release` 等文件修改的，上图并不能证明上述方法的有效性，本文真实性请自行判断（

最后别忘了把 `/etc/apt/sources.list`、`/etc/issue`、`/etc/os-release` 改回去。`/usr/lib/os-release` 虽然没有手动修改，但也跟着变了，所以也要改回去，具体原因没深究。

