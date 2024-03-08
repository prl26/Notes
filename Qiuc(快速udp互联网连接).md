# Qiuc(快速udp互联网连接)

从上个世纪 90 年代互联网开始兴起一直到现在，大部分的互联网流量传输只使用了几个网络协议。使用 IPv4 进行路由，使用 TCP 进行连接层面的流量控制，使用 SSL/TLS 协议实现传输安全，使用 DNS 进行域名解析，使用 HTTP 进行应用数据的传输

一方面是历史悠久使用广泛的古老协议，另外一方面用户的使用场景对传输性能的要求又越来越高。如下几个由来已久的问题和矛盾就变得越来越突出。

## 1.减少了 TCP 三次握手及 TLS 握手时间。

1. 经过上面的过程分析可知，要完成一次简短的HTTPS业务数据交互，需要经历：

- 新连接：4RTT + DNS。
- 会话重用：3RTT + DNS

究其原因一方面是TCP和TLS分层设计导致的：分层的设计需要每个逻辑层次分别建立自己的连接状态。另一方面是TLS的握手阶段复杂的密钥协商机制导致的。要降低建连耗时，需要从这两方面着手。



QUIC为规避TCP协议僵化的问题，将QUIC协议建立在了UDP之上。考虑到安全性是网络的必备选项，加密在QUIC里强制的。传输方面参考TCP并充分优化了TCP多年发现的缺陷和不足, 实现了一套端到端的可靠加密传输。通过将加密和连接管理两层合二为一，消除了当前TCP+TLS的分层设计传输引入的时延。

同TLS的握手一样, QUIC的加密握手的核心在于协商出一个加密会话数据的对称密钥。QUIC的握手使用了DH密钥协商算法来协商一个对称密钥。DH密钥协商算法简单来讲, 需要通信双方各自生成自己的非对称公私钥对,双发各自保留自己的私钥,将公钥发给对方,利用对方的公钥和自己的私钥可以运算出同一个对称密钥。详细的原理这里不展开叙述,有专业的[密码学书籍](https://cloud.tencent.com/developer/tools/blog-entry?target=https%3A%2F%2Fbook.douban.com%2Fsubject%2F19986936%2F)对其原理有详细的论述,网上也有很多好的教程对其由深入浅出的总结, 如[这一篇](https://cloud.tencent.com/developer/tools/blog-entry?target=https%3A%2F%2Flabuladong.gitbook.io%2Falgo%2Fdi-wu-zhang-ji-suan-ji-ji-shu%2Fmi-ma-ji-shu)。

如上所述, DH密钥协商需要通行双方各自生成自己的非对称公私钥对。server端与客户端的关系是1对N的关系,明显server端生成一份公私钥对, 让N个客户端公用, 能明显减少生成开销, 降低管理的成本。server端的这份公私钥对就是专门用于握手使用的, 客户端一经获取,就可以缓存下来后续建连时继续使用, 这个就是达成0-RTT握手的关键, 因此server生成的这份公钥称为0-RTT握手公钥

## 2.改进的拥塞控制。

**可插拔**

什么叫可插拔呢？就是能够非常灵活地生效，变更和停止。体现在如下方面：

1. 应用程序层面就能实现不同的拥塞控制算法，不需要操作系统，不需要内核支持。这是一个飞跃，因为传统的 TCP 拥塞控制，必须要端到端的网络协议栈支持，才能实现控制效果。而内核和操作系统的部署成本非常高，升级周期很长，这在产品快速迭代，网络爆炸式增长的今天，显然有点满足不了需求。
2. 即使是单个应用程序的不同连接也能支持配置不同的拥塞控制。就算是一台服务器，接入的用户网络环境也千差万别，结合大数据及人工智能处理，我们能为各个用户提供不同的但又更加精准更加有效的拥塞控制。比如 BBR 适合，Cubic 适合。
3. 应用程序不需要停机和升级就能实现拥塞控制的变更，我们在服务端只需要修改一下配置，reload 一下，完全不需要停止服务就能实现拥塞控制的切换

**单调递增的 Packet Number**

TCP 为了保证可靠性，使用了基于字节序号的 Sequence Number 及 Ack 来确认消息的有序到达。

QUIC 同样是一个可靠的协议，它使用 Packet Number 代替了 TCP 的 sequence number，并且每个 Packet Number 都严格递增，也就是说就算 Packet N 丢失了，重传的 Packet N 的 Packet Number 已经不是 N，而是一个比 N 大的值。而 TCP 呢，重传 segment 的 sequence number 和原始的 segment 的 Sequence Number 保持不变，也正是由于这个特性，引入了 Tcp 重传的歧义问题。

如上图所示，RTO 发生后，根据重传的 Packet Number 就能确定精确的 RTT 计算。如果 Ack 的 Packet Number 是 N+M，就根据重传请求计算采样 RTT。如果 Ack 的 Pakcet Number 是 N，就根据原始请求的时间计算采样 RTT，没有歧义性。

但是单纯依靠严格递增的 Packet Number 肯定是无法保证数据的顺序性和可靠性。QUIC 又引入了一个 Stream Offset 的概念。

即一个 Stream 可以经过多个 Packet 传输，Packet Number 严格递增，没有依赖。但是 Packet 里的 Payload 如果是 Stream 的话，就需要依靠 Stream 的 Offset 来保证应用数据的顺序。如错误! 未找到引用源。所示，发送端先后发送了 Pakcet N 和 Pakcet N+1，Stream 的 Offset 分别是 x 和 x+y。

假设 Packet N 丢失了，发起重传，重传的 Packet Number 是 N+2，但是它的 Stream 的 Offset 依然是 x，这样就算 Packet N + 2 是后到的，依然可以将 Stream x 和 Stream x+y 按照顺序组织起来，交给应用程序处理



**不允许 Reneging**

什么叫 Reneging 呢？就是接收方丢弃已经接收并且上报给 SACK 选项的内容 [8]。TCP 协议不鼓励这种行为，但是协议层面允许这样的行为。主要是考虑到服务器资源有限，比如 Buffer 溢出，内存不够等情况。

Reneging 对数据重传会产生很大的干扰。因为 Sack 都已经表明接收到了，但是接收端事实上丢弃了该数据。

QUIC 在协议层面禁止 Reneging，一个 Packet 只要被 Ack，就认为它一定被正确接收，减少了这种干扰。



**基于 stream 和 connecton 级别的流量控制**

QUIC 的流量控制 [22] 类似 HTTP2，即在 Connection 和 Stream 级别提供了两种流量控制。为什么需要两类流量控制呢？主要是因为 QUIC 支持多路复用。

1. Stream 可以认为就是一条 HTTP 请求。
2. Connection 可以类比一条 TCP 连接。多路复用意味着在一条 Connetion 上会同时存在多条 Stream。既需要对单个 Stream 进行控制，又需要针对所有 Stream 进行总体控制。

QUIC 实现流量控制的原理比较简单：

通过 window_update 帧告诉对端自己可以接收的字节数，这样发送方就不会发送超过这个数量的数据。

通过 BlockFrame 告诉对端由于流量控制被阻塞了，无法发送数据。

QUIC 的流量控制和 TCP 有点区别，TCP 为了保证可靠性，窗口左边沿向右滑动时的长度取决于已经确认的字节数。如果中间出现丢包，就算接收到了更大序号的 Segment，窗口也无法超过这个序列号。

## 3.避免队头阻塞的多路复用。

HTTP2 在一个 TCP 连接上同时发送 4 个 Stream。其中 Stream1 已经正确到达，并被应用层读取。但是 Stream2 的第三个 tcp segment 丢失了，TCP 为了保证数据的可靠性，需要发送端重传第 3 个 segment 才能通知应用层读取接下去的数据，虽然这个时候 Stream3 和 Stream4 的全部数据已经到达了接收端，但都被阻塞住了。

不仅如此，由于 HTTP2 强制使用 TLS，还存在一个 TLS 协议层面的队头阻塞 [12]。

## 4.连接迁移。

在处理第一个初始数据包（Initial packet）之后，每个端点使用其接收到的源连接ID（Source Connection ID）字段的值设置为后续数据包中的目标连接ID（Destination Connection ID）字段;QUIC 通过在传输参数（transport parameters）中包含相应的值来验证每个端点在握手过程中所选择的连接ID。这可以确保用于握手的所有连接ID也通过加密握手进行身份验证。

## 5.前向冗余纠错

