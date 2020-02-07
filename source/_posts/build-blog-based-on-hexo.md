---
title: Hexo 博客搭建与主题配置（零基础版）
date: 2019-08-27 22:21:33
updated: 2020-02-08 02:16:33
categories:
- 整理
tags:
- 杂谈
---

每个技术人多少都应该写几篇博客，可以选择发布在各大平台如：简书、掘金等；也可以发布于自己搭建的博客。

本文记录了基于 Hexo 框架 + GitHub 搭建个人博客的过程，你可以按照此文进行实践，也期待看到你自己的博客。

> 本文的实践基于 Deepin 15.11, 博客地址：[Mupceet](https://mupceet.com/)

## 环境搭建

Hexo 的介绍及详细信息请查看[官方文档](https://hexo.io/zh-cn/docs/index.html)，以下记录实操的具体细节。

### 安装 [Git](https://git-scm.com/)

```shell
$ sudo apt install git
```

[Hexo 文档](https://hexo.io/zh-cn/docs/index.html)中写的 Git 的安装命令为 `sudo apt-get install git-core`，这是因为老一点的 Ubuntu 中有一个软件也叫 GIT（GNU Interactive Tools），所以 Git 只能叫 `git-core` 了，后来因为 Git 的名气实在太大，所以 GNU Interactive Tools 就改名了，`git` 就变成了真正的 Git。

<!--more-->

安装完成后记得配置一下：

```shell
$ git config --global user.email "you@example.com"
$ git config --global user.name "Your Name"
```

### 安装 [Node.js](https://nodejs.org/en/)

**建议参考 [Ubuntu 安装 Node.js 的正确姿势](https://mupceet.com/2020/02/the-best-way-to-install-nodejs/)使用 NVM 方式进行安装。(2020-02-08)**

选择下载最新的 10.16.3 LTS 版本（*2019-08-28*），得到 `node-v10.16.3-linux-x64.tar.xz`，解压到 `/opt/`（适用于共享用户）或 `/usr/local/`（适用于用户个人资料）。以下以 `/opt/` 为例。

```shell
$ cd ~/Download
$ tar -xvf node-v10.16.3-linux-x64.tar.xz
$ sudo mv node-v10.16.3-linux-x64 /opt/node
```

将 `/opt/node/bin` 添加到 `PATH` 环境变量中，这样就可以从任意终端中执行 `npm` 命令了。

```shell
$ sudo vim /etc/profile
```

打开 `/etc/profile` 文件，增添以下内容，注意**等号前后没有空格**，保存退出。

```txt
# Node.js
export NODE_HOME=/opt/node/bin
export PATH=$PATH:$NODE_HOME
```

为了使该环境变量生效，可以在终端中执行 `source /etc/profile` 或者 `. /etc/profile`，但这样只在当前终端中生效。**要使得任意终端都生效，退出当前用户再登陆即可。**

> Deepin 下 zsh 使用 source 的方式会有问题，请使用注销再登录的方式。

也可以直接将环境变量配置到 Shell 的配置文件中，如 `~/.bashrc` 或 `~/.zshrc`，这样，重启终端该环境变量即可生效。

确保环境变量生效可以执行命令看是否可以查看 node 版本。

```shell
$ node -v
v10.16.3
```

Node 自带 `npm`，所以装完应该会有某个版本的 `npm`。但 `npm` 相比 Node 更新更频繁，所以，要是想确保使用的是最新版本的，你可以执行以下命令：

```shell
$ npm install npm -g
```

### 安装 [Hexo](https://hexo.io/zh-cn/docs/index.html)

```shell
$ npm install -g hexo-cli
```

如果网络条件不好——你懂的，比较难以下载成功，这个自行解决。如果碰到权限问题，可以参考链接：[处理npm权限问题](https://www.kancloud.cn/shellway/npm-doc/199985)

安装后可查看版本信息。一般来说不需要去关心这些信息，我只是看看……

```shell
$ hexo -v
hexo: 3.9.0
hexo-cli: 2.0.0
os: Linux 4.15.0-29deepin-generic linux x64
http_parser: 2.8.0
node: 10.16.3
v8: 6.8.275.32-node.54
uv: 1.28.0
zlib: 1.2.11
brotli: 1.0.7
ares: 1.15.0
modules: 64
nghttp2: 1.39.2
napi: 4
openssl: 1.1.1c
icu: 64.2
unicode: 12.1
cldr: 35.1
tz: 2019a
```

## 建站及配置

### 建站初始化

选择在 Documents 目录下创建 blog 文件夹来存放源文件。

```shell
$ cd ~/Documents
$ hexo init blog
$ cd blog
$ npm install
```

初始化完成后，目录内容结构为如下内容：

```txt
.
├── _config.yml
├── node_modules
├── package.json
├── package-lock.json
├── scaffolds
├── source
└── themes
```

### 生成文件及本地调试

初始化后执行 `hexo generate` 或 `hexo g` 可生成静态文件（`public` 文件夹）与缓存文件（`data.json`）。

然后我们执行 `hexo server` 或 `hexo s` 就可以启动本地服务器，访问网址 `http://localhost:4000/` 就可以查看文章效果了。

> 此处本该有预览图，但你已经看到这里了，看自己的效果吧：）

接下来就是部署网站等**简单的**操作，好了，本文到此为止，大家自行实践吧。

……

好吧，重点配置在后面……

### 部署远程服务器

感谢 [GitHub](https://github.com/) 提供的 Pages 服务，可用来部署静态网页。首先你有这个网站的账号，这里就不具体展开了。

注册账号之后，记得上传一下 SSH 公钥，便于部署、上传操作。具体操作请查看网站的帮助文档：[Github: Connecting to GitHub with SSH](https://help.github.com/articles/connecting-to-github-with-ssh/)

#### 创建 [GitHub Pages](https://pages.github.com/)

1. 创建项目

   创建一个与你用户名对应的项目：`username.github.io` ，例如我创建的项目为 `https://github.com/Mupceet/mupceet.github.io`。
    > 从这个例子中可以看到，这里 username 大小写可以不完全对应。

1. 部署内容

   将 Hexo 生成的静态文件上传到该仓库中，就可以通过访问 `username.github.io` 来查看你的博客，也就是可以看到刚才本地调试时看到的网页了。这个上传动作可以使用 Git 手动操作，将 Public 文件夹的内容 Push 到仓库的主分支上，但更建议使用 `hexo deploy` 来操作。

#### 部署

上面提到了要使用命令 `hexo deploy` 上传文件，这需要下载插件，并且要修改站点配置文件 `_comfig.yml`，关于这个配置文件后续会介绍，这里暂时先修改部署所需要的配置。

首先安装相关插件：

> 注意：插件下载时需要在博客的文件夹中，在本文就是 blog 文件夹

```shell
$ cd ~/Documents/blog
$ npm install hexo-deployer-git --save
```

然后修改配置，以我的配置为例：

```yml
# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repo:
    github: git@github.com:Mupceet/mupceet.github.io.git,master
```

当然，该插件还支持同时上传内容到不同的仓库，如果有需求可以自行探索。

最后执行 `hexo deploy` 或者 `hexo d` 就可以将其部署到服务器上。现在访问对应的域名 `username.github.io` 就可以看到你自己的博客啦！

后续更新文章内容或者配置，标准流程如下：

```shell
$ hexo g
$ hexo s
$ hexo d
```

事实上，我每次本地做了修改，执行 `hexo g` 生成内容后，都会使用 `hexo s -p 2333 --debug` 然后访问 `http://localhost:2333/` 查看排版格式是否正确、是否有错别字。加 debug 参数是为了可以在访问页面时看到调试信息，便于定位、解决问题。在调整到本地查看效果满意后，最后再 `hexo d` 部署到远程，完成一次博客的更新。

### 站点配置文件

上文提到了 `_config.yml` 这一配置文件，它是关于网站的一些配置，具体说明可见[ Hexo 官方文档](https://hexo.io/zh-cn/docs/configuration.html)。以下对官网有详细说明的内容就不再赘述，优先查看官网文档也是个好习惯。

```yml
# Site 网站
# URL 网址
# Directory 目录
# Writing 文章
# Home page setting 主页相关设置
# Category & Tag 分类 & 标签
# Date / Time format 日期 / 时间格式
# Pagination 分页
# Extensions 插件
```

以上内容在官网中都有详细的说明介绍，在你搭建的开始，不需要在配置上面花费过多的精力，大部分保持默认设置即可。这里只补充说明有必要修改的部分。

#### Site 网站

这一部分是显示在页面上的重要基本信息，如网站的标题、作者、说明等。这一定是要修改的，以我的配置为例：

```yml
# Site
title: Mupceet # 博客标题
subtitle: Don't Repeat Yourself # 博客副标题
description:  君子坦荡荡 小人长戚戚 # 博客描述，部分主题会用来生成简介
keywords: # 通常建议包含网站的关键词
author: Mupceet # 博客作者，部分主题会用来显示作者
language: zh-Hans # 语言，具体需要查看主题theme说明
timezone: # 默认使用电脑的时区
```

#### URL 网址

这一部分配置你的博客链接的具体格式形式，我参照使用了 [Android Developers Blog](https://android-developers.googleblog.com/) 的 URL 格式：

`https://android-developers.googleblog.com/2018/10/kotlin-momentum-for-android-and-beyond.html`

即：域名 + 年 + 月 + 以 - 为连接符的英文标题，因此修改配置如下：

```yml
# URL
## If your site is put in a subdirectory, set url as 'http://yoursite.com/child' and root as '/child/'
url: http://yoursite.com
root: /
# permalink: :year/:month/:day/:title/
permalink: :year/:month/:title/
permalink_defaults:
```

#### Home page setting 主页相关设置

默认配置文件中只写了 `index_generator` 这部分，也就是首页的配置，事实上，查看 `node_modules` 文件中，可以看到有以下几个 `generator`：

```
├── hexo-generator-archive
├── hexo-generator-category
├── hexo-generator-index
├── hexo-generator-tag
```

这也表明了我们博客可以通过以上几个维度来展示你的文章，它们分别是归档、分类、索引、标签，其中对于分类与标签我个人理解是一篇文章最好使用一个类别(category)和多个标签(tag)。

分别查看这些 module 的 README 文件，可以将它们内容组合起来，还是以我的配置为例进行说明：

```yml
# path: Root path for your blogs index page. (default = '')
# per_page: Posts displayed per page. (0 = disable pagination)
# order_by: Posts order. (Order by date descending by default)
index_generator: # 首页文章列表页面
  path: ''
  per_page: 10 # 首頁显示多少篇文章，默认为10，如果值为0则不分页。由于首页会显示文章的摘要内容，建议不要取太大的数值，造成翻页定位困难
  order_by: -date # 排序，默认时间逆序，即最新的在最上面

archive_generator: # 归档文章列表页面
  per_page: 20 # 默认10个文章标题，由于仅显示标题，可适当设置为较大的数值
  yearly: true  # 生成年视图
  monthly: true # 生成月视图
  daily: false # 生成日视图，默认关
  order_by: -date # 排序

tag_generator: # 标签中各个标签下文章列表页面
  per_page: 20 # 默认10篇文章

category_generator: # 分类中各个分类下文章列表页面
  per_page: 20 # 默认10篇文章
```

#### Extensions 插件

Hexo 有强大的插件系统，丰富的插件给 Hexo 带来了生气。

上面部署一节提到了 `deploy` 插件安装与使用。可以看到一个插件的生效不仅需要下载，可能还需要修改配置，后续我们会接触的几个插件皆是如此。

另一个插件就是主题(theme)。

```yml
## Themes: https://hexo.io/themes/
theme: landscape
```

可以看到默认的主题为 `landscape`，我们可以从 [Themes](https://hexo.io/themes/) 中选择更多好看优雅的主题。

你可以和我一样选择非常流行的 [NexT](https://github.com/theme-next/hexo-theme-next) 主题，它的介绍及说明见：[精于心，简于形](http://theme-next.iissnan.com/)，另外也可查看[官方博客](https://theme-next.org)，下面我们先简单介绍如何使用此主题。

## Hexo 主题：NexT

作为开发者，我更喜欢使用 master 分支，这将包含最新的特性。

```shell
$ cd blog
$ git clone https://github.com/theme-next/hexo-theme-next themes/next
```

后续主题可以及时更新到最新版本。

```shell
$ cd blog/themes/next
$ git pull --rebase
```

现在，我们有两份主要的配置文件，其名称都是 `_config.yml`：

* `blog/_config.yml`：Hexo 本身的配置，我们称为`站点配置文件`
* `blog/themes/next/_config.yml`：主题相关配置，我们称为`主题配置文件`

在 Hexo 中切换主题需要修改站点配置文件中的 `theme` 一节：

```yml
## Themes: https://hexo.io/themes/
theme: next
```

执行 `hexo g` 和 `hexo s` 就可以查看使用了 NexT 主题的博客样式了。

> 此处本该有预览图，但你已经看到这里了，看自己的效果吧：）

从预览可以看到，虽然站点配置文件设置了网站的语言为 `zh-Hans` 但还是英语显示（我是说主页与归档这两个词……），原因其实很简单，NexT 主题中，中文需要指定的 `language` 字段不是 `zh-Hans`，而是 `zh-CN`，所以 `language` 字段的配置需要查看具体的主题是如何定义。NexT 的语言列表对应关系可见 NexT 主题语言文件夹：`themes/next/language/`，修改之后再次预览就可以看到切换为中文显示了。

修改了站点配置文件已经可以使用上简洁的 NexT 主题了，而主题还可以进行配置使得显示效果更符合你的心意。

### 主题配置文件

#### 外观（Scheme）

NexT 主题通过 Scheme 设置可展现出不同的外观。而且几乎可以说所有的配置都可以在 Scheme 之间共用。

修改主题配置文件 `scheme` 一节：

```yml
# Schemes
#scheme: Muse
#scheme: Mist
#scheme: Pisces
scheme: Gemini
```

当前提供了四种样式，去掉某一样式的行首注释即可使用，修改之后可预览查看显示效果。你可以修改选择自己喜欢的样式。

#### 菜单（Menu）

在修改站点配置文件的时候说到博客可以显示归档、分类、标签等，下面展示如何修改主题配置文件使得能够真正地展示这几个页面。

修改主题配置文件 `menu` 一节：

```yml
menu:
  home: / || home
  #about: /about/ || user
  tags: /tags/ || tags # 取消注释，需要手动创建该页面
  categories: /categories/ || th # 取消注释，需要手动创建该页面
  archives: /archives/ || archive
  #schedule: /schedule/ || calendar
  #sitemap: /sitemap.xml || sitemap
  #commonweal: /404/ || heartbeat
```

取消 tags 和 categories 两行的注释，预览可以看到已经在界面上显示这两个菜单了，但单击选择时页面提示：`Cannot GET /categories/` ，这是因为我们还未创建对应的页面。

```shell
$ hexo new page "tags"
INFO  Created: ~/Documents/blog/source/tags/index.md
$ hexo new page "categories"
INFO  Created: ~/Documents/blog/source/categories/index.md
```

执行命令后还需要修改 index.md 。

分类与标签页不需要标题，当然你也可以自定义一个合适的标题，选择去掉 `title: ` 或修改它。另外，后续博文内容页会集成评论服务，页面会带有评论功能，因此添加 `comments: false` 在此页面上关闭评论功能。并声明对应的 `type: tags` （`type: categories`） 使得在文章中进行配置时可进行归类。

以 tags/index.md 修改为例（tags/index.md 和 categories/index.md 是一样的修改）：

```md
---
date: 2018-10-06 09:59:20
comments: false
type: tags
---
```

现在博客上就有了这两个页面，只不过，现在页面上还是空的，因为，文章并没有增加这两个属性，我们需要在文章头增加对应的内容才能显示到这两个页面上。


## 创建文章

目前为止还没有自己创建过文章，经过以上配置，我们就可以开始正式地创建文章了：）

创建新文章命令为：`hexo new post "name"`，这是以 `scaffolds/post.md` 为模板创建文章，我们打开该模板，可以看到已经添加了 tags，我们还需要添加 categories，这样创建出的文章中添加的标签即可就可以归类到对应的页面下。如前所述，一篇文章最好使用一个类别(category)和多个标签(tag)。

```
---
title: {{ title }}
date: {{ date }}
categories:
- 分类
tags:
- 标签
- 其他
---
```

我们真正来创建博客的第一篇文章：

```
$ hexo new post "my-first-post"
INFO  Created: ~/Documents/blog/source/_posts/my-first-post.md
```

修改 my-first-post.md：

> Tips: 使用 &lt; !--more--&gt; 将文章分为两部分，首页只展示前面部分，这样首页浏览更方便

```
---
title: my-first-post
date: 2018-10-06 10:10:24
categories:
- 技术向
tags:
- Android
- Performance
---

这是我发布的第一篇文章！

<!--more-->

这是更多的内容。
```

重新使用 hexo 命令生成、预览、发布，就可以看到博客内容的更新了！

## 完成基础搭建

经过以上步骤，一个 Hexo 博客已经搭建出来了，并且我们可以愉快地发表文章了！

后续可修改的内容还有很多，可以给博客添加更多的功能如搜索、评论、阅读量统计等，还可以对博客进行个性化定制如头像、背景的修改等等。这些内容我们可以查看主题的说明文档、网站、配置文件，它们一般都对支持的配置进行了说明。

博客的定制化修改不做过多说明，而是来考虑一个问题：程序员一般都不只一台电脑，想在不同电脑上都能维护博客怎么办？或者说以上配置、发布的内容丢失了，博客又怎么找回？

查看远程仓库主干上的内容，可以看到，上传的只有编译好的网页文件，而没有这些博客源文件、文章、主题、配置等，因此我们需要合理备份这些内容来解决上面这个问题。
