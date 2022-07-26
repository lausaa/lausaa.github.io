---
layout: post
title: "HDFS 写流程"
date: 2021-10-07
tags: [分布式存储, HDFS]
categories: "202110"
comments: true
author: lausaa
---

与读流程一样，HDFS 写流程也分为 Client 、 Namenode 和 Datanode 三端的子流程。
# Client 端
先看一段简单的客户端写文件代码：
```
FileSystem fs = FileSystem.get(new URI("hdfs://ip:port"), new Configuration());
FSDataOutputStream dataOut = fs.create(new Path("/test/data.file"));

dataOut.write(new byte[4609]);  // 一种写法
dataOut.writeBytes("Test write data");  // 另一种写法

dataOut.close();  // close 中包含了 flush
```
与 POSIX 文件系统类似，HDFS 文件的写数流程是 create/append -> write -> close 。
## create/append 文件
### 获取文件系统实例 DistributedFileSystem:  
FileSystem.get(URI uri, Configuration conf) 的实现中利用了一个 CACHE ，即如果此前已经构造了 FS ，则记录到 CACHE 中，以便于后续直接获取。  
当然也可以通过开关 `fs.$scheme.impl.disable.cache` 关闭掉 CACHE ，这样每次 get 都会构造一个新的 FS 。  
### create/append 的主要实现：
输入是 Path，输出是 FSDataOutputStream 。  
核心实现在 DFSClient 类中，主要包含两个操作：
1. 构造输出流: DFSOutputStream.newStreamForCreate(...)
2. 启动租约: beginFileLease(inodeId, outStream)
### 输出流
FSDataOutputStream 继承自 DataOutputStream ，主要维护了写相关的信息，封装了三个主要接口 write, flush 和 close ，其中也包含数据流处理器 DataStreamer 的构造和启动。  
构造 FSDataOutputStream 的首要参数就是文件，对应的是 HdfsFileStatus 类，通过 RPC `dfsClient.namenode.create(...)` 从 Namenode 获取到。
### 租约
租约在另一篇讲述。
## write 文件
write 接口有多个，一种典型的 write 输入为：(final byte buf[], int off, int len)。  
先看下正常写的调用栈：
```
writeData:110, DFSPacket (org.apache.hadoop.hdfs)
writeChunk:451, DFSOutputStream (org.apache.hadoop.hdfs)
writeChecksumChunks:217, FSOutputSummer (org.apache.hadoop.fs)
flushBuffer:164, FSOutputSummer (org.apache.hadoop.fs)
flushBuffer:145, FSOutputSummer (org.apache.hadoop.fs)
write1:136, FSOutputSummer (org.apache.hadoop.fs)
write:111, FSOutputSummer (org.apache.hadoop.fs)
write:57, FSDataOutputStream$PositionCache (org.apache.hadoop.fs)
write:107, DataOutputStream (java.io)
```
可以看到由于写入的内容包含实际数据和校验数据，所以实际的 output 类是 FSOutputSummer 。
DFSPacket 封装了网络传输的数据缓冲区，用户的要写入的数据会被按指定大小切分放入此 packet 对象中，然后由 DataStreamer 发送给 Datanode。  
DFSPacket 的 writeData 和 writeChecksum 两个方法，分别用于将用户的数据和校验数据拷贝到 packet 对象的内部 buf 。  
当 DFSPacket 的内部 buf 填满后，调用 `enqueueCurrentPacket()` 方法将此 packet 放入 DataStreamer 的处理队列中。  
一个 packet 包的实际数据量大小由参数 `dfs.client-write-packet-size` 指定，然后经过 `computePacketChunkSize(int psize, int csize)` 方法计算得到。

再回顾一下数据的组织形式：
```
file(用户文件) -> block(管理单位) -> packet(传输单位) -> trunk(校验单位)
```
至此，write 的核心内容结束。

## flush 文件
flush 操作要把数据真正的写入到 Datanode ，主要是在 DataStreamer 中完成的。

DataStreamer 包含两个线程，一个用于发送数据，另一个用于接收回执。  
相对应的，有两个队列，一个是发送数据的队列 dataQueue ，另一个是回执队列 ackQueue ，两个队列挂的都是 packet 。

在发送数据之前，客户端需要知道发往哪些 Datanode ，调用栈如下：
```
addBlock:1110, DFSOutputStream (org.apache.hadoop.hdfs)
locateFollowingBlock:2005, DataStreamer (org.apache.hadoop.hdfs)
nextBlockOutputStream:1803, DataStreamer (org.apache.hadoop.hdfs)
run:748, DataStreamer (org.apache.hadoop.hdfs)
```
在 addBlock 中，通过 RPC `dfsClient.namenode.addBlock(...)` 从 Namenode 中获取当前数据对应的数据块的所有 Datanode 。  
有了所有的 Datanode ，然后调用 `createSocketForPipeline(...)` 方法创建发送数据的流水线，具体参考 pipeline 篇。  

构造好了 pipeline ， 就可以正式写入数据了，参考调用栈：
```
write:456, SocketChannelImpl (sun.nio.ch)
performIO:63, SocketOutputStream$Writer (org.apache.hadoop.net)
doIO:144, SocketIOWithTimeout (org.apache.hadoop.net)
write:159, SocketOutputStream (org.apache.hadoop.net)
write:117, SocketOutputStream (org.apache.hadoop.net)
write:122, BufferedOutputStream (java.io)
write:107, DataOutputStream (java.io)
writeTo:193, DFSPacket (org.apache.hadoop.hdfs)
run:807, DataStreamer (org.apache.hadoop.hdfs)
```
需要注意的是，当前块的所有 packet 发送完之后，还会发送一个空 packet 做为结尾标志。然后在所有回执收到后，通过 `endBlock()` 关闭当前块的输出流，回执线程和 pipeline 等。

待一个文件的所有块都写完后，调用 `closeInternal()` 关闭当前块的输出流，回执线程、 pipeline 和释放 packet buf 等。

实际上在上面 write 的流程中，被填满的 packet 会被挂到发送队列，而数据块末尾的未填满的 packet 在 close 流程中会被挂到发送队列，因此 flush 的调用经常会被省略。

## close 文件
close 操作中主要做三件事：
1. 发送未满的 packet ，最后调用 `flushInternalWithoutWaitingAck()` 发送最后一个 packet 。
2. 通知 Namenode 此文件写完了，通过 RPC `dfsClient.namenode.complete(...)` 完成，参考调用栈：
```
completeFile:978, DFSOutputStream (org.apache.hadoop.hdfs)
completeFile:934, DFSOutputStream (org.apache.hadoop.hdfs)
closeImpl:917, DFSOutputStream (org.apache.hadoop.hdfs)
close:872, DFSOutputStream (org.apache.hadoop.hdfs)
close:72, FSDataOutputStream$PositionCache (org.apache.hadoop.fs)
close:101, FSDataOutputStream (org.apache.hadoop.fs)
```
3. 关闭数据流处理器 DataStreamer，其中还包含关闭租约 `endFileLease(fileId)` 。

至此，客户端返回写入成功。

# Namenode 端
Namenode 的写入流程涉及与客户端的几个 RPC 交互和与 Datanode 的块汇报交互。  

## RPC create
create 的操作对象是文件，创建指定文件的 inode ，然后加入到目录树中，此外还有维护租约等操作。先看下 create 的调用栈：
```
addFile:635, FSDirWriteFileOp (org.apache.hadoop.hdfs.server.namenode)
startFile:459, FSDirWriteFileOp (org.apache.hadoop.hdfs.server.namenode)
startFileInt:2815, FSNamesystem (org.apache.hadoop.hdfs.server.namenode)
startFileInt:1313, JDFSNamesystem (org.apache.hadoop.hdfs.server.namenode)
startFile:2714, FSNamesystem (org.apache.hadoop.hdfs.server.namenode)
create:802, NameNodeRpcServer (org.apache.hadoop.hdfs.server.namenode)
```
addFile 中即完成了新建 inode 并加入目录树的操作。  
对于覆盖写的情况，实际上是通过先执行 delete ，然后新写的方式实现的。

## RPC append
append 的操作与 create 不同，因为指定文件是已经存在的，要做的是在文件末尾追加写入数据。因此其操作对象是文件的最后一个数据块和相关的租约恢复。

此时需要总结一下数据块的几个状态：
- COMPLETE: 客户端写完了，足够的（最小）副本数已经都汇报上来了。
- UNDER_CONSTRUCTION: 正在写入数据中, 已写入的数据是可读的。
- UNDER_RECOVERY: 租约过期后，正在操作租约恢复流程。
- COMMITTED: 客户端发送 complete RPC 通知 Namenode 写完了，但是 Namenode 尚未收到足够的副本汇报。

对于 append ，简单来说就是要把末尾的 block 改为 UNDER_CONSTRUCTION ，调用的方法是`fsd.getBlockManager().convertLastBlockToUnderConstruction(...)`。  
如果没有 block ，或者末尾 block 已经满了，则走新块逻辑。

## RPC addBlock
addBlock 分为三步：
1. 检查文件当前数据块是否都是完成状态，是否已达到最大块数，等等。
2. 为新块分配 Datanode ，调用的方法是 `FSDirWriteFileOp.chooseTargetForNewBlock(...)` ，核心规则是第一个节点以和客户端距离最近优先，第二个节点再考虑比如跨机架等因素选择，此外根据实际应用场景，还要考虑可用容量，节点繁忙程度等等。
3. 创建一个新块，加入 BlocksMap 并挂到指定文件上，然后与分配的节点构造一个 LocatedBlock 并返回。参考如下调用关系：
```
getAdditionalBlockInt(...)
> FSDirWriteFileOp.storeAllocatedBlock(...)
  > fsn.createNewBlock(blockType)
    > saveAllocatedBlock(...)
      > addBlock(...)
        > blockInfo.convertToBlockUnderConstruction(...)
          > fsd.getBlockManager().addBlockCollection(blockInfo, fileINode)
          > fileINode.addBlock(blockInfo)
    > return makeLocatedBlock(...)
```

## RPC complete
complete 的调用代表客户端通知 Namenode 文件写完了，然后 Namenode 需要对文件和数据块做一些状态转换。主要在方法 `completeFileInternal(...)` 中实现，调用关系如下：
```
completeFileInternal(...)
> checkLease(...) // 如果租约过期但文件已经完成，则视为完成
> commitOrCompleteLastBlock(...) // 对末尾块状态转换
  > commitBlock(...) // 转为 COMMITTED
  > completeBlock() // 副本数达标则转为 COMPLETE
  > updateNeededReconstructions(...) // 副本数不全则放入待补队列
> addCommittedBlocksToPending(...) // 等待副本汇报
> finalizeINodeFileUnderConstruction(...)
  > toCompleteFile(...) // 转换文件状态
  > removeLease(...) // 删除租约
```

## IBR(Incremental Block Report)
此时 Namenode 已经被客户端通知文件写入完成，终于等来了 Datanode 的块副本汇报。  
Namenode 在处理 IBR 时，需要更新数据块和节点的关系，转换数据块和文件状态等。  
IBR 调用栈参考如下：
```
addStoredBlock:4357, BlockManager (org.apache.hadoop.hdfs.server.blockmanagement)
processAndHandleReportedBlock:5323, BlockManager (org.apache.hadoop.hdfs.server.blockmanagement)
addBlock:5296, BlockManager (org.apache.hadoop.hdfs.server.blockmanagement)
processIncrementalBlockReport:5396, BlockManager (org.apache.hadoop.hdfs.server.blockmanagement)
processIncrementalBlockReport:5363, BlockManager (org.apache.hadoop.hdfs.server.blockmanagement)
processIncrementalBlockReport:5480, FSNamesystem (org.apache.hadoop.hdfs.server.namenode)
```

# Datanode 端
Datanode 的写入流程可以划分为接收数据、数据落盘和副本汇报三个部分。

## 接收数据
数据的接收和流转由 BlockReceiver 类负责，其对象在 DataXceiver 中构造，数据包接转代码如下：
```
DataXceiver.writeBlock(...)
> blockReceiver.receiveBlock(...)
  > new PacketResponder(...) // 如果是第一个节点，则启动一个线程向客户端发送回执
  > while (receivePacket() >= 0) 

receivePacket()
> packetReceiver.receiveNextPacket(in)  // 读数据包
> packetReceiver.mirrorPacketTo(mirrorOut);  // 发给下游
> streams.writeDataToDisk(dataBuf.array(),
              startByteToDisk, numBytesToDisk);  // 本地固化
```
## 数据落盘
### 副本状态
数据块在 Datanode 中是以副本的形式管理的，对应 ReplicaInfo 类。  
DN 的主要工作都是围绕副本展开的，先总结一下副本的几个状态：
- FINALIZED: 副本已经落盘, 不再改动。
- RBW(Replica Being Written): 副本正在写入。
- RWR(Replica Waiting Recovered): 副本等待恢复，DN重启 RBW 转 RWR，租约恢复转 RUR。
- RUR(Replica Under Recovery): 副本正在恢复。
- TEMPORARY: 副本迁移时的状态。

### 磁盘管理
一个数据块副本要具体写入到哪里，是由一串的磁盘数据块管理器决定的：  
`FsDatasetImpl -> FsVolumeList -> FsVolumeImpl -> BlockPoolSlice`  
- 在DN 启动时，会构造一个 FsDatasetImpl 类对象，这个类实现自 FsDatasetSpi<FsVolumeImpl> 接口，负责所有数据块副本和磁盘的管理。
- 副本最终存储到的磁盘称为 Volume ，由 FsVolumeImpl 类对象表示，它是接口 FsVolumeSpi 的实现，封装了文件系统操作、磁盘类型、容量、副本和块池相关管理等等。
- FsVolumeList 类负责 Volume 的具体管理，其对象在 FsDatasetImpl 类中管理。其中维护了一个选盘管理器，选盘管理器可以自行定制，也可选用原生代码提供的 RoundRobinVolumeChoosingPolicy 等，然后在数据副本写入调用 createRbw 时通过 `blockChooser.chooseVolume(...)` 选择要写入的 Volume 。
- 每个 Volume 对于每一个 Namespace 所对应的块池，都提供了一个块池分片 BlockPoolSlice 用于副本读取和管理，在 DN 启动时初始化。
- 在获取到块池分片的目录后，为了便于管理，最终的写入目录通常还需要两层子目录，在方法 `DatanodeUtil.idToBlockDir(...)` 中通过 blockId 计算得到。

### 写入流程
写入流程是在块接收器 BlockReceiver 中完成的，调用栈参考如下：
```
write:327, FileOutputStream (java.io)
write:951, FileIoProvider$WrappedFileOutputStream (org.apache.hadoop.hdfs.server.datanode)
writeDataToDisk:148, ReplicaOutputStreams (org.apache.hadoop.hdfs.server.datanode.fsdataset)
receivePacket:749, BlockReceiver (org.apache.hadoop.hdfs.server.datanode)
receiveBlock:1000, BlockReceiver (org.apache.hadoop.hdfs.server.datanode)
writeBlock:946, DataXceiver (org.apache.hadoop.hdfs.server.datanode)
opWriteBlock:197, Receiver (org.apache.hadoop.hdfs.protocol.datatransfer)
processOp:106, Receiver (org.apache.hadoop.hdfs.protocol.datatransfer)
run:300, DataXceiver (org.apache.hadoop.hdfs.server.datanode)
```
## 副本汇报
写完成后也有一个回执线程 PacketResponder 负责向上游发送回执，其中一个主要操作，就是当一个块写完成时（收到空尾包），调用 finalizeBlock(...) 来结束当前数据块的写入。  
finalizeBlock 有两个主要步骤，一个是转为 FINALIZED 状态，参考调用栈如下：
```
addFinalizedBlock:1066, FsVolumeImpl (org.apache.hadoop.hdfs.server.datanode.fsdataset.impl)
finalizeReplica:1873, FsDatasetImpl (org.apache.hadoop.hdfs.server.datanode.fsdataset.impl)
finalizeBlock:1833, FsDatasetImpl (org.apache.hadoop.hdfs.server.datanode.fsdataset.impl)
finalizeBlock:1550, BlockReceiver$PacketResponder (org.apache.hadoop.hdfs.server.datanode)
run:1504, BlockReceiver$PacketResponder (org.apache.hadoop.hdfs.server.datanode)
run:748, Thread (java.lang)
```
另一个是触发增量块汇报给 Namenode，参考调用栈如下：
```
notifyNamenodeBlock:272, IncrementalBlockReportManager (org.apache.hadoop.hdfs.server.datanode)
notifyNamenodeBlock:333, BPOfferService (org.apache.hadoop.hdfs.server.datanode)
notifyNamenodeReceivedBlock:311, BPOfferService (org.apache.hadoop.hdfs.server.datanode)
notifyNamenodeReceivedBlock:1417, DataNode (org.apache.hadoop.hdfs.server.datanode)
closeBlock:2971, DataNode (org.apache.hadoop.hdfs.server.datanode)
finalizeBlock:1557, BlockReceiver$PacketResponder (org.apache.hadoop.hdfs.server.datanode)
run:1504, BlockReceiver$PacketResponder (org.apache.hadoop.hdfs.server.datanode)
run:748, Thread (java.lang)
```
在 notifyNamenodeBlock 中，通过调用 `addRDBI(rdbi, storage)` 和 `triggerIBR(isOnTransientStorage)` 完成数据块信息的封装和触发汇报。

至此，HDFS 的写流程大致梳理完成。