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

此时我们需要了解一些namenode 的元数据管理机制和概念。
- **EditLog**  
为了同步元数据信息，Active NN 在服务客户端请求（如新建文件）时会不断的生产日志事务，对应一个txid，即EditLog，若干条EditLog 构成一个edits 文件，可以说edits 文件是元数据的同步单元。  
edits 文件也就是所谓的EditLog Segment，分为写入和完成两种状态，分别对应两种命名格式的文件，edits_inprogress_$start_txid 文件为写入状态，edits_$start_txid-$end_txid 文件为完成状态。  
写入到完成也叫log-roll，有两个触发情况：  
一是Active NN 周期检查是否超过了edits 回滚阈值，周期参数为dfs.namenode.edit.log.autoroll.check.interval.ms，回滚阈值参数为dfs.namenode.edit.log.autoroll.multiplier.threshold * dfs.namenode.checkpoint.txns；  
二是Standby NN 会周期的通知Active NN 回滚，因为Standby 只会读完成状态的edits 文件，周期参数为dfs.ha.log-roll.period；

- **FSImage**  
edits 文件数会随着业务不断积累增长，为了解决此问题，需要找一个合适的机会对edits 文件做合并，合并后的文件即为FSImage。  
这个合并操作即为**Checkpoint**，也叫FSImage 回滚。  
Checkpoint 的触发条件有两个，满足其一即可：  
一是时间，对应参数dfs.namenode.checkpoint.period，默认是3600s；  
二是数量，即事务数量达到阈值，参数dfs.namenode.checkpoint.txns，默认1000000；  
由于Checkpoint 操作的计算和内存开销都比较大，且对客户端请求有直接的影响，因此由Standby NN 完成。  
而后，Standby NN 向Active NN 的ImageServlet 发送HTTP GET 请求，这个请求包含了新的FSImage 文件的txid 以及下载信息。Active NN 下载后将文件命名为fsimage.ckpt_*， 然后创建MD5 校验和，最后将fsimage.ckpt_* 重命名为fsimage_*。

基于上述元数据管理机制，我们需要一个可靠的中介模块来传输edits 文件，也就是上面提到的QJM 和NFS，而官方更倾向与QJM 方式。  

### JournalNode
JournalNode 是一个集群，由3 个以上的奇数个节点组成（paxos 协议？），JN 服务仅用于中介edits 文件传输，属于比较轻量级的。  
Active NN 会将edits 文件同步给JN，然后Standby NN 会周期的从JN 同步获取最新edits（周期参数dfs.ha.tail-edits.period），然后在本地执行Checkpoint 操作。




