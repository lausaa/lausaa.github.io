---
layout: post
title: "HDFS 读流程"
date: 2021-10-02
tags: [分布式存储, HDFS]
categories: "202110"
comments: true
author: lausaa
---

HDFS 读流程分为 Client 、 Namenode 和 Datanode 三端的子流程。
# Client 端
## open 文件
与 POSIX 文件系统类似，HDFS 文件的读取流程也是 open -> read -> close 。

其中 open 的输入是文件路径，输出是 FSDataInputStream 。
### FSDataInputStream
FSDataInputStream 继承自 DFSInputStream ，主要维护了读相关的信息，如客户端配置、文件块和缓存策略等。
### LocatedBlocks
LocatedBlocks 主要维护了待读文件所对应的所有数据块 LocatedBlock ，包含数据块状态和数据副本所在的 DN 等。通过 RPC 调用从 Namenode 查询而来。
```
namenode.getBlockLocations(src,start,length,client);
```
## read 文件
read 的输入有三个：(final byte buf[], int off, int len)。

主要分为如下几步实现：
### 定位数据块
由于分级被分割为一组数据块，所以要通过输入的偏移找到对应的数据块，其核心是一个二分查找。
```
Collections.binarySearch(blocks,key,comp);
findBlock:154, LocatedBlocks (org.apache.hadoop.hdfs.protocol)
fetchBlockAt:562, DFSInputStream (org.apache.hadoop.hdfs)
getBlockAt:546, DFSInputStream (org.apache.hadoop.hdfs)
blockSeekTo:681, DFSInputStream (org.apache.hadoop.hdfs)
```
### 选择 DN
对于多副本的情况，从 Namenode 查询到的 Block 所有副本所在的 DN 是按照一定顺序排列的（默认按照客户端的网络拓扑距离排序），此时客户端选择第一个 DN 即可。在读取数据包时，如果发生传输异常，则将当前的 DN 通过 addToLocalDeadNodes 方法记为死节点，然后重试选取下一个 DN。这个重试逻辑是在 readWithStrategy 方法中实现的。
```
<init>:1792, DFSInputStream$DNAddrPair (org.apache.hadoop.hdfs)
getBestNodeDNAddrPair:1141, DFSInputStream (org.apache.hadoop.hdfs)
chooseDataNode:1045, DFSInputStream (org.apache.hadoop.hdfs)
chooseDataNode:1028, DFSInputStream (org.apache.hadoop.hdfs)
blockSeekTo:692, DFSInputStream (org.apache.hadoop.hdfs)
readWithStrategy:900, DFSInputStream (org.apache.hadoop.hdfs)
```
### 数据块读取器 BlockReader
BlockReader 中包含建立与 DN 的网络链接和数据包接收器。
```
创建网络链接：
newConnectedPeer:3112, DFSClient (org.apache.hadoop.hdfs)
nextTcpPeer:824, BlockReaderFactory (org.apache.hadoop.hdfs.client.impl)
getRemoteBlockReaderFromTcp:749, BlockReaderFactory (org.apache.hadoop.hdfs.client.impl)
build:382, BlockReaderFactory (org.apache.hadoop.hdfs.client.impl)
getBlockReader:770, DFSInputStream (org.apache.hadoop.hdfs)
```
```
数据包接收器中维护了一系列的数据包处理方法，如数据包接受，包头解析分片等。
<init>:78, PacketReceiver (org.apache.hadoop.hdfs.protocol.datatransfer)
<init>:103, BlockReaderRemote (org.apache.hadoop.hdfs.client.impl)
newBlockReader:439, BlockReaderRemote (org.apache.hadoop.hdfs.client.impl)
getRemoteBlockReader:865, BlockReaderFactory (org.apache.hadoop.hdfs.client.impl)
getRemoteBlockReaderFromTcp:752, BlockReaderFactory (org.apache.hadoop.hdfs.client.impl)
build:382, BlockReaderFactory (org.apache.hadoop.hdfs.client.impl)
getBlockReader:770, DFSInputStream (org.apache.hadoop.hdfs)
```
### 读取和校验数据
由于数据包包含了若干数据 chunk 和所对应的 checksum，因此读取和校验操作是前后依次执行的。

具体的读取是在 PacketReceiver 的 doRead 方法中完成的，先读取数据包长度，然后取数据包全部数据，最后通过 reslicePacket 方法将数据包数据分片为 Header、Data 和 Checksum。
```
read:161, SocketInputStream (org.apache.hadoop.net)
readChannelFully:258, PacketReceiver (org.apache.hadoop.hdfs.protocol.datatransfer)
doReadFully:209, PacketReceiver (org.apache.hadoop.hdfs.protocol.datatransfer)
doRead:171, PacketReceiver (org.apache.hadoop.hdfs.protocol.datatransfer)
receiveNextPacket:102, PacketReceiver (org.apache.hadoop.hdfs.protocol.datatransfer)
readNextPacket:189, BlockReaderRemote (org.apache.hadoop.hdfs.client.impl)
read:148, BlockReaderRemote (org.apache.hadoop.hdfs.client.impl)
readFromBlock:120, ByteArrayStrategy (org.apache.hadoop.hdfs)
readBuffer:842, DFSInputStream (org.apache.hadoop.hdfs)
readWithStrategy:909, DFSInputStream (org.apache.hadoop.hdfs)
read:1016, DFSInputStream (org.apache.hadoop.hdfs)
read:100, DataInputStream (java.io)
```
一个数据包的结构如下，其中DATA 段为真正数据内容，默认最大长度65536，也可以由 io.file.buffer.size 指定更大长度：
```
PLEN    HLEN      HEADER     CHECKSUMS  DATA
32-bit  16-bit   <protobuf>  <variable length>
```

一个数据包读取完成后，通过 checksum.verifyChunkedSums 方法，对 Data 和 Checksum 逐 chunk 校验，chunk 默认大小为512B。
```
verifyChunked:391, DataChecksum (org.apache.hadoop.util)
verifyChunkedSums:383, DataChecksum (org.apache.hadoop.util)
readNextPacket:218, BlockReaderRemote (org.apache.hadoop.hdfs.client.impl)
read:148, BlockReaderRemote (org.apache.hadoop.hdfs.client.impl)
readFromBlock:120, ByteArrayStrategy (org.apache.hadoop.hdfs)
readBuffer:842, DFSInputStream (org.apache.hadoop.hdfs)
readWithStrategy:909, DFSInputStream (org.apache.hadoop.hdfs)
read:1016, DFSInputStream (org.apache.hadoop.hdfs)
read:100, DataInputStream (java.io)
```

对于读取数据时发生校验错误，将对应的节点和数据块记为坏块后，通过 reportCheckSumFailure 方法向 Namenode 汇报。

对于读取数据时发生IO错误，则会尝试重试当前 DN 的链路读取，如果还是错误，则选取下一 DN。

在接受完所有的数据内容后，最后会读取一个空尾包，代表读取操作正式结束。这个空尾包通过 readTrailingEmptyPacket 方法读取，里面没有数据内容，只有一个代表结尾的标记，其作用也就是进一步的确认读取完成。如果没有结尾标记，则代表读取预期与数据传输不一致，将抛出IO异常。
## close 文件
close 实际上就是把之前 open 返回的 FSDataInputStream 对象关闭掉，此处不多说了。
# Namenode 端
## 查询文件元数据
在整个读流程中，NN 端负责提供指定文件的所有块信息。 在 getBlockLocations 方法中，先找到指定文件的inode，然后调用 inode.getBlocks 获取。
```
inode.getBlocks
createLocatedBlocks:2095, BlockManager (org.apache.hadoop.hdfs.server.blockmanagement)
getBlockLocations:178, FSDirStatAndListingOp (org.apache.hadoop.hdfs.server.namenode)
getBlockLocations:2123, FSNamesystem (org.apache.hadoop.hdfs.server.namenode)
getBlockLocations:766, NameNodeRpcServer (org.apache.hadoop.hdfs.server.namenode)
```
拿到所有的数据块信息后，对于多副本的情况，还要做一个排序，按照网络拓扑的距离，离客户端近的排前面，远的排后面。
```
sortByDistance:979, NetworkTopology (org.apache.hadoop.net)
sortByDistance:936, NetworkTopology (org.apache.hadoop.net)
sortLocatedBlock:547, DatanodeManager (org.apache.hadoop.hdfs.server.blockmanagement)
sortLocatedBlocks:466, DatanodeManager (org.apache.hadoop.hdfs.server.blockmanagement)
sortLocatedBlocks:2222, FSNamesystem (org.apache.hadoop.hdfs.server.namenode)
getBlockLocations:2205, FSNamesystem (org.apache.hadoop.hdfs.server.namenode)
getBlockLocations:766, NameNodeRpcServer (org.apache.hadoop.hdfs.server.namenode)
```
# Datanode 端
## 建立传输链路
作为响应客户端的读取请求的内应，DataXceiver 负责与客户端建立网络传输链路。DataXceiver 对象构造后，会作为一个后台线程执行，负责接收客户端读请求，构造对应的数据块读取处理对象BlockSender。
```
<init>:149, DataXceiver (org.apache.hadoop.hdfs.server.datanode)
create:139, DataXceiver (org.apache.hadoop.hdfs.server.datanode)
run:220, DataXceiverServer (org.apache.hadoop.hdfs.server.datanode)
run:748, Thread (java.lang)
```
## 读取和发送数据
在读取本地文件流程中，涉及如下几个主要数据结构和子流程：
### 数据块发送器 BlockSender
BlockSender 负责提供数据块的读取、校验和发送全套服务。DataXceiver 线程启动后即开始接收来自客户端的读请求，读请求中包含了BlockSender 的主要构造参数，如block、offset 和length 等。

在BlockSender 的构造过程中完成了一系列的流程，主要包含输入输出流的构造、数据块文件和checksum 文件的检查等。

正式的发送数据包在 doSendBlock 方法中完成，如下：
```
while (endOffset > offset && !Thread.currentThread().isInterrupted)) {
  manageOsCache();
  long len = sendPacket(pktBuf, maxChunksPerPacket, streamForSendChunks,
      transferTo, throttler);
  offset += len;
  totalRead += len + (numberOfChunks(len) * checksumSize);
  seqno++;
}
```
上面在客户端读流程中也提到了，一个 packet 分为header、checksum 和data 三部分。在DN 端分为两批发送，第一批是header 和checksum，第二批是data。
```
// 第一批次发送header 和所有checksum
sockOut.write(buf, headerOff, dataOff - headerOff);

// 第二批次发送所有data
fileIoProvider.transferToSocketFully(
    ris.getVolumeRef().getVolume(), sockOut, fileCh,
    blockInPosition, dataLen, waitTime, transferTime);
```
header 记录packet 长度和data 长度等简单信息，这些长度都是通过计算得来，比较简单。

checksum 是通过文件的方式与数据块文件结对记录到本地磁盘的，因此需要按需读取。
```
readChecksumFully:90, ReplicaInputStreams (org.apache.hadoop.hdfs.server.datanode.fsdataset)
readChecksum:656, BlockSender (org.apache.hadoop.hdfs.server.datanode)
sendPacket:565, BlockSender (org.apache.hadoop.hdfs.server.datanode)
doSendBlock:781, BlockSender (org.apache.hadoop.hdfs.server.datanode)
```
data 部分采用零拷贝的方式发送，不需要像checksum 那样显示的读取，零拷贝的封装在FileChannelImpl 中实现。
```
transferToDirectlyInternal:428, FileChannelImpl (sun.nio.ch)
transferToDirectly:493, FileChannelImpl (sun.nio.ch)
transferTo:605, FileChannelImpl (sun.nio.ch)
transferToFully:223, SocketOutputStream (org.apache.hadoop.net)
transferToSocketFully:278, FileIoProvider (org.apache.hadoop.hdfs.server.datanode)
sendPacket:596, BlockSender (org.apache.hadoop.hdfs.server.datanode)
doSendBlock:781, BlockSender (org.apache.hadoop.hdfs.server.datanode)
```
在当前数据块的所有数据包读取并发送完成后，还要额外发送一个空尾包，确认发送结束。
```
sendPacket(pktBuf, maxChunksPerPacket, streamForSendChunks,
    transferTo, throttler);
out.flush();
```
至此，HDFS 的读流程梗概梳理完成。