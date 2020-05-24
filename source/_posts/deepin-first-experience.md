---
title: Deepin 初体验
date: 2018-01-15 23:38:33
categories:
- 小技巧
tags:
- 杂谈
---

在公司因为需要源码编译，使用的是 Ubuntu 开发，在宿舍里也同样如此。随着应用开发的比重增加，需要源码环境的时候也少了。是时候尝试一波 Deepin 了！

在武汉上学的时候就听说了这个系统。今晚就安装一下，做一个记录。

### 安装

从官网下载，并遵循[原生安装](https://wiki.deepin.org/index.php?title=%E5%8E%9F%E7%94%9F%E5%AE%89%E8%A3%85)步骤进行安装。

### Shadowsocks

安装完成之后，Shadowsocks 一定是要安装的。Deepin 15.5 版本中，可以直接获取 Shadowsocks：

```
sudo apt-get install shadowsocks-qt5
```

安装完成之后，填入自己的账号即可。

<!--more-->

### Chrome 同步

有了 Shadowsocks 那不就可以登录 Chrome 了吗？ 美滋滋。

结果打开之后不可以登录啊！傻了，想到需要一个 SwitchyOmega 扩展。但是我没登录怎么下载扩展呢？头疼。Chrome 又不像 Firfox 可以在设置里直接设置代理。

其实 Chrome 启动时是可以使用指定代理的，但是需要命令行启动：

```
google-chrome --proxy-server=socks5://127.0.0.1:1080
```

### 双系统时间不同步

简单设置之后，回到 Windows 下，发现时间不对了……

这是因为，两个操作系统对电脑硬件时间的定义不一样，Windows 认为电脑硬件时间是“本地时间”，因此它启动后直接用该时间作为“系统时间”并显示在桌面右下角的系统托盘里；而 Ubuntu 等 Linux 发行版则认为电脑硬件时间是“全球统一时间”（即 UTC），它在启动后在该时间的基础上，再加上电脑设置的时区数。

而且 Linux 是直接将硬件时间进行了修改的，因此导致时间不一致。百度之，解决方案如下：

[Win/Lin 双系统时间错误的调整](https://jingyan.baidu.com/article/154b46317b25ca28ca8f41e8.html)

### 有 1 个软件包未被升级

安装软件时意外中断，很可能在下次执行：

```
sudo apt-get upgrade
```

时提示：

```
...

升级了 0 个软件包，新安装了 0 个软件包，要卸载 0 个软件包，有 1 个软件包未被升级
```

这时候的你需要的是这一条命令：

```
sudo apt-get dist-upgrade
```

对该命令的说明可以在 help 中进行查看……因为我看了也不是太懂其含义。
