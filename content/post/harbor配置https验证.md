+++
title="Harbor配置https验证"
date=2020-04-15T11:02:38+08:00
categories=["Harbor"]
tags=["Harbor"]
toc=false
+++

### 一：安装Harbor
#### 1. 安装的先决条件
**硬件**：

|资源|最低要求|推荐的|
|:--:|:--:|:--:|
|CPU|2|4|
|内存|4GB|8GB|
|磁盘|40GB|160GB|

**软件**：

|软件|版本|描述|
|:--:|:--:|:--:|
|Docker Engine|Version 17.06.0-ce+ or higher|[官方文档](https://docs.docker.com/engine/installation/)|
|Docker Compose|Version 1.18.0 or higher|[官方文档](https://docs.docker.com/compose/install/)
|Openssl|最好是最新的|用于生成Harbor的自签证书和自签私钥|

**网络端口**：

|端口|协议|描述|
|:--:|:--:|:--:|
|80|HTTP|接受HTTP请求|
|443|HTTPS|接受HTTPS请求|
|4443|HTTPS|仅在启用公证人的情况下才需要|
>这里需要注意的是，国内大部分服务器是不开启80和443端口的，所以这里需要将默认的80和443端口切换为其他端口才能使用

#### 2. 安装Harbor
官方发布了在线安装和离线安装的两个版本，官方文档的说明是基于离线安装版本的。在自己的服务器上搭建Harbor的时候，个人建议使用在线安装的版本，因为安装包比较小，容易下载使用。
+ [下载页面](https://github.com/goharbor/harbor/releases)
+ 在线安装程序：harbor-online-installer-version.tgz
+ 离线安装程序：harbor-offline-installer-version.tgz
+ [官方文档](https://goharbor.io/docs/1.10/install-config/download-installer/)

### 二：配置HTTPS协议
#### 1. 使用Openssl进行自签证书
>这里需要注意的是，由于是自签证书，简单来说就是没有公信力的，是不被网络认可的，建议在主机数量少的局域网中使用，适用于测试环境。
+ 解决方案一：由于自签证书在网络中是不被认可的，所以需要将生成的证书和私钥导入局域网的其他主机中去，就像官方文档的做法一样，将证书和私钥导入docker中去
+ 解决方案二：修改docker的配置文件daemon.json（/etc/docker/daemon.json），添加`insecure-registries`参数，将安装Harbor的主机IP地址添加进去，使得Docker对该主机不进行HTTPS认证。

#### 2.使用有公信力的第三方机构的证书
1. 申请一个域名：由于只有拥有域名才可以申请第三方机构的证书。
2. 拥有一个是公有IP的主机：虽然域名可以解析到私有IP，但是需要修改DNS服务器，绕到后面还是需要一个公有IP
3. 按照域名服务商的要求，对域名进行解析
4. 下载申请好的证书到本地，然后修改Harbor的配置
    > 这里需要注意的是，这里申请的证书，是不会进行自动续签的，所以证书到期后需要手动续签

### 参考链接：
+ https://goharbor.io/docs/1.10/install-config/
+ https://www.cnblogs.com/sanduzxcvbnm/p/11956347.html
