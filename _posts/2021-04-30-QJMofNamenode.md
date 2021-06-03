---
layout: post
title: "HDFS HA 之Namenode 元数据一致性"
date: 2021-04-30
tags: [HDFS, 分布式存储]
categories: "202104"
comments: true
author: lausaa
---

Namenode 主备切换的基础是二者状态机的一致性，例如元数据信息等。为此，HDFS 提供了两种实现方式。
- Quorum Journal Manager（日志集群）
- NFS（共享文件系统）

两种方式的共同点比较明显，都是共享存储，即对所有日志事务做中心管理。

此时我们需要了解一些namenode 元数据磁盘结构管理机制。

## EditLog
为了同步元数据信息，Active NN 在服务客户端请求（如新建文件）时会不断的生产日志事务，对应一个txid，即EditLog，若干条EditLog 构成一个edits 文件，可以说edits 文件是元数据的同步单元。  
edits 文件也就是所谓的EditLog Segment，分为写入和完成两种状态，分别对应两种命名格式的文件：
- edits_inprogress_$start_txid 文件为写入状态
- edits_$start_txid-$end_txid 文件为完成状态

写入到完成也叫log-roll，有两个触发情况：  
- Active NN 启动一个daemon 线程NameNodeEditLogRoller，周期检查是否超过了edits 回滚阈值，如果超过了则调用rollEditLog 方法回滚。  
周期参数为dfs.namenode.edit.log.autoroll.check.interval.ms；  
回滚阈值参数为dfs.namenode.edit.log.autoroll.multiplier.threshold * dfs.namenode.checkpoint.txns；  
- Standby NN 在EditLogTailerThread 线程中主要完成相关的两件事：  
一是判断如果长时间没有回滚操作，则调用 triggerActiveLogRoll 方法通知Active NN 回滚，周期参数为dfs.ha.log-roll.period（因为Standby 只会读完成状态的edits 文件，inprogress 的不管）；
二是通过 doTailEdits 方法从JN 读取EditLogs ，加载并应用到本地的元数据内存；

## FSImage  
edits 文件数会随着业务不断积累增长，为了解决此问题，需要找一个合适的机会对edits 文件做合并，合并后的文件即为FSImage。  
这个合并操作即为Checkpoint，也叫FSImage 回滚。

## Checkpoint
由于Checkpoint 操作的计算和内存开销都比较大，且对客户端请求有直接的影响，因此由Standby NN 完成。  
Checkpoint 操作由StandbyCheckpointer::CheckpointerThread 线程调用doCheckpoint 方法执行。  
触发条件有两个，满足其一即可：
- 时间，对应参数dfs.namenode.checkpoint.period，默认是3600s；  
- 数量，即事务数量达到阈值，参数dfs.namenode.checkpoint.txns，默认1000000；

在Standby NN 的doCheckpoint 方法中，新的FSImage 文件合并完成后，要upload 到Active NN：
1. Standby NN 通过TransferFsImage.uploadImageFromStorage 函数向Active NN 的ImageServlet 发送HTTP 请求，这个请求包含了新的FSImage 文件的txid 等相关信息。  
2. Active NN 下载后将文件命名为fsimage.ckpt_xx， 然后创建MD5，最后将fsimage.ckpt_xx 重命名为fsimage_xx。相关操作的还包括删除多余的FSImage 文件和edits 文件。

edits 文件保存是日志事务数默认1000000 个，参数dfs.namenode.num.extra.edits.retained。  
FSImage 文件默认保存两个，对应参数dfs.namenode.num.checkpoints.retained。

    # 查看edits 文件：
    hdfs oev -i edits -o edits.xml  
    # 查看FSImage 文件：
    hdfs oiv -p XML -i fsimage -o fsimage.xml

## JournalNode
基于上述元数据管理机制，我们需要一个可靠的中介模块来传输edits 文件，也就是上面提到的QJM 和NFS，而官方更倾向与QJM 方式。  
JournalNode 是一个集群，由3 个以上的奇数个节点组成（paxos 协议？），JN 服务仅用于中介edits 文件传输，属于比较轻量级的。  
Active NN 会将edits 文件同步给JN，然后Standby NN 会周期的从JN 同步获取最新edits（周期参数dfs.ha.tail-edits.period），然后在本地执行Checkpoint 及后续操作。

## NameNode 启动加载元数据磁盘结构流程
1. loadFSImageFile - 读取最新的fsimage_xx 生成内存结构；
2. loadEdits - 读取fsimage_xx 之后的edits 文件，并应用到内存结构；
3. Checkpoint - 将当前状态产生一个新的fsimage_xx文件；


