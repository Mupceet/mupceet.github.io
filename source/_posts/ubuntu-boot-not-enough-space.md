---
title: Ubuntu 提示 boot 空间不足的解决办法
date: 2017-05-30 22:06:33
categories:
- 小技巧
tags:
- Ubuntu
---

安装双系统时，给 /boot 文件目录分配 100M 空间，其实 /boot 可以单独分成一个区，也可以不单独分，不单独分会在 / 下自动创建一个 boot 文件夹。

/boot 目录中是系统引导文件和内核，100M 的空间是足够的，但是由于更新内核之后旧内核还存放在里面，安装软件的时候就会提示 /boot 空间不足，这时最佳解决办法就是将不用的旧内核删除。

<!--more-->

**一、查看当前系统使用的内核相关信息。**

运行 `uname -a`

可以看到输出为：
```
Linux ubuntu 4.4.0-59-generic #80~14.04.1-Ubuntu SMP Fri Jan 6 18:02:02 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
```

所以使用的是 4.4.0-59-generic 的内核。

**二、查看安装的内核信息**

因为要删除 linux-image，所以可以直接运行 `sudo apt-get remove linux-image-`

然后 Tab 两次提示输入，可以看到：

```
linux-image-4.4.0-58-generic        linux-image-extra-4.4.0-59-generic
linux-image-4.4.0-59-generic        linux-image-generic-lts-xenial
linux-image-extra-4.4.0-58-generic
```

也可以输入 `dpkg --get-selections | grep linux-image`

得到输出为：
```
linux-image-4.4.0-31-generic			deinstall
linux-image-4.4.0-58-generic			install
linux-image-4.4.0-59-generic			install
linux-image-extra-4.4.0-31-generic		deinstall
linux-image-extra-4.4.0-58-generic		install
linux-image-extra-4.4.0-59-generic		install
linux-image-generic-lts-xenial			install
```

deinstall 表示该旧内核已经被删除了，需要的是删除不用的 58 的内核。

**三、删除旧内核及解决问题**

`sudo apt-get remove linux-image-4.4.0-58-generic`

删除该旧内核之后，可能提示为：
```
The link /initrd.img.old is a damaged link
Removing symbolic link initrd.img.old
 you may need to re-run your boot loader[grub]
```
运行命令 `sudo  /usr/sbin/update-grub` 更新即可。

若在安装软件过程中提示系统中有多余的不被依赖的包，可以运行 `sudo apt-get autoremove` 以删除无用的包。
