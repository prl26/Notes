# RLPX握手协议:

## 这里先普及一下知识:

一些符号:

- `X || Y`：表示X和Y的串联
- `X ^ Y`： X和Y按位异或
- `X[:N]`：X的前N个字节
- `[X, Y, Z, ...]`：[X, Y, Z, ...]的RLP递归编码
- `keccak256(MESSAGE)`：以太坊使用的keccak256哈希算法
- `ecies.encrypt(PUBKEY, MESSAGE, AUTHDATA)`：RLPx使用的非对称身份验证加密函数 AUTHDATA是身份认证的数据，并非密文的一部分 但是AUTHDATA会在生成消息tag前，写入HMAC-256哈希函数
- `ecdh.agree(PRIVKEY, PUBKEY)`：是PRIVKEY和PUBKEY之间的椭圆曲线Diffie-Hellman协商函数

### **ECDH**:

- **用途：** ECDH 用于密钥协商，允许两个通信方通过椭圆曲线密码学来协商一个共享的对称密钥，用于后续的对称加密通信。
- **关键概念：** 双方各自有自己的椭圆曲线密钥对，通过协商生成一个共享的对称密钥。

### **ECIES**:

- **用途：** ECIES 结合了椭圆曲线 Diffie-Hellman(ECDH) 密钥协商、对称密钥加密和其他密码学原语，为通信方提供端到端的安全性。

- **关键概念：** 使用 ECDH 进行密钥协商，然后使用对称密钥加密消息，同时包括 HMAC 用于认证。

- RLPx使用的密码系统包括：

  - 椭圆曲线 secp256k1，其生成器为 G。
  - KDF(k, len)：NIST SP 800-56串联密钥派生函数。
  - MAC(k, m)：使用SHA-256哈希函数的HMAC。
  - AES(k, iv, m)：CTR模式下的AES-128加密函数。

  ​    Alice希望发送一条只有Bob的静态私钥 kB 可以解密的加密消息。Alice知道Bob的静态公钥为 KB。

  ​    为了加密消息 m，Alice生成一个随机数 r，并计算相应的椭圆曲线公钥 R = r * G，然后计算共享秘密 S = Px，其中 (Px, Py) = r * KB。(这里的计算是使用椭圆点乘法实现的)

  ​    她从中派生加密和身份验证的密钥材料，即 kE || kM = KDF(S, 32)，以及一个随机的初始化向量 iv。

  ​    (这个函数的目的是从输入 *S* 派生一个固定长度的输出，即 32 字节。KDF 常用于从高熵的输入（在这里是 *S*）生成更多用于加密的密钥材料。输出的 32 字节被分为两部分，分别用于加密和身份验证：

  - *kE*（Encryption Key）: 用于加密消息的密钥。

  - *kM*（MAC Key）: 用于消息认证码（MAC）的密钥。

    )

  ​    Alice将加密后的消息发送给Bob，消息内容为 R || iv || c || d，其中 c = AES(kE, iv, m)，而 d = MAC(sha256(kM), iv || c)。

  ​    为了解密消息 R || iv || c || d，Bob需要从中派生共享秘密 S = Px，其中 (Px, Py) = kB * R，并计算加密和身份验证密钥 kE || kM = KDF(S, 32)。Bob通过检查 d == MAC(sha256(kM), iv || c) 来验证消息的真实性，然后获取明文 m = AES(kE, iv || c)。

    (根据椭圆点乘法的性质,Alice和Bob生成的S=Px是相同的)

### **ECDSA:**

- **用途：** ECDSA 是基于椭圆曲线的数字签名算法，用于在数字通信中生成和验证数字签名。
- **关键概念：** 使用椭圆曲线上的私钥生成数字签名，同时使用对应的公钥进行验证。

### **secp256k1:**

- **用途：** secp256k1 是椭圆曲线的一种，用于实现密码学算法，主要用于比特币和其他加密货币中的地址生成、数字签名等。

- **关键概念：** 比特币使用 secp256k1 曲线进行数字签名，以确保交易的合法性。

    

### **椭圆曲线:**	

1. **NIST 曲线：**
   - **NIST P-256 (secp256r1)**: 使用 256 位素数域上的椭圆曲线，被用于 ECC（椭圆曲线加密）和 ECDSA（椭圆曲线数字签名算法）。
   - **NIST P-384 (secp384r1)**: 使用 384 位素数域上的椭圆曲线，同样用于 ECC 和 ECDSA。
   - **NIST P-521 (secp521r1)**: 使用 521 位素数域上的椭圆曲线，同样用于 ECC 和 ECDSA。
2. **Brainpool 曲线：**
   - 由德国 Brainpool 标准组织定义，具有一些特定的性质，可用于 ECC。
3. **Curve25519 和 Curve448：**
   - **Curve25519**: 由 Daniel J. Bernstein 设计，使用 255 位素数域上的椭圆曲线，用于密钥协商。
   - **Curve448**: 由 Daniel J. Bernstein 设计，使用 448 位素数域上的椭圆曲线，同样用于密钥协商。
4. **secp256k1：**
   - 在比特币中广泛使用的椭圆曲线，使用 256 位素数域上的曲线，主要用于比特币的地址生成和数字签名。

## RLPx

###  节点身份 

​	所有的加密操作都基于secp256k1椭圆曲线。每个节点被期望维护一个静态的secp256k1私钥，该私钥在不同会话之间保存和恢复。建议私钥只能通过手动方式进行重置，例如通过删除文件或数据库条目。

###   初始握手

​	通过创建TCP连接并就进一步的加密和身份验证通信达成临时密钥材料，建立RLPx连接。创建这些会话密钥的过程被称为“握手”，在“发起者”（打开TCP连接的节点）和“接收者”（接受TCP连接的节点）之间进行。

- 发起者连接到接收者并发送其身份验证消息
- 接收者接受、解密并验证身份验证消息（检查签名的恢复是否等于keccak256(ephemeral-pubk)）
- 接收者从远程临时公钥和随机数生成身份验证应答消息
- 接收者派生密钥并发送包含Hello消息的第一个加密帧
- 发起者接收身份验证应答并派生密钥
- 发起者发送包含发起者Hello消息的第一个加密帧
- 接收者接收并验证第一个加密帧
- 发起者接收并验证第一个加密帧
- 如果第一个加密帧的MAC在双方上都是有效的，密码握手完成
- 如果第一个帧的数据包身份验证失败，任何一方均可断开连接。

​	这里推荐大家看官方的一个md:https://github.com/ethereum/devp2p/blob/master/rlpx.md,里面关于rlpx协议的流程等写的很详细.

## 流程:

​     在p2p这个模块中,RLPX主要出现在setupConn(建立连接)的函数中,所有的加密操作都基于**secp256k1**椭圆曲线。每个节点维护一个静态的**secp256k1**私钥。建议该私钥只能进行手动重置（例如删除文件或数据库条目）。

​	这里结合一下代码来看:   这里是调用的上层函数

```
remotePubkey, err := c.doEncHandshake(srv.PrivateKey)
if err != nil {
   srv.log.Trace("Failed RLPx handshake", "addr", c.fd.RemoteAddr(), "conn", c.flags, "err", err)
   return fmt.Errorf("%w: %v", errEncHandshakeError, err)
}
```

**协议版本握手**

> 协议握手,输入输出均是protoHandshake对象,包含了版本号、名称、容量、端口号、ID和一个扩展属性,握手时会对这些信息进行验证

## 加密握手

> 握手时主动发起者叫**initiator**
>
> 接收方叫**receiver**
>
> 分别对应两种处理方式**initiatorEncHandshake**和receiverEncHandshake
>
> 两种处理方式成功以后都会得到一个**secrets**对象,保存了共享密钥信息,它会跟原有的**net.Conn**对象一起生成一个帧处理器:**rlpxFrameRW**
>
> 握手双方使用到的信息有:各自的公私钥地址对**(iPrv,iPub,rPrv,rPub)**、各自生成的随机公私钥对**(iRandPrv,iRandPub,rRandPrv,rRandPub)**、各自生成的临时随机数**(initNonce,respNonce).** 其中i开头的表示发起方**(initiator)**信息,r开头的表示接收方**(receiver)**信息.
>
> ​	这里是initiatorEncHandshake实现的代码

```
// runInitiator negotiates a session token on conn.
// it should be called on the dialing side of the connection.
//
// prv is the local client's private key.
func (h *handshakeState) runInitiator(conn io.ReadWriter, prv *ecdsa.PrivateKey, remote *ecdsa.PublicKey) (s Secrets, err error) {
   h.initiator = true
   //Import an ECDSA public key as an ECIES public key.密钥类型转化
   h.remote = ecies.ImportECDSAPublic(remote)
   //通过私钥生成消息
   authMsg, err := h.makeAuthMsg(prv)
   if err != nil {
      return s, err
   }
   //生成握手所需要的消息包
   authPacket, err := h.sealEIP8(authMsg)
   if err != nil {
      return s, err
   }
   //将消息写入连接conn
   if _, err = conn.Write(authPacket); err != nil {
      return s, err
   }
   //读取远程节点的认证响应消息
  // readHandshakeMsg比较简单。 首先用一种格式尝试解码。如果不行就换另外一种。应该是一
  //种兼容性的设置。 基本上就是使用自己的私钥进行解码然后调用rlp解码成结构体。

//结构体的描述就是下面的authRespV4,里面最重要的就是对端的随机公钥。 双方通过自己的私钥和
//对端的随机公钥可以得到一样的共享秘密。 而这个共享秘密是第三方拿不到的
   authRespMsg := new(authRespV4)
   authRespPacket, err := h.readMsg(authRespMsg, prv, conn)
   if err != nil {
      return s, err
   }
   //处理消息,验证有效性
   if err := h.handleAuthResp(authRespMsg); err != nil {
      return s, err
   }
   // 返回通过握手协议生成的 Secrets 对象
   return h.secrets(authPacket, authRespPacket)
}
```

​	