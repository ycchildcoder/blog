---
title:  netstat 
date: 2022-01-27 10:42:23
tags:
---



# **简介**

Netstat 命令用于显示各种网络相关信息，如网络连接，路由表，接口状态 (Interface Statistics)，masquerade 连接，多播成员 (Multicast Memberships) 等等。

# **输出信息含义**

执行netstat后，其输出结果为

```shell
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address Foreign Address State
tcp 0 2 210.34.6.89:telnet 210.34.6.96:2873 ESTABLISHED
tcp 296 0 210.34.6.89:1165 210.34.6.84:netbios-ssn ESTABLISHED
tcp 0 0 localhost.localdom:9001 localhost.localdom:1162 ESTABLISHED
tcp 0 0 localhost.localdom:1162 localhost.localdom:9001 ESTABLISHED
tcp 0 80 210.34.6.89:1161 210.34.6.10:netbios-ssn CLOSE

Active UNIX domain sockets (w/o servers)
Proto RefCnt Flags Type State I-Node Path
unix 1 [ ] STREAM CONNECTED 16178 @000000dd
unix 1 [ ] STREAM CONNECTED 16176 @000000dc
unix 9 [ ] DGRAM 5292 /dev/log
unix 1 [ ] STREAM CONNECTED 16182 @000000df
```

从整体上看，netstat的输出结果可以分为两个部分：

一个是`Active Internet connections`，称为有源TCP连接，其中`Recv-Q`和`Send-Q`指%0A的是接收队列和发送队列。这些数字一般都应该是0。如**果不是则表示软件包正在队列中堆积**。这种情况只能在非常少的情况见到。

另一个是`Active UNIX domain sockets`，称为有源Unix域套接口(和网络套接字一样，但是只能用于本机通信，性能可以提高一倍)。

-   Proto显示连接使用的协议
-   RefCnt表示连接到本套接口上的进程号
-   Types显示套接口的类型
-   Path表示连接到套接口的其它进程使用的路径名。
-   Local Address代表本机IP地址和打开的端口号
-   Foreign Address是远程计算机IP地址和端口号
-   State表明当前的连接状态
     \>  LISTENING 时表示处于侦听状态，就是说该端口是开放的，等待连接，但还没有被连接
     \>  ESTABLISHED 的意思是建立连接。表示两台机器正在通信。
     \>  CLOSE_WAIT 对方主动关闭连接或者网络异常导致连接中断,TIME_WAIT 我方主动调用close()断开连接，收到对方确认后状态变为TIME_WAIT

# **常见参数**

-a (all)显示所有选项，默认不显示LISTEN相关
-t (tcp)仅显示tcp相关选项
-u (udp)仅显示udp相关选项
-n 拒绝显示别名，能显示数字的全部转化成数字。
-l 仅列出有在 Listen (监听) 的服務状态

-p 显示建立相关链接的程序名
-r 显示路由信息，路由表
-e 显示扩展信息，例如uid等
-s 按各个协议进行统计
-c 每隔一个固定时间，执行该netstat命令。

提示：LISTEN和LISTENING的状态只有用-a或者-l才能看到

# **实用命令实例**

##  

## **1. 列出所有端口 (包括监听和未监听的)**

列出相关连接的程序名都增加-p 参数

 <u>**列出所有端口 netstat -a**</u>

```shell
# netstat -a | more
 Active Internet connections (servers and established)
 Proto Recv-Q Send-Q Local Address           Foreign Address         State
 tcp        0      0 localhost:30037         *:*                     LISTEN
 udp        0      0 *:bootpc                *:*
 
Active UNIX domain sockets (servers and established)
 Proto RefCnt Flags       Type       State         I-Node   Path
 unix  2      [ ACC ]     STREAM     LISTENING     6135     /tmp/.X11-unix/X0
 unix  2      [ ACC ]     STREAM     LISTENING     5140     /var/run/acpid.socket
```

 <u>**列出所有 tcp 端口 netstat -at**</u>

```shell
# netstat -at
 Active Internet connections (servers and established)
 Proto Recv-Q Send-Q Local Address           Foreign Address         State
 tcp        0      0 localhost:30037         *:*                     LISTEN
 tcp        0      0 localhost:ipp           *:*                     LISTEN
 tcp        0      0 *:smtp                  *:*                     LISTEN
 tcp6       0      0 localhost:ipp           [::]:*                  LISTEN
```

 <u>**列出所有 udp 端口 netstat -au**</u>

```shell
# netstat -au
 Active Internet connections (servers and established)
 Proto Recv-Q Send-Q Local Address           Foreign Address         State
 udp        0      0 *:bootpc                *:*
 udp        0      0 *:49119                 *:*
 udp        0      0 *:mdns                  *:*
```

## **2. 列出所有处于监听状态的 Sockets**

列出相关连接的程序名都增加-p 参数

 **只显示监听端口 netstat -l**

```shell
# netstat -l
 Active Internet connections (only servers)
 Proto Recv-Q Send-Q Local Address           Foreign Address         State
 tcp        0      0 localhost:ipp           *:*                     LISTEN
 tcp6       0      0 localhost:ipp           [::]:*                  LISTEN
 udp        0      0 *:49119                 *:*
```

 **只列出所有监听 tcp 端口 netstat -lt**

```shell
# netstat -lt
 Active Internet connections (only servers)
 Proto Recv-Q Send-Q Local Address           Foreign Address         State
 tcp        0      0 localhost:30037         *:*                     LISTEN
 tcp        0      0 *:smtp                  *:*                     LISTEN
 tcp6       0      0 localhost:ipp           [::]:*                  LISTEN
```

 **只列出所有监听 udp 端口 netstat -lu**

```shell
# netstat -lu
 Active Internet connections (only servers)
 Proto Recv-Q Send-Q Local Address           Foreign Address         State
 udp        0      0 *:49119                 *:*
 udp        0      0 *:mdns                  *:*
```

 **只列出所有监听 UNIX 端口 netstat -lx**

```shell
# netstat -lx
 Active UNIX domain sockets (only servers)
 Proto RefCnt Flags       Type       State         I-Node   Path
 unix  2      [ ACC ]     STREAM     LISTENING     6294     private/maildrop
 unix  2      [ ACC ]     STREAM     LISTENING     6203     public/cleanup
 unix  2      [ ACC ]     STREAM     LISTENING     6302     private/ifmail
 unix  2      [ ACC ]     STREAM     LISTENING     6306     private/bsmtp
```

## **3. 显示每个协议的统计信息**

 **显示所有端口的统计信息 netstat -s**

```shell
# netstat -s
 Ip:
 11150 total packets received
 1 with invalid addresses
 0 forwarded
 0 incoming packets discarded
 11149 incoming packets delivered
 11635 requests sent out
 Icmp:
 0 ICMP messages received
 0 input ICMP message failed.
 Tcp:
 582 active connections openings
 2 failed connection attempts
 25 connection resets received
 Udp:
 1183 packets received
 4 packets to unknown port received.
 .....
```

 **显示 TCP 或 UDP 端口的统计信息 netstat -st 或 -su**

## **4. 在 netstat 输出中显示 PID 和进程名称 netstat -p**

netstat -p 可以与其它开关一起使用，就可以添加 “PID/进程名称” 到 netstat 输出中，这样 debugging 的时候可以很方便的发现特定端口运行的程序。

```shell
# netstat -pt
 Active Internet connections (w/o servers)
 Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
 tcp        1      0 ramesh-laptop.loc:47212 192.168.185.75:www        CLOSE_WAIT  2109/firefox
 tcp        0      0 ramesh-laptop.loc:52750 lax:www ESTABLISHED 2109/firefox
```

## **5. 在 netstat 输出中不显示主机，端口和用户名 (host, port or user)**

当你不想让主机，端口和用户名显示，使用 netstat -n。将会使用数字代替那些名称。

同样可以加速输出，因为不用进行比对查询。

```shell
# netstat -an
```

如果只是不想让这三个名称中的一个被显示，使用以下命令

```shell
# netsat -a --numeric-ports
# netsat -a --numeric-hosts
# netsat -a --numeric-users
```

## 6. 显示核心路由信息 netstat -r

```shell
# netstat -r
 Kernel IP routing table
 Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
 192.168.1.0     *               255.255.255.0   U         0 0          0 eth2
 link-local      *               255.255.0.0     U         0 0          0 eth2
 default         192.168.1.1     0.0.0.0         UG        0 0          0 eth2
```

**注意：** 使用 netstat -rn 显示数字格式，不查询主机名称。

## 7. 找出程序运行的端口

并不是所有的进程都能找到，没有权限的会不显示，使用 root 权限查看所有的信息。

```shell
# netstat -ap | grep ssh
 tcp        1      0 dev-db:ssh           101.174.100.22:39213        CLOSE_WAIT  -
 tcp        1      0 dev-db:ssh           101.174.100.22:57643        CLOSE_WAIT  -
```

 **找出运行在指定端口的进程**

```
# netstat -an | grep ':80'
```

## 8. 显示网络接口列表

```shell
# netstat -i
 Kernel Interface table
 Iface   MTU Met   RX-OK RX-ERR RX-DRP RX-OVR    TX-OK TX-ERR TX-DRP TX-OVR Flg
 eth0       1500 0         0      0      0 0             0      0      0      0 BMU
 eth2       1500 0     26196      0      0 0         26883      6      0      0 BMRU
 lo        16436 0         4      0      0 0             4      0      0      0 LRU
```

```
先把状态全都取出来,然后使用uniq -c统计，之后再进行排序。
```



分析状态个数

```shell
wss8848@ubuntu:~$ netstat -nat |awk '{print $6}'|sort|uniq -c
143 ESTABLISHED
1 FIN_WAIT1
1 Foreign
1 LAST_ACK
36 LISTEN
6 SYN_SENT
113 TIME_WAIT
1 established)
```