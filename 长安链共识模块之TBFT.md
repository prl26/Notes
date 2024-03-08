# 长安链共识模块之TBFT



1.所在模块位置:

![image-20231030215505961](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231030215505961.png)

主要实现的接口定义在protocol/v2下的consensus_interface下,主要的也就是start和stop两个接口

![image-20231030215808361](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231030215808361.png)

本次笔记要研究的是TBFT,所以我们来到对应的接口实现下

![image-20231030221957721](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231030221957721.png)

我们详细看一下这里:

![image-20231030223303025](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231030223303025.png)

这里有一个判断,进入条件是共识模块的sync服务配置不为空并且它的validatorSet大小有且不为1

![image-20231030223508110](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231030223508110.png)

这里是一个消息总线的注册

![image-20231030223750846](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231030223750846.png)

剩下的我简单注释一下他们的功能,我们主要看它的消息到来时的处理

![image-20231031194956800](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231031194956800.png)

## 目前正在研究leader节点打包好区块,然后向网络广播消息的步骤,对应的case即是proposseBlock

![image-20231031195252342](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231031195252342.png)

![image-20231031195722587](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231031195722587.png)

​	首先肯定是加锁的操作,防止有其他协程重复进行handle,以及一些校验,例如区块的高度,当前节点是否是prroposer,目前所属步骤是否是step_popose,然后会通知核心引擎自己已经不能再提议区块了,该权限会转换到其它节点

![image-20231031195940388](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231031195940388.png)

![image-20231031195919309](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231031195919309.png)

这里还有一个额外的handler的定义,但目前似乎还未投入使用

![image-20231031200810748](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231031200810748.png)

这里有一个qc的定义,似乎是对一些区块信息的填充



![image-20231031201128338](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231031201128338.png)

这里是一个特殊的操作,是tbft中,可以对之前的一些错误交易进行剔除,从而保证链的稳定性与安全性,当收到剔除票数大于指定值时,就会调用核心引擎进行删除.

![image-20231031201412088](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231031201412088.png)

在本方法的最后,有两个很重要的方法调2用,一个是sendConsensusProposal,这个是用于向区块链网络广播提案消息,一个是enterPrevote,这里leader节点会直接进入到preVote阶段,而不会像其它follower一样需要等待收到2/3的节点的合法验证消息后才进入

## 我们先来看一看sendConsensusProposal

![image-20231031202207632](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231031202207632.png)

其方法内容比较简单,一个createProposalTBFTMsg  

从名称我们可以看出是用于创建一个Tbft的共识消息

**![image-20231031202408603](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231031202408603.png)**

比较简单,主要是组装了一下消息内容以及加上了一个type

![image-20231031202532772](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231031202532772.png)

然后是sendConsensusMsg,**两个参数主要是消息内容以及目的地址**,对于to的定义,当为空时意为广播到所有的验证点集合,不为空则为指定地址,而validatorSet是根据链配置的共识节点来填充的

![image-20231031202640894](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231031202640894.png)

当然由于set中节点较多,这个在转发消息时会新开一个goroutine,然后会调用msgbus(消息总线)的publish的方法,主题为SendConsensusMsg

到这里共识模块的任务暂时结束,p2p网络模块的后台消息处理进程会监听到该地址,然后调用对应的处理方法

![image-20231031203700848](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231031203700848.png)

这里我们可以定位到net_service,这里我们看到ns模块注册了该topic的监听

![image-20231031203839410](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231031203839410.png)

![image-20231031203848848](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231031203848848.png)

而register最重要的一点就是会在后台新开一个goroutine,接受msg的消息(channel的使用)

![image-20231031204130718](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231031204130718.png)

收到之后会再开一个goroutine来处理msg,这里会调用所有注册该topic的模块的OnMessage方法,这里我们用到的是net_service下面的

![image-20231031204511660](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231031204511660.png)

## 定位对应的case(这里只有一个),即msg的topic为SendConsensusMsg,这里会使用HandleMsgBusSubscriberOnMessage这个函数来处理对应的消息,如果有错误的话会进行一个日志打印

这里首先有一个对消息类型的断言,如果消息格式正确才会继续执行,而下一个判断是对于消息的类型进行判断,不正确则写入日志并返回

![image-20231101203442450](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231101203442450.png)

这里很明显是一个根据to.目的地址来进行不同的处理,我们目前的流程to为nil,会调用广播的方法

![image-20231101204032447](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231101204032447.png)

这里的判断依旧比较简单,主要就是消息类型的判断和共识节点列表是否为空,通过后会调用consensusBroadcastMsg

![image-20231101204236281](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231101204236281.png)

遍历共识节点列表,wg是一个waitgroup,用于并发控制的流程,每对一个共识节点发起通信都会开启一个后台goroutione来处理任务,最后wail等待所有的任务完成,而每一个循环中的核心函数就是SendMsg方法,这里涉及到p2p模块进行转发,后续会专门写一下这块的内容