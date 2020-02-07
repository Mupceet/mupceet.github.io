---
title: Ubuntu 安装 Node.js 的正确姿势
date: 2020-02-08 02:04:33
categories:
- 小技巧
tags:
- 杂谈
- Ubuntu
---

> 若要直接查看安装 Node.js 的正确姿势，请拉到文章最后一节。

## 安装 Node

一般情况下，你第一次安装会这样：

首先到 [Node.js 官网](https://nodejs.org/en/)下载压缩包。下载得到 `node-vXxx.tar.xz`，然后解压到 `/opt/`（适用于共享用户）或 `/usr/local/`（适用于用户个人资料）。以 `/opt/` 为例：

```shell
$ cd ~/Download
$ tar -xvf node-v10.16.3-linux-x64.tar.xz
$ sudo mv node-v10.16.3-linux-x64 /opt/node
```

然后将 `/opt/node/bin` 添加到 `PATH` 环境变量中，这样就可以从任意终端中执行 `npm` 命令了。确保环境变量生效可以执行命令看是否可以查看 node 版本。

```shell
$ node -v
```

<!--more-->

这种安装方式的问题在于要更新 Node.js 版本时，要再次手动下载压缩包，替换掉原安装路径的内容，并且不小心的话会把已下载的全局 lib 给替换没了。

于是，你在官网上仔细查看，发现了 [Installing Node.js via package manager](https://nodejs.org/en/download/package-manager/)，顿时感觉这包管理器安装才是正经办法。

第二次安装循着指引：

> Debian and Ubuntu based Linux distributions, Enterprise Linux/Fedora and Snap packages
>
> [Official Node.js binary distributions](https://github.com/nodesource/distributions/blob/master/README.md) are provided by NodeSource.

然后根据指导进行安装：

```shell
# Using Ubuntu
$ curl -sL https://deb.nodesource.com/setup_13.x | sudo -E bash -
$ sudo apt-get install -y nodejs
```

这种安装方式下，升级不用手动下载替换了，可以使用一个 npm 模块 `n` 来升级，可以参见：[升级node.js和npm](https://segmentfault.com/a/1190000009025883)。这里不具体展开，因为在这之前就有另外的问题让你头疼。

## 问题

### 无法下载

你安装好 Node.js 之后，准备安装一个包，常因为网络原因迟迟无法完成下载。这时候，你需要[淘宝 NPM 镜像](http://npm.taobao.org/)，使用方法很简单：

* 可以使用定制的 cnpm (gzip 压缩支持) 命令行工具代替默认的 npm:

```shell
$ npm install -g cnpm --registry=https://registry.npm.taobao.org
```

* 或者直接通过添加 npm 参数 alias 一个新命令:

```shell
$ alias cnpm="npm --registry=https://registry.npm.taobao.org \
--cache=$HOME/.npm/.cache/cnpm \
--disturl=https://npm.taobao.org/dist \
--userconfig=$HOME/.cnpmrc"

# Or alias it in .bashrc or .zshrc
$ echo '\n#alias for cnpm\nalias cnpm="npm --registry=https://registry.npm.taobao.org \
  --cache=$HOME/.npm/.cache/cnpm \
  --disturl=https://npm.taobao.org/dist \
  --userconfig=$HOME/.cnpmrc"' >> ~/.zshrc && source ~/.zshrc
  ```

### 提示无权限

npm 可以把包安装到全局目录，也可以安装到本地本地，常用的工具类的包通常是在命令行中作为命令使用，如果不安装到全局目录则在执行命令时要加上命令的路径，很不方便。

当你尝试全局安装某个包的时候，等待许久最终却看到 EACCES 错误，提示你没有权限写入用于存储全局包和命令的目录，真是令人崩溃。

两种解决办法，一种是修改全局目录为当前用户有权限的目录，具体可参考：[更改npm全局包安装目录的解决方案](https://www.jianshu.com/p/0cff9f4167c9)。

另一种则是既然没有权限，那加上 sudo 来执行吧: 

```shell
$ sudo npm install -g xxxxx
sudo: npm：command not found
```

会提示找不到命令，需要参考《[[译] sudo后使用别名](https://legolasng.github.io/2016/10/21/using-sudo-with-an-alias/)》进行修改：

```shell
$ alias sudo='sudo '
```

这样就可以了，但每次安装都要 sudo 其实并不愉快。事实上，我们一开始就可以选择更合适的 Node.js 的安装方法。

## NVM

> NVM: 全称是 Node Version Manager, 也就是 Node 版本管理器。

如果你已经安装过了 Node，最好先把原来的卸载。

### 1. 卸载 Node, 可能需要 root 权限.

```shell
$ sudo apt-get remove nodejs
```

### 2. 移除你之前的全局 node_modules 包.

```shell
#执行前请确认这个包是否存在这个位置
$ sudo rm -rf /usr/lib/node_moudles
```

### 3. 安装 NVM

查看 NVM 的 [Github 仓库](https://github.com/creationix/nvm)：

```shell
$ curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.35.2/install.sh | bash
```

然后设置环境变量：

```shell
$ export NVM_DIR="$([ -z "${XDG_CONFIG_HOME-}" ] && printf %s "${HOME}/.nvm" || printf %s "${XDG_CONFIG_HOME}/nvm")"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh" # This loads nvm
```

### 4. 使用 NVM

```shell
# 将安装最新版本
$ nvm install node
# 列出可安装版本，选择安装某一版本
$ nvm list
$ nvm install 12.14.1 # or 10.10.0, 8.9.1, etc
```

NVM 最大的好处就是你可以安装多个版本的 node 到你的系统里，直接一条命令就可以切换版本。

```shell
$ nvm use v13.6.0
```

更多的使用说明可直接查看 [Github 仓库](https://github.com/creationix/nvm)的 README 文档。

现在查看使用 NVM 下载的 node 命令的目录：

```shell
$ where node
/home/mupceet/.nvm/versions/node/v13.6.0/bin/node
```

可以看到现在使用的目录都是在 Home 目录下，权限问题也就不成问题了，再加上[淘宝 NPM 镜像](http://npm.taobao.org/)，体验上就很舒服了。


## 参考链接

1. [Node.js 官网](https://nodejs.org/en/)
1. [升级node.js和npm](https://segmentfault.com/a/1190000009025883)
1. [淘宝 NPM 镜像](http://npm.taobao.org/)
1. [更改npm全局包安装目录的解决方案](https://www.jianshu.com/p/0cff9f4167c9)
1. [[译] sudo后使用别名](https://legolasng.github.io/2016/10/21/using-sudo-with-an-alias/)
1. [ubuntu中npm安装全局插件提示没有root管理员权限](https://zhouyuexie.github.io/ubuntu%E4%B8%ADnpm%E5%AE%89%E8%A3%85%E5%85%A8%E5%B1%80%E6%8F%92%E4%BB%B6%E6%8F%90%E7%A4%BA%E6%B2%A1%E6%9C%89root%E7%AE%A1%E7%90%86%E5%91%98%E6%9D%83%E9%99%90/)
1. [nvm-sh/nvm](https://github.com/nvm-sh/nvm)