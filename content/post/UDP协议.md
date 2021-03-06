+++
title="（八）UDP协议"
date=2020-03-20T19:34:59+08:00
categories=["网络协议"]
toc=false
+++

### UDP和TCP有哪些区别？
UDP是面向无连接的，TCP是面向连接的。

什么叫面向连接，什么叫无连接呢？在互通之前，面向连接的协议会先建立连接。例如，TCP会三次握手，而UDP不会。为什么要建立连接呢？TCP三次握手，UDP也可以发三个包呀，之间有什么区别呢？

**所谓的建立连接，是为了在客户端和服务端维护连接，而建立一定的数据结构来维护双方交互的状态，用这样的数据结构来保证所谓的面向连接的特性**。例如，**TCP提供可靠交付**。通过TCP连接传输的数据，无差错、不丢失、不重复、并且按序到达。我们都知道IP包是没有任何可靠性保证的，一旦发出去，出现问题都只能随它去。但是TCP号称能做到维护连接。而**UDP继承了IP包的特性，不保证不丢失，不保证按顺序到达**。

**TCP是面向字节流的**。即发送的时候是一个流，没头没尾。但是IP包并不是一个流，而是一个个的IP包，之所以变成流，这也是TCP自己的状态维护所做的。而**UDP继承了IP的特性，基于数据包的，一个一个的发，一个一个的收**。

还有**TCP是可以拥塞控制的**。它意识到包丢弃了或者网络的环境不好了，就会根据情况调整自己的行为，看看是不是发快了，要不要慢点。而**UDP就不会，应用让我发，我就发，管它是不是发快了**。

因而**TCP其实是一个有状态服务**，通俗的讲就算有脑子的，里面精确的记着发送了没有，接收到没有，发送到哪个了，应该接收哪个了，错一点都不行。而**UDP则是无状态服务**。通俗的说是没脑子的，天真无邪的，发出去就发出去了。

我们可以这样比喻，如果MAC层定义了本地局域网的传输行为，IP层定义了整个网络端到端的传输行为这两层基本定义了这样的基因：网络传输是以包为单位的，二层叫帧，网络层叫包，传输层叫段。我们笼统的称为包。包单独传输，自行选路，在不同的设备封装解封装保证到达。基于这个基因，生下来的孩子UDP完全继承了这些特性，几乎没有自己的思想。

### UDP包头是什么样的？
当发送的包到达目标机器后，发现MAC地址匹配，于是就取下来，将剩下的包传给处理IP层的代码。把IP头取下来，发现目标IP匹配，接下来呢？这里面的数据包是给谁呢？

发送的时候，我知道我发的是一个UDP包，收到的那台机器怎么知道呢？所以在IP头里面有个8位协议，会存放数据到底是TCP还是UDP。知道是UDP还是TCP之后，就可以把数据解析出来，那解析出来的数据给谁呢？

无论应用程序写的使用TCP传数据，还是UDP传数据，都要监听一个端口。正是这个端口，用来区分应用程序。
![](https://pic.downk.cc/item/5e7975149dbe9d88c5ac1de9.jpg)

### UDP的三大特点
1. 沟通简单：不需要大量的数据结构、处理逻辑和包头字段
2. 轻信他人：它不会建立连接，虽然有端口号，但是监听在这个地方，谁都可以传给他数据，他也可以传给任何人数据，甚至可以同时传给多个人数据。
3. 不同改变：它不会根据网络的情况进行发包的拥塞控制，无论网络包丢成什么样，它该怎么发还是怎么发。

### UDP的三大使用场景
1. **需要资源少，在网络比较好的内网，或者对于丢包不敏感的应用**。像DHCP和PXE中下载使用的TFTP都是基于UDP协议的。
2. **不需要一对一沟通，建立连接，而是可以广播的应用**。UDP的不面向连接的功能，使得可以承载广播或者多播的协议。
3. **需要处理速度快，时延低，可以容忍少数丢包，但是要求即便网络拥塞，也毫不退缩，以往无前的时候**。


