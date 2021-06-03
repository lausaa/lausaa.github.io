---
layout: post
title: "HADOOP 监控之JMX"
date: 2021-06-03
tags: [HDFS, HADOOP, 监控]
categories: "202106"
comments: true
author: lausaa
---

JMX 是HADOOP 集成的监控模块，提供了通过WEB 访问获取几乎所有指标统计的方式。  
JMX 通过JMXJsonServlet 类实现，该类继承了javax.servlet.http.HttpServlet。

访问JMX 指标统计的方式：  
```
WEB 访问 http://10.207.40.136:50070/jmx （默认端口50070）
命令行访问 curl -i http://10.207.40.136:50070/jmx
```
返回结构为json 格式。

通常来说，由于指标太多，返回较慢，我们可以通过 `qry` 参数指定查询项。例如：  
```
http://10.207.40.136:50070/jmx?qry=Hadoop:service=NameNode,name=FSNamesystem
```
还嫌多，`get` 参数可以查询指定项目中的具体指标。例如：
```
http://10.207.40.136:50070/jmx?get=Hadoop:service=NameNode,name=FSNamesystem::BlocksTotal
```

在 JMXJsonServlet 类中，doGet 方法会将指定的指标统计项转换成json 串，各个业务模块的指标项数据的获取途径是 MBeanServer。

JMXJsonServlet 启动的时候会构造一个 MBeanServer，然后各个业务模块启动的时候会将统计对象注册到MBeanServer。如 FSNamesystem 类：
```
  // 直接继承相关的统计类，几个统计类中包含了所有的统计指标获取接口
  public class FSNamesystem implements Namesystem, FSNamesystemMBean,
    NameNodeMXBean, ReplicatedBlocksMBean, ECBlockGroupsMBean {...}

  // 将几个统计项注册到 MBeanServer
  private void registerMBean() {
    // We can only implement one MXBean interface, so we keep the old one.
    try {
      StandardMBean namesystemBean = new StandardMBean(
          this, FSNamesystemMBean.class);
      StandardMBean replicaBean = new StandardMBean(
          this, ReplicatedBlocksMBean.class);
      StandardMBean ecBean = new StandardMBean(
          this, ECBlockGroupsMBean.class);
      namesystemMBeanName = MBeans.register(
          "NameNode", "FSNamesystemState", namesystemBean);
      replicatedBlocksMBeanName = MBeans.register(
          "NameNode", "ReplicatedBlocksState", replicaBean);
      ecBlockGroupsMBeanName = MBeans.register(
          "NameNode", "ECBlockGroupsState", ecBean);
    } catch (NotCompliantMBeanException e) {
      throw new RuntimeException("Bad MBean setup", e);
    }
    LOG.info("Registered FSNamesystemState, ReplicatedBlocksState and " +
        "ECBlockGroupsState MBeans.");
  }
```









