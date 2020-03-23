+++
title="（一）ifconfig"
date=2020-03-13T16:09:14+08:00
categories=["网络协议"]
description="查看宿主机的ip地址"
toc=false
+++

## ifconfig（net-tools）和ip addr（iproute2）
运行一下ip addr:
```shell script
[root@test]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:86:84:7e brd ff:ff:ff:ff:ff:ff
    inet 192.168.189.137/24 brd 192.168.189.255 scope global noprefixroute dynamic ens33
       valid_lft 1015sec preferred_lft 1015sec
    inet6 fe80::c495:8a34:d3eb:84a2/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether 52:54:00:76:2a:f5 brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
4: virbr0-nic: <BROADCAST,MULTICAST> mtu 1500 qdisc pfifo_fast master virbr0 state DOWN group default qlen 1000
    link/ether 52:54:00:76:2a:f5 brd ff:ff:ff:ff:ff:ff
5: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:77:62:31:c0 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
```
这个命令显示了这台机器上所有的网卡。大部分网卡都会有一个IP地址，当然这不是必须的。也会存在没有IP地址的情况。

### IPv4的分类
![](https://pic.downk.cc/item/5e6b4af9e83c3a1e3a8f08c4.png)
在网络地址中，至少在当时设计的时候，对于A、B、C类主要两部分，前面一部分是网络号，后面一部分是主机号。这很好理解，大家都是六单元1001号，但我是A小区的，你是B小区的。

下面的表格详细的展示了A、B、C三类地址所能包含的主机的数量：
![](https://pic.downk.cc/item/5e6b4df8e83c3a1e3a910177.png)

这里就出现了尴尬的事情，C类地址包含的最大主机数量太少，而B类地址能包含的最大主机数量又太多了。于是就有了一个折中的方式叫做**无类型域间选路**。

### 无类型域间选路（CIDR）
这种方式打破了原来设计的几类地址的做法，将32位的IP地址一分为二，前面是**网络号**，后面是**主机号**。我们观察一下上面的IP地址`192.168.189.137/24`，这个IP地址中有一个斜杠，斜杠后面有个数字24。这种地址表示形式就是CIDR。后面24的意思是，32位中，前24位是网络号，后8位是主机号。

伴随着CIDR存在的，一个是**广播地址**，192.168.189.255。如果发送这个地址，所有192.168.189网络里面的机器都可以收到。另外一个是**子网掩码**，255.255.255.0。

将子网掩码和IP地址进行AND运算，就可以得到**网络号**。将子网掩码取反和IP地址进行或运算，就可以得到**广播地址**。

### 公有IP地址和私有IP地址
![](https://pic.downk.cc/item/5e6b7419e83c3a1e3aa420c5.png)
表格中的192.168.0.x是最常用的私有IP地址。不需要将十进制转为二进制，就能明显看出192.168.0是网络号，后面是主机号。而整个网络里面的第一个地址192.168.0.1，往往就是你这个私有网络的出口地址。例如，你家的电脑连接Wi-Fi，Wi-Fi路由器的地址就是192.168.0.1，而192.168.0.255就是广播地址。一旦发送这个地址，整个192.168.0网络里面的所有机器都能收到。

在IP地址的后面还有个scope，对于ens33这张网卡来讲，是global，说明这张网卡是可以对外的，可以接收来自各个地方的包。对于lo来讲，是host，说明这张网卡仅仅可以供本机相互通信。

lo全称是**loopback**，又称**环回接口**，往往会被分配到127.0.0.1这个地址。这个地址用于本机通信，经过内核处理后直接返回，不会在任何网络中出现。

### MAC地址
在IP地址的上一行是link/ether 00:0c:29:86:84:7e brd ff:ff:ff:ff:ff:ff，这个被称为**MAC地址**，是一个网卡的物理地址，用十六进制，6个byte表示。

MAC地址号称全局唯一，不会有两个网卡有相同的MAC地址，而且网卡自产生出来，就带着这个地址。很多人看到这会想，既然这样，整个互联网的通信，全部用MAC地址好了，只要知道了对方的MAC地址，就可以把信息传过去。

这样当然不行的。**一个网络包要从一个地方传到另一个地方，除了要有确定的地址，还需要有定位功能**。而有门牌号属性的IP地址，才是有远程定位功能的。**MAC地址更像是身份证，是一个唯一的标识**。它的唯一性设计是为了组网的时候，不同的网卡放在一个网络里面的时候，可以不用担心冲突。从硬件角度，保证不同的网卡有不同的标识。

MAC地址是有一定定位功能的，只不过范围非常有限。例如，你可以通过IP地址找到A小区1幢1单元101室，但是依然找不到某个具体的人，这个时候可以大声喊身份证XXXX的是哪一位，那个人听到了就会回答。但是如果你在B小区到处喊身份证XXXX的是哪位，那个人不在那里，当然不会回答。

所以，MAC地址的通信范围比较小，局限在一个子网里面。例如，从192.168.0.2/24访问192.168.0.3/24是可以用MAC地址的。一旦跨子网，即从192.168.0.2/24到192.168.1.2/24，MAC地址就不行了，需要IP地址起作用了。

### 网络设备的状态标识
解析完了MAC地址，我们再来看<BROADCAST,MULTICAST,UP,LOWER_UP>是干什么的？这个叫做**net_device flags，网络设备的状态标识**。
+ UP：表示网卡处于启动的状态
+ BROADCAST：表示这个网卡有关播地址，可以发送广播包
+ MULTICAST：表示网卡可以发送多播包
+ LOWER_UP：表示L1是启动的，即网线是插着的
+ mtu 1500：最大输出单元为1500，是以太网的默认值

MTU是二层MAC层的概念。MAC层有MAC的头，以太网规定连MAC头带正文合起来，不允许超过1500的字节。正文里面有IP的头、TCP的头、HTTP的头。如果放不下，就需要分片来传输。

+ qdisc pfifo_fast：qdisc全称是**queueing discipline**，也叫**排队规则**。内核如果需要通过某个网络接口发送数据包，它都需要按照这个接口配置qdisc把数据包加入队列。
   - pfifo，最简单的排队规则，它不对进入的数据包做任何处理，数据包采用先进先出的方式通过队列
   - pfifo_fast，它的队列包括三个波段（band）。在每个波段里面，使用先进先出规则。三个波段的优先级也不相同。band 0的优先级最高，band 2的最低。如果band 0里面有数据包，系统就不会处理band 1里面的数据包，band 1和band 2之间也是一样。
   <br/>数据包是按照服务类型（**Type of Service，TOS**）被分配到三个波段里面的。TOS是IP头里面的一个字段，代表了当前的包是高优先级的，还是低优先级的。


