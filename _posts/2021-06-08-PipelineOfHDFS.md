---
layout: post
title: "HDFS 之Pipeline"
date: 2021-06-08
tags: [分布式存储, HDFS]
categories: "202106"
comments: true
author: lausaa
---

在副本冗余模式的分布式存储系统中，副本分发属于重量级的操作，其中pipeline 思路的应用一定程度上解决了效率问题。

## 设计思路
例如面对一个三副本的冗余配置，我们很轻易的可以想到一个简单的分发方案：  

先写第一副本，写完后拷贝给第二三副本，收工。

乍一看，我们完成了副本分发的需求，但思考之后存在几个问题：  
1. 对于客户端写请求来说，需要快一点，因为出于数据安全性角度来说，客户端需要三副本全部写完后才会认为写请求完成；
2. 对于客户端读请求来说，需要快一点，在拷贝第二三副本之前的无论多少个客户端的读请求都只能集中到第一副本了，压力倍增。

因此，数据流pipeline 思想在此被完美运用。

客户端要写入的数据被拆分成若干小包（packet），在发起写请求之前从NN 获取到三个副本的承载DataNode，然后建立pipeline 链路。  
如下面流程示意，客户端数据被拆成ABCD 四个包，按照pipeline 链路依次发送，当DN 接收到一个包时，先固化到本地，然后转手给下游，直到最后一个DN 固化完成。  
最后一个DN 固化完成后，会原路返回结果，最终客户端收到返回结果，代表本次写请求正式完成。
```
step1: Client(ABCD) -> DN1()     -> DN2()     -> DN3()
step2: Client(BCD)  -> DN1(A)    -> DN2()     -> DN3()
step3: Client(CD)   -> DN1(AB)   -> DN2(A)    -> DN3()
step4: Client(D)    -> DN1(ABC)  -> DN2(AB)   -> DN3(A)
step5: Client()     -> DN1(ABCD) -> DN2(ABC)  -> DN3(AB)
step6: Client()     -> DN1(ABCD) -> DN2(ABCD) -> DN3(ABC)
step7: Client()     -> DN1(ABCD) -> DN2(ABCD) -> DN3(ABCD)
```

## 代码实现
### pipeline 需求场景
1. 新写文件或新写块；
2. Append 文件（最后一个块未满）；
3. 副本恢复（补块）；

### 建立pipeline
以客户端需要新写文件为例，客户端先从NameNode 获取到一组DataNode，然后创建pipeline。  
pipeline 的源头在客户端DFSOutputStream->DataStreamer 类，如下：
```
DataStreamer::run() {
    setPipeline(nextBlockOutputStream());
}

nextBlockOutputStream()
--> createBlockOutputStream
        // nodes[0] 即为第一个DataNode
    --> createSocketForPipeline(nodes[0], nodes.length, dfsClient);
```
pipeline 在DataNode 端有上游和下游两个角色，均在DataXceiver 类管理。  
DataXceiver 在DataXceiverServer 中构造，Server accept 端对应TcpPeerServer 类，Server 处理请求和Client 端对应DataXceiver 类。  
DataNode 进程启动的时候，会构造并启动一个DataXceiverServer Daemon 线程，该线程通过TcpPeerServer 对象accept pipeline 上游连接请求，然后构造并启动对应的DataXceiver，如下:
```
  DataXceiverServer::run() {
    while (datanode.shouldRun && !datanode.shutdownForUpgrade) {
      try {
        peer = peerServer.accept();

        ...

        new Daemon(datanode.threadGroup,
            DataXceiver.create(peer, datanode, this))
            .start();
      }
    }
  }
```
可以看出，一个DataXceiver 对象代表一个pipeline 链路节点，在DataXceiver 中完成了具体的数据读写处理，如下：
```
DataXceiver::run() {
  do {
    op = readOp();
    processOp(op);
  } while ((peer != null) &&
          (!peer.isClosed() && dnConf.socketKeepaliveTimeout > 0));
}

// 连接下游的动作发生在WRITE_BLOCK 这个op 中，最后一个DataNode 没有下游
processOp(op)
--> opWriteBlock(...)
    --> DataXceiver.writeBlock(...)
        --> mirrorTarget = NetUtils.createSocketAddr(mirrorNode);
        --> mirrorSock = datanode.newSocket();
        --> NetUtils.connect(mirrorSock, mirrorTarget, timeoutValue);
```

上下游数据接转由BlockReceiver 类负责，其对象blockReceiver 在DataXceiver 中构造，数据包接转代码如下：
```
DataXceiver.writeBlock(...)
--> blockReceiver.receiveBlock(...)
    --> while (receivePacket() >= 0)

receivePacket()
--> packetReceiver.receiveNextPacket(in)  // 读数据包
--> packetReceiver.mirrorPacketTo(mirrorOut);  // 发给下游
--> streams.writeDataToDisk(dataBuf.array(),
              startByteToDisk, numBytesToDisk);  // 本地固化
```
至此，pipeline 链路建立完成。



