---
title:  linux ss 命令
date: 2022-01-26 15:42:23
tags:
---

**ss**

用来显示处于活动状态的套接字信息。ss命令可以用来获取socket统计信息，它可以显示和netstat类似的内容。但ss的优势在于它能够显示更多更详细的有关TCP和连接状态的信息，而且比netstat更快速更高效。

当服务器的socket连接数量变得非常大时，无论是使用netstat命令还是直接cat`/proc/net/tcp`，执行速度都会很慢。可能你不会有切身的感受，但请相信我，当服务器维持的连接达到上万个的时候，使用netstat等于浪费 生命，而用ss才是节省时间。

天下武功唯快不破。ss快的秘诀在于，它利用到了TCP协议栈中tcp_diag。tcp_diag是一个用于分析统计的模块，可以获得Linux 内核中第一手的信息，这就确保了ss的快捷高效。当然，如果你的系统中没有tcp_diag，ss也可以正常运行，只是效率会变得稍慢。

 

**选项**

```
-h：显示帮助信息；
-V：显示指令版本信息；
-n：不解析服务名称，以数字方式显示；
-a：显示所有的套接字；
-l：显示处于监听状态的套接字；
-o：显示计时器信息；
-m：显示套接字的内存使用情况；
-p：显示使用套接字的进程信息；
-i：显示内部的TCP信息；
-4：只显示ipv4的套接字；
-6：只显示ipv6的套接字；
-t：只显示tcp套接字；
-u：只显示udp套接字；
-d：只显示DCCP套接字；
-w：仅显示RAW套接字；
-x：仅显示UNIX域套接字。
```
**实例**

**显示TCP连接**

```shell
[root@localhost ~]# ss -t -a
State       Recv-Q Send-Q                            Local Address:Port                                Peer Address:Port   
LISTEN      0      0                                             *:3306                                           *:*       
LISTEN      0      0                                             *:http                                           *:*       
LISTEN      0      0                                             *:ssh                                            *:*       
LISTEN      0      0                                     127.0.0.1:smtp                                           *:*       
ESTAB       0      0                                112.124.15.130:42071                              42.156.166.25:http    
ESTAB       0      0                                112.124.15.130:ssh                              121.229.196.235:33398 
```

**显示 Sockets 摘要**

```shell
[root@localhost ~]# ss -s
Total: 172 (kernel 189)
TCP:   10 (estab 2, closed 4, orphaned 0, synrecv 0, timewait 0/0), ports 5

Transport Total     ip        IPv6
*         189       -         -        
RAW       0         0         0        
UDP       5         5         0        
TCP       6         6         0        
INET      11        11        0        
FRAG      0         0         0   
```

列出当前的established, closed, orphaned and waiting TCP sockets

**查看进程使用的socket**

```shell
[root@localhost ~]# ss -pl
State       Recv-Q Send-Q                                Local Address:Port                                    Peer Address:Port   
LISTEN      0      128                                              :::ssh                                               :::*        users:(("sshd",1292,4))
LISTEN      0      128                                               *:ssh                                                *:*        users:(("sshd",1292,3))
LISTEN      0      128                                       127.0.0.1:ipp                                                *:*        users:(("cupsd",1165,7))
LISTEN      0      128                                             ::1:ipp                                               :::*        users:(("cupsd",1165,6))
LISTEN      0      128                                               *:32957                                              *:*        users:(("rpc.statd",1104,9))
LISTEN      0      128                                              :::57637                                             :::*        users:(("rpc.statd",1104,11))
LISTEN      0      80                                               :::mysql                                             :::*        users:(("mysqld",1528,17))
LISTEN      0      128                                               *:6379                                               *:*        users:(("redis-server",1672,5))
LISTEN      0      128                                              :::6379                                              :::*        users:(("redis-server",1672,4))
LISTEN      0      128                                              :::sunrpc                                            :::*        users:(("rpcbind",1084,11))
LISTEN      0      128                                               *:sunrpc                                             *:*        users:(("rpcbind",1084,8))
LISTEN      0      128                                               *:http                                               *:*        users:(("nginx",1685,13),("nginx",3698,13),("nginx",3699,13))
```


**找出打开套接字/端口应用程序**

```shell
[root@localhost ~]# ss -pl | grep 3306
0      0                            *:3306                          *:*        users:(("mysqld",1718,10))
```

**关于Recv-Q和Send-Q状态**- es

**recv-Q 表示网络接收队列**
**表示收到的数据已经在本地接收缓冲，但是还有多少没有被进程取走，recv()**
如果接收队列Recv-Q一直处于阻塞状态，可能是遭受了拒绝服务 denial-of-service 攻击。

s**end-Q 表示网路发送队列**
**对方没有收到的数据或者说没有Ack的,还是本地缓冲区.**
如果发送队列Send-Q不能很快的清零，可能是有应用向外发送数据包过快，或者是对方接收数据包不够快。

这两个值通常应该为0，如果不为0可能是有问题的。packets在两个队列里都不应该有堆积状态。可接受短暂的非0情况。



对于上边的说法不能说错，但最起码不完全正确，我感觉下边的才是正解，来自：[TCP queue 的一些问题](http://jaseywang.me/2014/07/20/tcp-queue-的一些问题/) 

1.  当 client 通过 connect 向 server 发出 SYN 包时，client 会维护一个 socket 等待队列，而 server 会维护一个 SYN 队列
2.  此时进入半链接的状态，如果 socket 等待队列满了，server 则会丢弃，而 client 也会由此返回 connection time out；只要是 client 没有收到 SYN+ACK，3s 之后，client 会再次发送，如果依然没有收到，9s 之后会继续发送
3.  半连接 syn 队列的长度为 max(64, /proc/sys/net/ipv4/tcp_max_syn_backlog) 决定
4.  当 server 收到 client 的 SYN 包后，会返回 SYN, ACK 的包加以确认，client 的 TCP 协议栈会唤醒 socket 等待队列，发出 connect 调用
5.  client 返回 ACK 的包后，server 会进入一个新的叫 accept 的队列，该队列的长度为 min(backlog, somaxconn)，默认情况下，somaxconn 的值为 128，表示最多有 129 的 ESTAB 的连接等待 accept()，而 backlog 的值则由 [int listen(int sockfd, int backlog)](http://http//linux.die.net/man/2/listen) 中的第二个参数指定，listen 里面的 backlog 的含义请看这里。需要注意的是，[一些 Linux 的发型版本可能存在对 somaxcon 错误 truncating 方式](http://serverfault.com/questions/518862/testifying-rasing-net-core-somaxconn-can-make-a-difference)。
6.  当 accept 队列满了之后，即使 client 继续向 server 发送 ACK 的包，也会不被响应，此时，server 通过 /proc/sys/net/ipv4/tcp_abort_on_overflow 来决定如何返回，0 表示直接丢丢弃该 ACK，1 表示发送 RST 通知 client；相应的，client 则会分别返回 read timeout 或者 connection reset by peer。上面说的只是些理论，如果服务器不及时的调用 accept()，当 queue 满了之后，服务器并不会按照理论所述，不再对 SYN 进行应答，返回 ETIMEDOUT。根据[这篇](http://www.douban.com/note/178129553/)文档的描述，实际情况并非如此，服务器会随机的忽略收到的 SYN，建立起来的连接数可以无限的增加，只不过客户端会遇到延时以及超时的情况。

 

注意，[之前我对 Recv-Q/Send-Q 的理解有些误差](http://jaseywang.me/2012/10/12/netstat-中的-recvsend/)，使用 ss 获取到的 Recv-Q/Send-Q 在 LISTEN 状态以及非 LISTEN 状态所表达的含义是不同的。从 [tcp_diag.c 源码](http://www.cs.fsu.edu/~baker/devices/lxr/http/source/linux/net/ipv4/tcp_diag.c?v=2.6.25)中可以看到二者的区别:

 

**LISTEN 状态: Recv-Q 表示的当前等待服务端调用 accept 完成三次握手的 listen backlog 数值，也就是说，当客户端通过 connect() 去连接正在 listen() 的服务端时，这些连接会一直处于这个 queue 里面直到被服务端 accept()；Send-Q 表示的则是最大的 listen backlog 数值，这就就是上面提到的 min(backlog, somaxconn) 的值。**
**其余状态: 非 LISTEN 状态之前理解的没有问题。Recv-Q 表示 receive queue 中的 bytes 数量；Send-Q 表示 send queue 中的 bytes 数值。**