---
layout: post
title: "HDFS 网络通信之RPC"
date: 2021-05-12
tags: [HDFS, 分布式存储]
categories: "202105"
comments: true
author: lausaa
---

作为一个分布式存储系统，节点之间的通信是必不可少的，通信方式至少包含两类：
- 指令通信，特点是数据量小，要求低延迟；
- 数据通信，特点是数据量大，要求高带宽；

我们先说一下指令通信，也叫管理网络，也叫IPC（Inter-process communication），也叫RPC（Remote Procedure Call）。

试想一下，节点A 需要获取节点B 的状态S 的值，我们大致的实现如下：
```
1. NodeA -> command(getValue, S) -> NodeB
2. NodeA <- result(S, 0xabc) <- NodeB
```
第一步，NodeA 发送指令给NodeB，指令包含操作类型“getValue” 和操作对象“S”。  
第二步，NodeB 执行操作完成后，发送结果给NodeA，结果包含操作对象“S”和值“0xabc”。

可见，要完成一个RPC，至少需要几个实现要点：
1. 建立网络连接，用于节点间数据收发；
2. 设置指令协议，用于指令识别与处理；
3. 指令及参数内容序列化，用于网络打包收发；

由于一个分布式系统可能需要几十上百种RPC 指令，因此需要一个模块，将上述实现要点封装，然后统一注册配置，以期望保证系统设计与实现的简洁便利性。  
Hadoop RPC 就是干这个用的，指令协议与序列化在其中被封装，名为stub。

Hadop RPC 的实现，主要涉及java 的动态代理，java NIO 和protobuf 等基础技术。

下面我们分析下Namenode 与Datanode 的RPC 相关实现。

## NameNode 作为Server
NameNodeRpcServer 类包含了所有的RPC Server 对象，在进程启动时构造，如下：
```
RPC.setProtocolEngine(conf, ClientNamenodeProtocolPB.class,
    ProtobufRpcEngine2.class);

serviceRpcServer = new RPC.Builder(conf)
    .setProtocol(
        org.apache.hadoop.hdfs.protocolPB.ClientNamenodeProtocolPB.class)
    .setInstance(clientNNPbService)
    .setBindAddress(bindHost)
    .setPort(serviceRpcAddr.getPort())
    .setNumHandlers(serviceHandlerCount)
    .setVerbose(false)
    .setSecretManager(namesystem.getDelegationTokenSecretManager())
    .build();
```
然后，在ProtobufRpcEngine2 做Server 的初始化工作，如下：
```
  public RPC.Server getServer(Class<?> protocol, Object protocolImpl,
      String bindAddress, int port, int numHandlers, int numReaders,
      int queueSizePerHandler, boolean verbose, Configuration conf,
      SecretManager<? extends TokenIdentifier> secretManager,
      String portRangeConfig, AlignmentContext alignmentContext)
      throws IOException {
    return new Server(protocol, protocolImpl, conf, bindAddress, port,
        numHandlers, numReaders, queueSizePerHandler, verbose, secretManager,
        portRangeConfig, alignmentContext);
  }
```
RPC Server 包含三个重要组件，callQueue、connectionManager 和responder，如下：
```
// 用于管理指令队列
this.callQueue = new CallQueueManager<Call>(getQueueClass(prefix, conf),
        getSchedulerClass(prefix, conf),
        getClientBackoffEnable(prefix, conf), maxQueueSize, prefix, conf);

// 网络监听及会话管理
listener = new Listener(port);
connectionManager = new ConnectionManager();

// 用于返回调用结果
responder = new Responder();
```
在Listener 的构造中，启动一个listener 线程用于Socket accept，然启动若干个Reader 线程管理会话，所有的会话会保存到ConnectionManager.connections 管理。

在Responder 的构造中，启动一个线程用于返回结果。


## Datanode 作为Client
Datanode 的client 角色是在进程启动时创建的，也叫BPOfferService，如下：
```
// 每个NS 对应一个BPOfferService
BPOfferService bpos = createBPOS(nsToAdd, nnIds, addrs,
              lifelineAddrs);
```
在BPOfferService 中，为每个Namenode 创建了一个BPServiceActor，如下：
```
for (int i = 0; i < nnAddrs.size(); ++i) {
      this.bpServices.add(new BPServiceActor(nameserviceId, nnIds.get(i),
          nnAddrs.get(i), lifelineNnAddrs.get(i), this));
    }
```
每个BPServiceActor 都是一个独立的线程，如下：
```
//This must be called only by BPOfferService
void start() {
  if ((bpThread != null) && (bpThread.isAlive())) {
    //Thread is started already
    return;
  }
  bpThread = new Thread(this);
  bpThread.setDaemon(true); // needed for JUnit testing

  if (lifelineSender != null) {
    lifelineSender.start();
  }
  bpThread.start();
}
```
BPServiceActor 线程在启动时会构造RPC client 相关组件，如下:
```
run()
-> connectToNNAndHandshake()
    -> connectToNN(nnAddr)
        -> new DatanodeProtocolClientSideTranslatorPB(nnAddr, getConf())

  public DatanodeProtocolClientSideTranslatorPB(InetSocketAddress nameNodeAddr,
      Configuration conf) throws IOException {
    RPC.setProtocolEngine(conf, DatanodeProtocolPB.class,
        ProtobufRpcEngine2.class);
    UserGroupInformation ugi = UserGroupInformation.getCurrentUser();
    rpcProxy = createNamenode(nameNodeAddr, conf, ugi);
  }
```
最终，我们在ProtobufRpcEngine2 类中，找到实际的网络client 初始化，如下：
```
getProxy
-> new Invoker()
    -> Client.ConnectionId.getConnectionId()
        -> new ConnectionId()  // 只是赋值server 地址之类的参数
```
而真正的连接发生在RPC 调用时，首先构造一个connection 对象，如下：
```
Client.call()
-> getConnection()

// connections 用于保存connection 对象，以不必每次call 都重新构造
connection = connections.computeIfAbsent(remoteId,
            id -> new Connection(id, serviceClass, removeMethod));
```
然后创建与server 的会话：
```
connection.setupIOstreams(fallbackToSimpleAuth);
-> setupConnection(ticket); // Socket 配置及连接
-> start(); // 启动接收返回结果的线程
    -> run()
        -> receiveRpcResponse()
```
