​		



# 节点发现Discovery

​	节点发现功能主要涉及 **Server** **Table** **udp** 这几个数据结构，它们有独自的事件响应循环，节点发现功能便是它们互相协作完成的。其中，每个以太坊客户端启动后都会在本地运行一个**Server**，并将网络拓扑中相邻的节点视为**Node**，而**Table**是**Node**的容器，**udp**则是负责维持底层的连接。这些结构的关系如下图：		

![image-20201123210628944](https://camo.githubusercontent.com/b54f5c900834c1be40fd30aadbed487f2b76e40f23e352ca9312980c571ec161/68747470733a2f2f747661312e73696e61696d672e636e2f6c617267652f303038314b636b77677931676b7a657462707a6f776a33307a303065677462682e6a7067)



## p2p服务开启节点发现

在P2p的server.go的start方法中.可以看到这里也是由配置驱动的

![image-20231222142710020](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231222142710020.png)

我们主要来研究一下ListenV4,

![image-20231222142834964](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231222142834964.png)

这里主要做了以下几件事

### 1.新建路由表

```
tab, err := newMeteredTable(t, ln.Database(), cfg)
```

新建路由表做了以下几件事:

```
tab, err := newTable(t, db, cfg) //初始化table对象
```

- 设置bootnode（setFallbackNodes）
  - 节点第一次启动的时候，节点会与硬编码在以太坊源码中的`bootnode`进行连接，所有的节点加入几乎都先连接了它。连接上`bootnode`后，获取`bootnode`部分的邻居节点，然后进行节点发现，获取更多的活跃的邻居节点
  - nursery 是在 Table 为空并且数据库中没有存储节点时的初始连接节点（上文中的 6 个节点），通过 bootnode 可以发现新的邻居

- tab.seedRand：使用提供的种子值将生成器初始化为确定性状态
- loadSeedNodes：加载种子节点；从保留已知节点的数据库中随机的抽取30个节点，再加上引导节点列表中的节点，放置入k桶中，如果K桶没有空间，则假如到替换列表中。

#### 2.测试邻居节点连通性

首先知道UDP协议是没有连接的概念的，所以需要不断的ping 来测试对端节点是否正常，在新建路由表之后，就来到下面的循环，不断的去做上面的事。

```
go tab.loop()// loop schedules runs of doRefresh, doRevalidate and copyLiveNodes.

```

定时运行`doRefresh`、`doRevalidate`、`copyLiveNodes`进行刷新K桶。

首先看一下ethereum的一些配置:

```
const (
   alpha           = 3  // Kademlia concurrency factor //Kademlia 并发因子，表示在查找操作中，每一级的最大并行查询数目
   bucketSize      = 16 // Kademlia bucket size	//桶的大大小
   maxReplacements = 10 // Size of per-bucket replacement list	//每个桶的替代节点列表的最大大小。

   // We keep buckets for the upper 1/15 of distances because
   // it's very unlikely we'll ever encounter a node that's closer.
   hashBits          = len(common.Hash{}) * 8
   nBuckets          = hashBits / 15       // Number of buckets 桶的数量确定方法
   bucketMinDistance = hashBits - nBuckets // Log distance of closest bucket 

   // IP address limits.桶中的 IP 地址限制，表示在同一个桶内最多允许有两个相同的 IP 地址，且这两个地址必须在相同的子网（/24）中
   bucketIPLimit, bucketSubnet = 2, 24 // at most 2 addresses from the same /24]
   //表格中的 IP 地址限制，表示在整个表格中，同一个 IP 地址最多出现在 10 个桶中，且这个 IP 地址必须在相同的子网（/24）中
   tableIPLimit, tableSubnet   = 10, 24 

   copyNodesInterval = 30 * time.Second  //定期进行复制节点的操作，时间间隔为 30 秒
   seedMinTableTime  = 5 * time.Minute  //种子节点（seed nodes）的表格最小保持时间为 5 分钟
   seedCount         = 30   //表示系统中的种子节点个数
   seedMaxAge        = 5 * 24 * time.Hour  //种子节点的最大存活时间，表示一个种子节点最多能够存在 5 天
)
```

另外还有一些关于定时器的定义

```
c.revalidateInterval = 10 * time.Minute  
```

```
if cfg.PingInterval == 0 {
   cfg.PingInterval = 10 * time.Second
}
if cfg.RefreshInterval == 0 {
   cfg.RefreshInterval = 30 * time.Minute
}
```

### `2.doRefresh`

doRefresh对随机目标执行查找以保持K桶已满。如果表为空（初始引导程序或丢弃的有故障），则插入种子节点。

主要以下几步：

1. 从数据库加载随机节点和引导节点。这应该会产生一些以前见过的节点

   ```
   tab.loadSeedNodes()
   ```

   

2. 将本地节点ID作为目标节点进行查找最近的邻居节点

   ```
   tab.net.lookupSelf()
   ```

   

   ```
   func (t *UDPv4) lookupSelf() []*enode.Node {
   	return t.newLookup(t.closeCtx, encodePubkey(&t.priv.PublicKey)).run()
   }
   ```

   

   ```
   func (t *UDPv4) newLookup(ctx context.Context, targetKey encPubkey) *lookup {
   	...
   		return t.findnode(n.ID(), n.addr(), targetKey)
   	})
   	return it
   }
   ```

   

   向这些节点发起`findnode`操作查询离target节点最近的节点列表,将查询得到的节点进行`ping-pong`测试,将测试通过的节点落库保存

   经过这个流程后,节点的K桶就能够比较均匀地将不同网络节点更新到本地K桶中。

   ```
   unc (t *UDPv4) findnode(toid enode.ID, toaddr *net.UDPAddr, target encPubkey) ([]*node, error) {
   	t.ensureBond(toid, toaddr)
   	nodes := make([]*node, 0, bucketSize)
   	nreceived := 0
     // 设置回应回调函数，等待类型为neighborsPacket的邻近节点包，如果类型对，就执行回调请求
   	rm := t.pending(toid, toaddr.IP, p_neighborsV4, func(r interface{}) (matched bool, requestDone bool) {
   		reply := r.(*neighborsV4)
   		for _, rn := range reply.Nodes {
   			nreceived++
         // 得到一个简单的node结构
   			n, err := t.nodeFromRPC(toaddr, rn)
   			if err != nil {
   				t.log.Trace("Invalid neighbor node received", "ip", rn.IP, "addr", toaddr, "err", err)
   				continue
   			}
   			nodes = append(nodes, n)
   		}
   		return true, nreceived >= bucketSize
   	})
     //上面了一个管道事件，下面开始发送真正的findnode报文，然后进行等待了
   	t.send(toaddr, toid, &findnodeV4{
   		Target:     target,
   		Expiration: uint64(time.Now().Add(expiration).Unix()),
   	})
   	return nodes, <-rm.errc
   }
   ```

   

3. 查找3个随机的目标节点

   ```
   for i := 0; i < 3; i++ {
   		tab.net.lookupRandom()
   	}
   ```

### 3.`doRevalidate`

doRevalidate检查随机存储桶中的最后一个节点是否仍然存在，如果不是，则替换或删除该节点。

主要以下几步：

1. 返回随机的非空K桶中的最后一个节点

   ```
   last, bi := tab.nodeToRevalidate()
   ```

   

2. 对最后的节点执行Ping操作，然后等待Pong

   ```
   remoteSeq, err := tab.net.ping(unwrapNode(last))
   ```

   

3. 如果节点ping通了的话，将节点移动到最前面

   ```
   tab.bumpInBucket(b, last)
   ```

   

4. 没有收到回复，选择一个替换节点，或者如果没有任何替换节点，则删除该节点

   ```
   tab.replace(b, last)
   ```

##### `copyLiveNodes`

copyLiveNodes将表中的节点添加到数据库,如果节点在表中的时间超过了5分钟。

这部分代码比较简单，就简单阐述。

```
if n.livenessChecks > 0 && now.Sub(n.addedAt) >= seedMinTableTime {
				tab.db.UpdateNode(unwrapNode(n))
			}
```

## 检测各类信息 go t.loop 

​	

```}
resetTimeout := func() {...}  //这是用于重置timeout 定时器,确保在下一个挂起服务超时时触发
```

loop循环主要监听以下几类消息：

- case <-t.closeCtx.Done()：检测是否停止

- p := <-t.addReplyMatcher：检测是否有添加新的待处理消息

  ​	遍历回复匹配器来处理对应的情况,并移除处理完成的匹配器,重置超时计数

- r := <-t.gotreply：检测是否接收到其他节点的回复消息

#### 处理UDP数据包

```
go t.readLoop(cfg.Unhandled)
```

主要有以下两件事：

1. 循环接收其他节点发来的udp消息

   ```
   nbytes, from, err := t.conn.ReadFromUDP(buf)
   ```

   

2. 处理接收到的UDP消息

   ```
   t.handlePacket(from, buf[:nbytes])
   ```

   接下来对这两个函数进行进一步的解析。

   ### 接收UDP消息

   接收UDP消息比较的简单，就是不断的从连接中读取Packet数据，它有以下几种消息：

   - `ping`：用于判断远程节点是否在线。
   - `pong`：用于回复`ping`消息的响应。
   - `findnode`：查找与给定的目标节点相近的节点。
   - `neighbors`：用于回复`findnode`的响应，与给定的目标节点相近的节点列表

##### 处理UDP消息

主要做了以下几件事：

1. 数据包解码

   ```
   packet, fromKey, hash, err := decodeV4(buf)
   ```

   

2. 检查数据包是否有效，是否可以处理

   ```
    packet.preverify(t, from, fromID, fromKey)
   ```

在校验这一块，涉及不同的消息类型不同的校验，我们来分别对各种消息进行分析。

1：`ping`

- 校验消息是否过期
- 校验公钥是否有效

2：`pong`

- 校验消息是否过期
- 校验回复是否正确

3：`findNodes`

- 校验消息是否过期
- 校验节点是否是最近的节点

4：`neighbors`

- 校验消息是否过期
- 用于回复`findnode`的响应，校验回复是否正确

5：`ENRRequest`

6：`ENRResponse`



```

packet.handle(packet, from, fromID, hash)
```

相同的，也会有6种消息，

![image-20231222222151177](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231222222151177.png)

但是我们这边重点讲处理findNodes的消息：

```
func (t *UDPv4) handleFindnode(h *packetHandlerV4, from *net.UDPAddr, fromID enode.ID, mac []byte) {
   req := h.Packet.(*v4wire.Findnode)

   // Determine closest nodes.
   //确定最近的节点
   target := enode.ID(crypto.Keccak256Hash(req.Target[:]))
   //返回最接近的节点所在桶中最接近id的n个节点
   closest := t.tab.findnodeByID(target, bucketSize, true).entries

   // Send neighbors in chunks with at most maxNeighbors per packet
   // to stay below the packet size limit.
   //以每个数据包最多maxNeighbors个块的形式发送节点信息，以保持在数据包大小限制以下。
   p := v4wire.Neighbors{Expiration: uint64(time.Now().Add(expiration).Unix())}
   var sent bool
   for _, n := range closest {
      if netutil.CheckRelayIP(from.IP, n.IP()) == nil {
         p.Nodes = append(p.Nodes, nodeToRPC(n))
      }
      if len(p.Nodes) == v4wire.MaxNeighbors {
         t.send(from, fromID, &p)
         p.Nodes = p.Nodes[:0]
         sent = true
      }
   }
   //最后,如果还有剩余的节点信息未发送或者没有成功发送过，则通过 t.send 方法发送 p
   if len(p.Nodes) > 0 || !sent {
      t.send(from, fromID, &p)
   }
}
```

​	节点发现的实现很大部分上是基于KAD算法,以下是对节点发现逻辑的一个大概整理

![img](https://camo.githubusercontent.com/62707e119bc9fe7d30a3385ff531ddbe9e5c9537c2b207a199154d698a388e7c/68747470733a2f2f747661312e73696e61696d672e636e2f6c617267652f303038314b636b77677931676b7a65786879316b716a33316438306d716166682e6a7067)
