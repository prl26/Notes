

go-ethereum 源码解析系列一(启动流程)

​	以太坊通过geth来启动一个网络节点的,而`geth`位于`cmd/geth/main.go`文件中，入口如下：

![image-20231115172125465](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231115172125465.png)

我们通过下面这张图可以看出来：main()并不是真正意义上的入口，在初始化完常量和变量以后，会先调用模块的init()函数，然后才是main()函数。所以初始化的工作是在init()函数里完成的。

![img](https://camo.githubusercontent.com/8404654b8b16326e169a81f2e8b5fb1d90122a56c1b99b6b57df92e1d7c5cdbd/68747470733a2f2f747661312e73696e61696d672e636e2f6c617267652f30303753385a496c677931676a6d6b756c333032706a33316775306d636e64322e6a7067)

init函数如下:

![image-20231115172236389](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231115172236389.png)

​	以太坊是基于一系列的启动参数来配置不同的服务的,app.Run()会根据不同的命令行参数来选择并执行对应的子命令,这里会涉及cli库的使用,关于其中command的使用,参考这段代码.

**![image-20231129192132522](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231129192132522.png)**

从这里我们可以找到入口函数geth:

![image-20231115172313041](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20231115172313041.png)

可以看到默认是启动一个全节点,

主要有几个函数会被调用:

```
prepare   //操作内存缓存余量并设置度量系统。
```

```
makeFullNode   //加载配置和注册服务
```

```
startNode      //启动节点
```

然后stack.Wait(),守护进程,等待堆栈退出,或者说是节点下线