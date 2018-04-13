---
layout: post
title: zookeeper集群相关问题
categories: [Hadoop,Zookeeper]
description: zookeeper集群第一次启动和后续启动相关问题
keywords:  zookeeper
---

### 问题一
##### 问题描述
第N次启动zookeeper集群时,按照节点顺序启动时,无法正常启动

##### 解决方案
需要先启动上次的leader节点.再一次启动其他节点,可以正常启动

### 发现的问题
第一次启动集群时,无需关注启动节点的顺序.第N(N>1)次启动时,需要先启动上次的leader节点,后启动其他节点,才能启动集群


