---
title:  TCP-IP SOCKET
date: 2022-01-27 10:42:23
tags:
---


TCP/IP（Transmission Control Protocol/Internet Protocol）即传输控制协议/网间协议，是一个工业标准的协议集，它是为广域网（WANs）设计的。

*UDP（User Data Protocol，用户数据报协议）是与TCP相对应的协议。它是属于TCP/IP协议族中的一种。
这里有一张图，表明了这些协议的关系。
![tcp-ip 4层](https://raw.githubusercontent.com/ycchildcoder/markdown/main/20180913104811138)
TCP/IP协议族包括运输层、网络层、链路层。现在你知道TCP/IP与UDP的关系了吧。
Socket在哪里呢？
在图1中，我们没有看到Socket的影子，那么它到底在哪里呢？还是用图来说话，一目了然。
![socket在哪里](https://raw.githubusercontent.com/ycchildcoder/markdown/main/2018091310482621)

**Socket是应用层与TCP/IP协议族通信的中间软件抽象层**，它是一组接口。在设计模式中，Socket其实就是一个门面模式，它把复杂的TCP/IP协议族隐藏在Socket接口后面，对用户来说，一组简单的接口就是全部，让Socket去组织数据，以符合指定的协议。
你会使用它们吗？
前人已经给我们做了好多的事了，网络间的通信也就简单了许多，但毕竟还是有挺多工作要做的。以前听到Socket编程，觉得它是比较高深的编程知识，但是只要弄清Socket编程的工作原理，神秘的面纱也就揭开了。
一个生活中的场景。你要打电话给一个朋友，先拨号，朋友听到电话铃声后提起电话，这时你和你的朋友就建立起了连接，就可以讲话了。等交流结束，挂断电话结束此次交谈。 生活中的场景就解释了这工作原理，也许TCP/IP协议族就是诞生于生活中，这也不一定。
![tcp交互](https://raw.githubusercontent.com/ycchildcoder/markdown/main/20180913104844708)*

先从服务器端说起。服务器端先初始化Socket，然后与端口绑定(bind)，对端口进行监听(listen)，调用accept阻塞，等待客户端连接。在这时如果有个客户端初始化一个Socket，然后连接服务器(connect)，如果连接成功，这时客户端与服务器端的连接就建立了。客户端发送数据请求，服务器端接收请求并处理请求，然后把回应数据发送给客户端，客户端读取数据，最后关闭连接，一次交互结束。