---
layout: post
title: "HDFS HA 之Namenode 主备状态管理"
date: 2021-04-29
tags: [HDFS, 分布式存储]
categories: "202104"
comments: true
author: lausaa
---

早期的HDFS 2.0.0，Namenode 是单点的，挂了就全玩儿完。  
为了保证HDFS 的高可用，通常会配置两个Namenode，以active/standby 的模式运行，而zkfc 和zk 负责状态管理。  
zkfc 是一个独立进程，通常与namenode 跑在同一台机器，运行在zk 与namenode 之间，入口类为DFSZKFailoverController。  
zkfc 包含HealthMonitor 和ActiveStandbyElector 两个主要模块。

## ActiveStandbyElector 负责A/S 模式管理
zkfc 通过createLockNodeAsync 在zk 中创建一个EPHEMERAL znode，此模式znode 的特点是临时性，当创建者与zk 的session 中断后，此znode 即被删除。  
session 中断是有一个timeout 的，zk 配置文件中的maxSessionTimeOut 和minSessionTimeOut 两个配置项，意思是zkClient 的连接请求带过来一个sessionTimeout 参数，这个参数在这个[min, max] 区间才合法，否则就取对应的边界值。  
这个znode 相当于一把独占锁，谁成功创建谁就有权成为active namenode。  
ActiveStandbyElector::processResult -> becomeActive 然后通过RPC（HAServiceProtocolHelper.transitionToActive）通知namenode 进程切换为active。  
为了避免双active 的情况发生，在becomeActive 之前，需要fenceOldActive，也是通过RPC（transitionToStandby） 通知原active namenode 切换为standby。

## HealthMonitor 负责Namenode 健康状态监控
HealthMonitor 会启动一个后台轮询线程，调用doHealthChecks 检查所对应的namenode 的状态，然后通过verifyChangedServiceState 函数处理是否需要切换namenode 模式。

## 四个RPC
- transitionToActive(ha.failover-controller.new-active.rpc-timeout.ms)
- transitionToStandby(ha.failover-controller.graceful-fence.rpc-timeout.ms)
- getServiceStatus(ha.failover-controller.new-active.rpc-timeout.ms)
- monitorHealth(ha.health-monitor.rpc-timeout.ms)

## 两个切换方式
1. 手动切换：
    - 通过"hdfs haadmin -ns xxx -failover xxx xxx" 命令手动切换，FailoverController::failover 中先对fromNode 做fence，然后对toNode 做transitionToActive。
2. 自动切换：
    - 如果HealthMonitor 检查到active namenode 状态不正常，则对应zkfc 通过quitElection 接口删除znode，而standy namenode 通过joinElection 创建znode 而成为active。
    - 如果是active namenode 宕机，则其znode 在session 超时后由zk 自动删除，而standby namenode 的zkfc 则通过watcher 回调创建znode，以使namenode becomeActive。
