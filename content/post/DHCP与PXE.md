+++
title="（二）DHCP与PXE"
date=2020-03-14T09:01:16+08:00
categories=["网络协议"]
description="IP是怎么来的，又是怎么没的"
toc=false
+++


### 如何配置IP地址？
可以使用ifconfig，也可以使用ip addr。设置好了以后，用这两个命令，将网卡up一下，就可以开始工作了。
+ 使用net-tools
    ```shell script
    sudo ifconfig eth1 10.0.0.1/24
    sudo ifconfig eth1 up
    ```
+ 使用iproute2
    ```shell script
    sudo ip addr add 10.0.0.1/24 dev eth1
    sudo ip link set up eth1
    ```
你可能会问，自己配置这个自由度太大了吧。是不是配什么IP地址都可以？如果配置一个和谁都不搭边的地址呢？例如，旁边的机器都是192.168.1.x，我非得配置一个16.158.23.6，会出现什么现象呢？

不会出现任何现象，就是包发不出去，为什么发不出去呢？

我们知道，**只要是在网络上跑的包，都是完整的，可以有下层没上层，绝对不可能有上层没下层**。所以，虽然有目标IP地址192.168.1.6，但是没有目标MAC地址。自己的MAC地址自己知道，但是目标MAC填什么呢？是不是填192.168.1.6这台机器的MAC地址呢？

当然不是。Linux会线判断，要去的这个地址和我是同一个网段的吗，或者和我的网卡是同一个网段的吗？只有是一个网段的，它才会发送ARP请求，获取MAC地址。如果发现不是呢？

<b>Linux默认的逻辑是，如果这是一个跨网段的调用(源地址和目标地址的网络号不同)，它便不会直接将包发送到网络上，而是企图将包发送到网关。</b>

如果你配置了网关的话，Linux会获取网关的MAC地址，然后将包发出去。对于192.168.1.6这台机器来讲，虽然路过它家门的这个包，目标IP是它。但是MAC地址不是它的，所以它的网卡是不会把包收进去的。如果没有配置网关呢？那包压根就发不出去。如果将192.168.1.6配置为网关呢？不可能，因为网关要和当前的网络至少一个网卡是同一网段的。所以16.158.23.6的网关不可能是192.168.1.6。

通常配置都是使用配置文件的。
```shell script
## /etc/sysconfig/network-scripts
## ifcfg-lo
DEVICE=lo
IPADDR=127.0.0.1
NETMASK=255.0.0.0
NETWORK=127.0.0.0
BROADCAST=127.255.255.255
ONBOOT=yes
NAME=loopback
```
**不同系统的配置文件格式不同，但无非就是CIDR、子网掩码、广播地址和网关地址**

### 动态主机配置协议（DHCP）
如果是数据中心里面的服务器，IP一旦配置好，基本不会变，这就相当于买房自己装修。DHCP的方式就相当于租房。你不用装修，都是帮你配置好的。你暂时用一下，用完退租就可以了。

#### 解析DHCP的工作方式
1. **DHCP Discover**
新来的机器使用IP地址0.0.0.0发送一个广播包，目的IP地址为255.255.255.255。广播包封装了UDP，UDP封装了BOOTP。其实DHCP是BOOTP的增强版，但是如果你去抓包的话，很可能看到的名称还是BOOTP协议。
<br/>在这个广播包里面，新人大声喊：我是新来的（Boot request），我的MAC地址是这个，我还没有IP，谁能给租给我个IP地址！
<br/>格式就像这样：
![](https://pic.downk.cc/item/5e6ee106e83c3a1e3a3b4b05.png)
<br/>如果一个网络管理员在网络里面配置了**DHCP Server**的话，他就相当于这些IP的管理员。他立刻能知道来了一个“新人”。这个时候，我们可以体会MAC地址唯一的重要性了。当一台机器带着自己的MAC地址加入一个网络的时候，MAC是它唯一的身份，如果连这个都重复了，就没办法配置了。
<br/>只有MAC唯一，IP管理员才能知道这是一个新人，需要租给它一个IP地址，这个过程我们称为**DHCP Offer**。同时DHCP Server为此客户保留为它提供的IP地址，从而不会为其他DHCP客户端分配此IP地址。

2. **DHCP Offer**
DHCP Offer的格式就像这样，里面有给新人分配的地址。
![](https://pic.downk.cc/item/5e6ee2b3e83c3a1e3a3c2631.png)
DHCP Server仍然使用关播地址作为目的地址，因此，此时请求分配IP的新人还没有自己的IP。DHCP Server回复说，我分配了一个可用的IP给你，你看如何？除此之外，服务器还发送了子网掩码、网关和IP地址租用等信息。
<br/>如果有多个DHCP Server，这台新机器会收到多个IP地址，它会选择其中一个DHCP Offer，一般是最先到达的那个，并且会向网络发送一个DHCP Request广播数据包，包含客户端的MAC地址、接受租约中的IP地址、提供此租约的DHCP服务器地址等，并告诉所有DHCP Server它将接受哪一台服务器提供的IP地址，告诉其他DHCP服务器，并请求撤销它们提供的IP地址，以便提供给下一个IP租用请求者。

3. **DHCP Request**
其格式如下：
![](https://pic.downk.cc/item/5e6ee4eae83c3a1e3a3de65b.png)
此时，由于还没有得到DHCP Server的最后确认，客户端仍然使用0.0.0.0为源IP地址、255.255.255.255为目标地址进行广播。在BOOTP里面，接受某个DHCP Server分配的IP。
<br/>当DHCP Server接收到客户机的DHCP Request之后，会广播返回客户机一个DHCP ACK消息包，表明已经接受客户机的选择，并将这一IP地址的合法租用信息和其他的配置信息都放入该广播包，发给客户机。

4. **DHCP ACK**
其格式如下：
![](https://pic.downk.cc/item/5e6ee637e83c3a1e3a3eb54a.png)

### IP地址的收回和续租
客户机会在租期过去50%的时候，直接向为其提供IP地址的DHCP Server发送DHCP Request消息包。客户机接收到该服务器回应的DHCP ACK消息包，会根据包中所提供的新的租期以及其他已经更新的TCP/IP参数，更新自己的配置。这样，IP租用更新就完成了。

### 预启动执行环境（PXE）
在数据中心中，管理员可能一下拿到几百台空的机器，所以管理员不仅希望可用自动分配IP地址，还要自动安装系统。装好系统之后自动分配IP地址，直接启动就可以使用。

这个过程和操作系统启动的过程有点像。首先，启动BIOS，这是一个特别小的小系统，只能干特别小的一件事情，就是读取硬盘的MBR启动扇区，将GRUB启动起来；然后将权力交给GRUB，GRUB加载内核、加载作为文件根系统的initramfs文件；然后将权力交给内核；最后内核启动，初始化整个操作系统。

那么安装操作系统的过程，只能插在BIOS启动之后了。因为没安装系统之前，连启动扇区都没有。因而这个过程叫做**预启动执行环境（Pre-boot Execution Environment）**，简称**PXE**。

PXE协议分为客户端和服务器端，由于没有操作系统，只能先把客户端放在BIOS里面。当计算机启动时，BIOS把PXE客户端调入内存里面，就可用连接到服务端做一些操作了。

首先，PXE客户端自己也需要有个IP地址。因为PXE的客户端启动起来，就可用发送一个DHCP请求，让DHCP Server给它分配一个地址。PXE客户端有了自己的地址，那它怎么知道PXE服务器在哪里呢？好在DHCP Server除了分配IP地址以外，还可以做一些其他的事情。这里有一个DHCP Server的一个样例配置：
```shell script
ddns-update-style interim;
ignore client-updates;
allow booting;
allow bootp;
subnet 192.168.1.0 netmask 255.255.255.0
{
    ## 默认路由（网关）
    option routers 192.168.1.1;
    ## 默认子网掩码
    option subnet-mask 255.255.255.0;
    option time-offset -18000;
    ## 默认续租时间
    default-lease-time 21600;
    ## 最大续租时间
    max-lease-time 43200;
    ## 地址池
    range dymaic-bbotp 192.168.1.240 192.168.1.250;
    filename "pxelinux.0";
    net-server 192.168.1.180;
}
```
按照上面的原理，默认的DHCP Server是需要配置的，无非是我们配置IP的时候所需要的IP地址段、子网掩码、网关地址、地址、租期等。如果想使用PXE，则需要配置net-server，指向PXE服务器的地址，另外要配置初始启动文件filenmae。这样PXE客户端启动之后，发送DHCP请求之后，除了能得到一个IP地址，还可以知道PXE服务器在哪里，也可以指定如何从PXE服务器上下载某个文件，去初始化操作系统。

### 解析PXE的工作过程
1. 启动PXE客户端，通过DHCP协议向DHCP Server发出请求
2. DHCP Server租给它一个IP地址，同时也给它PXE服务器的地址、启动文件（上面的pxelinux.0）
3. 客户端使用TFTP协议向PXE服务器请求下载这个文件（所以PXE服务器上，往往还需要一个TFTP服务器）
4. TFTP服务器将这个文件传给它
5. PXE客户端接收到这个文件后，开始执行这个文件。这个文件会指示PXE客户端，向TFTP服务器请求计算机的配置信息`pxelinux.cfg`。TFTFP服务器会给PXE客户端一个配置文件，里面会说内核在哪里，initramfs在哪里。PXE客户端会请求这些文件
6. 启动Linux内核。
![](https://pic.downk.cc/item/5e6efa95e83c3a1e3a460aff.jpg)


### 其他链接
[如何配置DHCP Server](https://blog.csdn.net/qq_38228830/article/details/82119267?depth_1-utm_source=distribute.pc_relevant.none-task&utm_source=distribute.pc_relevant.none-task)
[如何配置PXE](https://www.cnblogs.com/mython/p/10919355.html)


