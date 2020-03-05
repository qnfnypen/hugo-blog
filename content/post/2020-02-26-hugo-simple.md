+++
title="Hugo的简单使用"
categories=["工具"]
tags=["Hugo"]
description="使用Hugo搭建自己的博客"
date=2020-02-26T16:51:13+08:00
toc=true
+++

## 一：Hugo的安装
1. **二进制安装（推荐：简单、快速）**
到[Hugo Releases](https://github.com/gohugoio/hugo/releases)下载对应的操作系统版本的Hugo二进制文件（hugo或者huogo.exe)
Mac下直接使用`Homebrew`安装：
    ```shell script
    brew install hugo
    ```
2. **源码安装**
源码编译安装，首先安装好依赖的工具：
    + Git
    + Go 1.3+ (Go 1.4+ on Windows)

    设置好`GoPATH`环境变量，获取源码并并编译：
    ```shell script
    go get -v github.com/spf13/hugo
    ```
    源码会下载到`$GOPATH/src/`目录，二进制在`$GOPATH/bin/
    `
    如果需要更新所有Hugo的依赖库，增加`-u`参数：
    ```shell script
    go get -u -v github.com/spf13/hugo
    ```
    在源码目录使用`go istall`进行编译，生成可执行文件

## 二：Hugo的简单使用
1. **生成站点**
使用Hugo快速生成站点，比如希望生成到`/path/to/site`路径：
    ```shell script
    hugo new site /path/to/site
    ```
    这样就在`/path/to/site`目录里生成了初始站点，进去目录：
    ```shell script
    cd /path/to/site
    ```
    站点目录结构：
    ```shell script
    ├── archetypes
    │   └── default.md
    ├── config.toml
    ├── content
    ├── data
    ├── layouts
    ├── static
    └── themes
    ```
2. **创建文章**
创建一个`about`页面：
    ```shell script
    hugo new about.md
    ```
    `about.md`自动生成到了`content/about.md`，打开`about.md`看下：
    ```shell script
    +++
    date = "2015-10-25T08:36:54-07:00"
    draft = true
    title = "about"
    +++

    正文内容
    ```
    内容是`Markdown`格式的，`+++`之间的内容是[TOML](https://github.com/toml-lang/toml)格式的，根据你的喜好，你可以换成[YAML](https://yaml.org/)格式（使用`---`标记）或者[JSON](https://www.json.org/json-en.html)格式。
    创建第一篇文章，放到`post`目录，方便之后生成聚合页面。
    ```shell script
    hugo new post/first.md
    ```
    打开编辑`post/first.md`
    ```shell script
    ---
    date: "2015-10-25T08:36:54-07:00"
    title: "first"
    ---

    ### Hello Hugo

    1. aaa
    1. bbb
    1. ccc
    ```
3. **安装主题**
本站使用的主题是`maupassant`,如果你觉得本站的主题不错，可以把它克隆到`themes`目录下来：
    ```shell script
    cd themes
    git clone git@github.com:flysnow-org/maupassant-hugo.git
    ```
    当然，Hugo提供了很多主题，你可以选择你喜欢的主题。
    + [Hugo主题简介](https://3mile.github.io/archives/201/)
4. **运行Hugo**
在你的站点根目录执行`Hugo`命令进行预览和调试：
    ```shell script
    hugo server --theme=maupassant --buildDrafts
    ```
    (注明：v0.15版本之后，不需要使用`--watch`参数了)。
    浏览器打开：`http://localhost:1313`
5. **部署**
假设你需要部署在`Github Pages`上，首先在Github上创建一个Repository，命名为`coderzh.github.io`（coderzh替换为你的github用户名）。
在站点根目录执行`Hugo`命令生成最终页面：
    ```shell script
    hugo --theme=maupassant --baseUrl="https://coderzh.github.io/"
    ```
    (注意，以上命令不会生成草稿页面，如果未生成任何文章，请去掉头部的`draft=true`再重新生成。)
    如果一切顺利，所有静态页面都会生成到`public`目录，将public目录里所有文件`push`到刚创建的Repository的`master`分支。（注意，如果你刚建的仓库不是空的，需要先`pull`然后再`push`。）
    ```shell script
     cd public
     git init
     git remote add origin https://github.com/coderzh/coderzh.github.io.git
     git add -A
     git commit -m "first commit"
     git push -u origin master
    ```
    浏览器里访问：`https://coderzh.github.io/`（注意：可能由于网络原因部署好之后显示的和预览效果不同，等待一段时间就好了。）
    其它更详细的介绍，你看浏览[Hugo官网](https://gohugo.net/)

## 三：Hugo+Netlify
如果你有自己的服务器和域名的话，可以使用[Netlify](https://app.netlify.com/)作可持续部署，当然Netlify也提供了自己的域名供免费使用。（注意，需要把克隆的主题中的`.git`文件删除，不然在使用Netlify部署的时候会报错。）

1. 将Hugo生成好的站点`push`到远程仓库上去，使用`.gitignore`文件将生成的静态页面文件`public`忽略掉。假设使用的远程仓库是Github，那么就将远程仓库授权给Netlify。
![](https://pic.downk.cc/item/5e567d2f6127cc0713dc236f.png)
![](https://pic.downk.cc/item/5e567def6127cc0713dc4887.png)
2. 部署好之后，点击预览，这个时候使用的是Netlify提供的免费的域名，如果想要使用自己的域名，可以进行更改（注意：国内的域名使用DNS解析是需要进行备案的。）
![](https://pic.downk.cc/item/5e568a526127cc0713de177b.png)
![](https://pic.downk.cc/item/5e568b1e6127cc0713de3321.png)
3. 添加自己的域名
![](https://pic.downk.cc/item/5e568b1e6127cc0713de3321.png)
4. 这样就完成部署了，下次再提交更改到远程仓库的时候，Netlify会自动进行部署。
![](https://pic.downk.cc/item/5e568d9e6127cc0713de7ec2.png)
