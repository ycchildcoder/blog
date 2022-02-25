---
title:  http 起源（发展史）
date: 2022-02-21 14:50:31
tags:
---

## HTTP：起源（http0.9 - 1991）

HTTP 的这个概念验证版本与我们现在所知道和喜欢的 HTTP 几乎没有相似之处。

**没有标头，也没有状态码。典型的请求只有`GET/path`而已。响应仅包含 HTML，且 TCP 连接关闭就会结束。**

## HTTP/1.0 出现（1996）

在随后几年中，互联网迎来爆炸式的发展，尽管传输 HTML 仍然是 HTTP 的主要特色，但它逐渐发展成一种可扩展且灵活的通用协议。HTTP 的三大重要更新奠定了这一演变的基础：

-   引入POST类型- 方法的引入使客户能确定其想要执行操作的类型（**POST 命令和 HEAD 命令**）
-   状态码 - 状态码为客户端提供了一种确认服务器已成功处理请求的方法
-   header
-   HTML 支持了图像，样式和其他链接资源（**传输图像、视频、二进制文件。**）

<img src="https://raw.githubusercontent.com/ycchildcoder/markdown/main/9337f0aca5808d665159767e8c05e97a.webp" alt="img" style="zoom: 33%;" />

**在 HTTP/0.9 与 早期 HTTP/1.0 中，默认的就是这种，但这种频繁的创建、断开连接无疑是极大的消耗性能**

所以 HTTP/1.0 引入了 keep-alive 长连接，HTTP/1.0 中是默认关闭的，可以通过 `Connection: keep-alive;` 开启 ，HTTP/1.1 默认是开启的，无论加没加 `Connection: keep-alive;`

当时其实也存在一些别的问题如下：

-   **HTTP/1.0 版的主要缺点是，每个 TCP 连接只能发送一个请求**。发送数据完毕，连接就关闭，如果还要请求其他资源，就必须再新建一个连接。
-   **TCP 连接的新建成本很高，因为需要客户端和服务器三次握手，并且开始时发送速率较慢（slow start）**。



# HTTP/1.1 的标准化（1997 ）

**新版引入的两个最关键的更改是默认使用持久 TCP 连接（保持活动状态）和 HTTP 管线化。**

HTTP 管线化的意思就是客户端无需在发送后续 HTTP 请求之前等待服务器响应请求。此功能可以更有效地利用带宽并减少延迟，但它的改进空间甚至更大。HTTP 管线化仍要求服务器按照接收到的请求顺序进行响应，因此，如果管线化中的单个请求执行得很慢，则对客户端的所有后续响应都将相应地延迟下去。这个问题被称为线头阻塞。

所谓长连接，即在 HTTP 请求建立 TCP 连接时，请求结束，TCP 连接不断开，继续保持一段时间（timeout），在这段时间内，同一客户端向服务器发送请求都会复用该 TCP 连接，并重置 timeout 时间计数器，在接下来 timeout 时间内还可以继续复用 TCP 。这样无疑省略了反复创建和销毁 TCP 连接的损耗。

<img src="https://raw.githubusercontent.com/ycchildcoder/markdown/main/fadfc464089b4dbbb60b0daed9e57dc3.webp" alt="img" style="zoom: 67%;" />

## 长连结复用

`HTTP/1.X`默认开启持久连接，在**一个 `TCP` 连接上可以传送多个 HTTP 请求和响应**，减少了建立和关闭连接的消耗和延迟。

但是**同一域名下的请求有一定数量限制**，超过限制数目的请求会被阻塞。

但是同时也存在一些问题如下：

-   **"队头堵塞"（Head-of-line blocking）**
-   HTTP1.x 在传输数据时，所有传输的内容都是明文，
-   HTTP1.x 在使用时，header 里携带的内容过大



# SPDY 和 HTTP/2

HTTP/2 标准基于 SPDY，并进行了一些改进。HTTP/2 通过在单个打开的 TCP 连接上多路复用 HTTP 请求，解决了线头阻塞问题。这允许服务器以任何顺序响应请求，然后客户端可以在接收到响应时重新组合响应，从而在单个连接中加快整个交换的速度。

综合了 HTTPS 和 HTTP 两者优点于一体的传输协议，主要解决：

-   **降低延迟**，针对 HTTP 高延迟的问题，SPDY 优雅的采取了**多路复用（multiplexing）**。
-   **header 压缩**。
-   **基于 HTTPS 的加密协议传输，大大提高了传输数据的可靠性**。
-   **服务端推送（server push）**

## 多路复用

`HTTP/2.0`支持多路复用，这是 HTTP/1.x 持久连接的升级版。

`多路复用`就是在**一个 TCP 连接中可以存在多条流**，也就是可以发送多个请求，服务端则可以通过帧中的标识知道该帧属于哪个流(即请求)，通过重新排序还原请求。

多路复用允许并发的发起多个请求，每个请求及该请求的响应不需要等待其他的请求或响应，**避免了线头阻塞问题**。这样某个请求任务耗时严重，不会影响到其它连接的正常执行，极大的**提高传输性能、降低数据延迟**。



**为什么 HTTP/2 引入多路复用？**

这是因为：

-   HTTP/1.x 虽然引入了 keep-alive 长连接，但它每次请求必须等待上一次响应之后才能发起，
-   所以，在 HTTP/1.1 中提出了管道机制（默认不开启），下一次的请求不需要等待上一个响应来之后再发送，但这要求服务端必须按照请求发送的顺序返回响应，当顺序请求多个文件时，其中一个请求因为某种原因被阻塞时，在后面排队的所有请求也一并被阻塞，这就是队头阻塞 (Head-Of-Line Blocking)
-   人们采取了很多方法去解决，例如使用多个域名、引入雪碧图、将小图内联等，但都没有从根本上解决问题

**HTTP/2 是怎么做的喃？**

-   首先它引入了 帧（frame）和流（stream），因为 HTTP/1.x 是基于文本的，因为是文本，就导致了它必须是个整体，在传输是不可切割的，只能整体去传
-   既然，**HTTP/2 是基于二进制流的，它就可以把 HTTP 消息分解为独立的帧，交错发送，然后在另一端通过帧中的标识重新组装，这就是多路复用**
-   这就实现了在同一个TCP连接中，同一时刻可以发送多个请求和响应，且不用按照顺序一一对应，即使某个请求任务耗时严重，也不会影响到其它连接的正常执行

<img src="https://raw.githubusercontent.com/ycchildcoder/markdown/main/cb0cde790c2c1737b6dcc7da92712a13.webp" alt="img" style="zoom:67%;" />

## HTTP/1.x keep-alive 与 HTTP/2 多路复用区别

总结一下，HTTP/1.x keep-alive 与 HTTP/2 多路复用区别：

-   HTTP/1.x 是基于文本的，只能整体去传；HTTP/2 是基于二进制流的，可以分解为独立的帧，交错发送
-   HTTP/1.x keep-alive 必须按照请求发送的顺序返回响应；HTTP/2 多路复用不按序响应
-   HTTP/1.x keep-alive 为了解决队头阻塞，将同一个页面的资源分散到不同域名下，开启了多个 TCP 连接；HTTP/2 同域名下所有通信都在单个连接上完成
-   HTTP/1.x keep-alive 单个 TCP 连接在同一时刻只能处理一个请求（两个请求的生命周期不能重叠）；HTTP/2 单个 TCP 同一时刻可以发送多个请求和响应

<img src="https://raw.githubusercontent.com/ycchildcoder/markdown/main/16cff873bf2ec175%7Etplv-t2oaga2asx-watermark.awebp" alt="http2.0" style="zoom:80%;" />

connection -> request -> response -close_connection						keep-alive (request -> response)				 		多路复用



# HTTP/3 革命

