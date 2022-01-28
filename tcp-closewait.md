---
title:  TCP CLOSE_WAIT 太多问题
date: 2022-01-26 15:42:23
tags:
---



### TCP四次挥手

我们来看看 **TCP** 的四次挥手是怎么样的流程：



<img src="https://raw.githubusercontent.com/ycchildcoder/markdown/main/167928e06180fc5e%7Etplv-t2oaga2asx-watermark.awebp" alt="image-20181208175427046" style="zoom: 50%;" />



**图五：TCP四次挥手**

用中文来描述下这个过程：

Client: `服务端大哥，我事情都干完了，准备撤了`，这里对应的就是客户端发了一个**FIN**

Server：`知道了，但是你等等我，我还要收收尾`，这里对应的就是服务端收到 **FIN** 后回应的 **ACK**

经过上面两步之后，服务端就会处于 **CLOSE_WAIT** 状态。过了一段时间 **Server** 收尾完了

Server：`小弟，哥哥我做完了，撤吧`，服务端发送了**FIN**

Client：`大哥，再见啊`，这里是客户端对服务端的一个 **ACK**

到此服务端就可以跑路了，但是客户端还不行。为什么呢？客户端还必须等待 **2MSL** 个时间，这里为什么客户端还不能直接跑路呢？主要是为了防止发送出去的 **ACK** 服务端没有收到，服务端重发 **FIN** 再次来询问，如果客户端发完就跑路了，那么服务端重发的时候就没人理他了。这个等待的时间长度也很讲究。

<u>**Maximum Segment Lifetime** 报文最大生存时间，它是任何报文在网络上存在的最长时间，超过这个时间报文将被丢弃</u>

这里一定不要被图里的 **client／server** 和项目里的客户端服务器端混淆，你只要记住：主动关闭的一方发出 **FIN** 包（Client），被动关闭（Server）的一方响应 **ACK** 包，此时，被动关闭的一方就进入了 **CLOSE_WAIT** 状态。如果一切正常，稍后被动关闭的一方也会发出 **FIN** 包，然后迁移到 **LAST_ACK** 状态。

既然是这样， **TCP** 抓包分析下：

```
/go # tcpdump -n port 3306
# 发生了 3次握手
11:38:15.679863 IP 172.18.0.5.38822 > 172.18.0.3.3306: Flags [S], seq 4065722321, win 29200, options [mss 1460,sackOK,TS val 2997352 ecr 0,nop,wscale 7], length 0
11:38:15.679923 IP 172.18.0.3.3306 > 172.18.0.5.38822: Flags [S.], seq 780487619, ack 4065722322, win 28960, options [mss 1460,sackOK,TS val 2997352 ecr 2997352,nop,wscale 7], length 0
11:38:15.679936 IP 172.18.0.5.38822 > 172.18.0.3.3306: Flags [.], ack 1, win 229, options [nop,nop,TS val 2997352 ecr 2997352], length 0

# mysql 主动断开链接
11:38:45.693382 IP 172.18.0.3.3306 > 172.18.0.5.38822: Flags [F.], seq 123, ack 144, win 227, options [nop,nop,TS val 3000355 ecr 2997359], length 0 # MySQL负载均衡器发送fin包给我
11:38:45.740958 IP 172.18.0.5.38822 > 172.18.0.3.3306: Flags [.], ack 124, win 229, options [nop,nop,TS val 3000360 ecr 3000355], length 0 # 我回复ack给它

... ... # 本来还需要我发送fin给他，但是我没有发，所以出现了close_wait。那这是什么缘故呢？
```

结合上面的信息，我用文字说明下：**MySQL负载均衡器** 给我的服务发送 **FIN** 包，我进行了响应，此时我进入了 **CLOSE_WAIT** 状态，但是后续作为被动关闭方的我，并没有发送 **FIN**，导致我服务端一直处于 **CLOSE_WAIT** 状态，无法最终进入 **CLOSED** 状态。

-   **大量`CLOSE_WAIT`有什么危害?** `CLOSE_WAIT`状态不会自己消失,除非对应的应用进程死掉,不会消失就意味着一直占用服务器资源,端口总数又只有65535,因此这里的服务器作为连接的发起者就会造成大量端口被占用,一旦占用完就导致后面的请求都发不出去,也就是一开始图上另一个项目发请求出现的`Address already in use (Bind failed)`错误.
-   被动方什么情况下`FIN`包会发送失败?
    -   **程序问题**：如果代码层面忘记了 close 相应的 socket 连接，那么自然不会发出 FIN 包，从而导致 CLOSE_WAIT 累积；或者代码不严谨，出现死循环之类的问题，导致即便后面写了 close 也永远执行不到。
    -   **响应太慢或者超时设置过小**：如果连接双方不和谐，一方不耐烦直接 timeout，另一方却还在忙于耗时逻辑，就会导致 close 被延后。响应太慢是首要问题，不过换个角度看，也可能是 timeout 设置过小。
    -   BACKLOG 太大：此处的 backlog 不是 syn backlog，而是 accept 的 backlog，如果 backlog 太大的话，设想突然遭遇大访问量的话，即便响应速度不慢，也可能出现来不及消费的情况，导致多余的请求还在队列里就被对方关闭了。

如果你通过「netstat -ant」或者「ss -ant」命令发现了很多 CLOSE_WAIT 连接，请注意结果中的「Recv-Q」和「Local Address」字段，通常「Recv-Q」会不为空，它表示应用还没来得及接收数据，而「Local Address」表示哪个地址和端口有问题，我们可以通过「lsof -i:<PORT>」来确认端口对应运行的是什么程序以及它的进程号是多少。

如果是我们自己写的一些程序，比如用 HttpClient 自定义的蜘蛛，那么八九不离十是程序问题，如果是一些使用广泛的程序，比如 Tomcat 之类的，那么更可能是响应速度太慢或者 timeout 设置太小或者 BACKLOG 设置过大导致的故障。

此外还有一点需要说明：按照前面图例所示，当被动关闭的一方处于 CLOSE_WAIT 状态时，主动关闭的一方处于 FIN_WAIT2 状态。 那么为什么我们总听说 CLOSE_WAIT 状态过多的故障，但是却相对少听说 FIN_WAIT2 状态过多的故障呢？这是因为 Linux 有一个「tcp_fin_timeout」设置，控制了 FIN_WAIT2 的最大生命周期。坏消息是 CLOSE_WAIT 没有类似的设置，如果不重启进程，那么 CLOSE_WAIT 状态很可能会永远持续下去；好消息是如果 socket 开启了 [keepalive](http://www.tldp.org/HOWTO/html_single/TCP-Keepalive-HOWTO/) 机制，那么可以通过相应的设置来清理无效连接，不过 keepalive 是治标不治本的方法，还是应该找到问题的症结才对。

本来想多写点的，但是着急回家，就写到这吧，推荐两个案例：

-   [PHP升级导致系统负载过高问题分析](http://mp.weixin.qq.com/s?__biz=MjM5NzUwNDA5MA==&mid=200667929&idx=1&sn=67cada895ac100115fded319b6b23a21&3rd=MzA3MDU4NTYzMw==&scene=6#rd)
-   [又见CLOSE_WAIT](http://mp.weixin.qq.com/s?__biz=MzI4MjA4ODU0Ng==&mid=402163560&idx=1&sn=5269044286ce1d142cca1b5fed3efab1&3rd=MzA3MDU4NTYzMw==&scene=6#rd)

写得都比我好，建议大家仔细阅读。