---
layout: post
title: "HDFS 之数据节点管理"
date: 2021-08-10
tags: [分布式存储, HDFS]
categories: "202108"
comments: true
author: lausaa
---

今天研究一下Namenode 是如何将众多Datanode 管理起来的。

在Namenode 中，DatanodeManager 类用来管理所有DN。构造路径：  
```
FSNamesystem
--> BlockManager
    --> DatanodeManager 
```

## DN 数据结构
DN 对应三个数据结构，继承关系如下：
```
DatanodeDescriptor -> DatanodeInfo -> DatanodeID
```
* DatanodeID：维护了一些比较基础的信息，如hostname，IP，端口、storageID 等；
* DatanodeInfo：除了继承扩展DatanodeID，还维护一些简单的信息，如空间，容量相关，服务状态等；
* DatanodeDescriptor：除了继承扩展DatanodeInfo，还维护更多具体信息，状态，块相关指令等；
* DatanodeStorageInfo：用来描述DN 上的一个存储目录（一块盘）；

## DN 记录在哪里
### datanodeMap
datanodeMap 以StorageID -> DatanodeDescriptor 的方式，存储了所有的DN。  
所有DN 的查找、遍历、排序，都可以由此结构提供。
```
private final Map<String, DatanodeDescriptor> datanodeMap
      = new HashMap<>();
```

### networktopology
networktopology 以拓扑树的形式维护了所有DN。  
在树中，最高级为根节点“/”，叶子节点为一个DN（DatanodeDescriptor），中间节点为InnerNode（数据中心、机房、机架等）。
```
private final NetworkTopology networktopology;
```

### host2DatanodeMap
host2DatanodeMap，以host -> DatanodeDescriptor 的方式，存储所有DN。  
一般来说与datanodeMap 是一致的，但同一个DN 的StorageID 是可能变的，因此也维护了这个map。
```
private final Host2NodesMap host2DatanodeMap = new Host2NodesMap();
--> private final HashMap<String, DatanodeDescriptor[]> map
    = new HashMap<String, DatanodeDescriptor[]>();
```

## 怎么记录的
DatanodeManager 初始是没有记录任何DN 的。每个DN 启动的时候根据配置文件向目标NN 发送注册请求，然后NN 接收到DN 的注册请求后才记录。

### 注册流程
DN 的BPServiceActor 对象维护一个DN 与NN 通信的主体线程，DN 侧的执行流程如下：
```
connectToNNAndHandshake();
--> bpNamenode = dn.connectToNN(nnAddr);  // 先建立链接，拿到nnproxy
--> NamespaceInfo nsInfo = retrieveNamespaceInfo();  // 然后握手，返回对应的NS 信息
--> register(nsInfo);  // 最后注册
```
注册流程调用：
```
void register(NamespaceInfo nsInfo) throws IOException
--> DatanodeRegistration newBpRegistration = bpos.createRegistration();
--> newBpRegistration = bpNamenode.registerDatanode(newBpRegistration);
```
NN 侧注册调用：
```
NameNodeRpcServer::registerDatanode(DatanodeRegistration nodeReg)
--> namesystem.registerDatanode(nodeReg);
    --> blockManager.registerDatanode(nodeReg);
        --> datanodeManager.registerDatanode(nodeReg);
```
在registerDatanode 方法中，主要构造DatanodeDescriptor 对象，并更新维护上述提到的三个记录DN 的数据结构。

对于networktopology 来说，其核心是一个以clusterMap 为根的拓扑树，其中间节点InnerNode 构造自自机架等信息，流程如下：
1. 获取DN 的机架；
2. 通过机架构造InnerNode；

clusterMap 的初始化：
```
InnerNode clusterMap;

this.factory = InnerNodeImpl.FACTORY;
this.clusterMap = factory.newInnerNode(NodeBase.ROOT);
```
clusterMap 的实际类型为DFSTopologyNodeImpl，其继承关系如下：
```
public class DFSTopologyNodeImpl extends InnerNodeImpl {}
public class InnerNodeImpl extends NodeBase implements InnerNode {}
```
对于获取机架，HADOOP 采用了一个叫机架感知的技术，即我们通过参数指定一个脚本，该脚本的输入参数是一个DN 的IP 地址，输出是该DN 的机架字符串，然后在注册流程中，NN 拿到DN IP 后通过该脚本获取对应机架。

获取机架的代码如下：
```
DatanodeManager::resolveNetworkLocation(DatanodeID node)
--> dnsToSwitchMapping.resolve(names);
    --> ScriptBasedMapping::resolve(List<String> names)

* 脚本配置参数：net.topology.script.file.name
```
新机架的DN 加入时，会通过机架名称构造InnerNode。  
获取机架，创建机架节点并挂到父节点（clusterMap），最后把DN 加到机架节点中：
```
networktopology.add(node)
--> clusterMap.add(node)
    --> String parentName = getNextAncestorName(n);
    --> parentNode = createParentNode(parentName);
    --> children.add(parentNode);
    --> childrenMap.put(parentNode.getName(), parentNode);
    --> parentNode.add(n)
```
## 总结
除上述外，还有节点删除、刷新、心跳和上下线等操作，逻辑比较简单。  
总之，所有的DN 在NN 端都是在这三个MAP 中管理起来的，论用途的话，不同的场合选用不同的结构。  
比如datanodeMap 和host2DatanodeMap 都可以看做简单的HashMap，key 不同而已，而networktopology 这个拓扑树用作节点索引，通常用作为数据块选点选盘。

