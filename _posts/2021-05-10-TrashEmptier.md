---
layout: post
title: "HDFS 之回收站TrashEmptier"
date: 2021-05-10
tags: [HDFS, 分布式存储]
categories: "202105"
comments: true
author: lausaa
---

HDFS 提供类似回收站的功能，即被执行删除的文件会被移动到指定的Trash 目录，之后在一个合适的时机才被真正删除。  
参考删除命令：
    
    Usage: hadoop fs [generic options] -rm [-f] [-r|-R] [-skipTrash] [-safely] <src> ...

可以看出，只有指定了”-skipTrash“ 参数，文件才会直接删除，否则进Trash。

## Trash 目录
每个用户都有自己的Trash 目录：
    
    /user/$USERNAME/.Trash  # 当前被删除的文件会被放在.Trash 的子目录Current 下

## 命令行删除流程
命令行在处理”-rm“ 指令时，先尝试执行moveToTrash，后执行delete。

    if (moveToTrash(item) || !canBeSafelyDeleted(item)) {
      return;
    }
    if (!item.fs.delete(item.path, deleteDirs)) {
      throw new PathIOException(item.toString());
    }

对于moveToTrash 操作，真正的执行方法是TrashPolicyDefault::moveToTrash，最终会调用rename 完成。

    // move to current trash
    fs.rename(path, trashPath,
        Rename.TO_TRASH);
    LOG.info("Moved: '" + path + "' to trash at: " + trashPath);
    return true;

## 清空回收站
Active NN 会通过startTrashEmptier 方法启动一个后台线程作为清理器。

    this.emptier = new Thread(new Trash(fs, conf).getEmptier(), "Trash Emptier");

清理器涉及两个重要参数：
- `fs.trash.interval`：用于配置回收站中数据的保存周期，0代表不开启回收站功能；
- `fs.trash.checkpoint.interval`：清理器检查处理周期；

清理器的执行实例主体为：TrashPolicyDefault::Emptier，在run() 函数中完成清理操作，其中涉及一个Checkpoint 概念，其实很简单，被删除的文件被移动到Trash 目录下的Current 子目录，创建检查点会吧Current 子目录重命名为当前时间戳以供删除时判断是否过期：
```
Collection<FileStatus> trashRoots;
trashRoots = fs.getTrashRoots(true);      // list all trash dirs

for (FileStatus trashRoot : trashRoots) {   // dump each trash
    if (!trashRoot.isDirectory())
        continue;
    try {
        TrashPolicyDefault trash = new TrashPolicyDefault(fs, conf);
        // 删除过期的文件
        trash.deleteCheckpoint(trashRoot.getPath(), false);
        // 将当前Trash 子目录Current 以时间戳重命名作为下一次的检查点
        trash.createCheckpoint(trashRoot.getPath(), new Date(now));
    } catch (IOException e) {
        LOG.warn("Trash caught: "+e+". Skipping " +
            trashRoot.getPath() + ".");
    } 
}
```

回收站也可以手动触发清理：

    hdfs dfs -expunge
    hadoop fs -expunge






