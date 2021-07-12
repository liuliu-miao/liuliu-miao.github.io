---
layout: post
title: Flink Checkpoint AWS S3中
categories: [Flink]
description: flink checkpoint save to aws s3
keywords: flink checkpoint aws s3
---

# 背景
公司目前flink集群采用standlone模式，目前一个jobmanager，一个taskmanager，checkpoint目前只在taskmanager本机文件系统中，后期考虑到数据量上涨，将扩展机器集群模式或flink on yarn。
需要将checkpoint 存储到分布式文件系统，由于集群在国外，选择了aws s3.
简单使用，已做备注。

# 步骤
以下操作均在Flink客户端家目录下操作
## 1.拷贝插件jar包到插件目录
``` bash
mkdir plugins/s3-fs-presto/
cp opt/flink-s3-fs-presto-1.10.1.jar  plugins/s3-fs-presto/
#如果使用hadoop文件系统与s3交互,则使用flink-s3-fs-hadoop-1.10.1.jar包，对应 plugins目录s3-fs-hadoop
```
## 2.修改conf/flink-conf.ymal
``` yml
s3.access-key: xxxxxxxx
s3.secret-key: xxxxxx
s3.ssl.enabled: false
s3.path.style.access: true
s3.endpoint: s3.us-xxx-1.amazonaws.com

#state.backend: filesystem
state.backend: rocksdb

# Directory for checkpoints filesystem, when using any of the default bundled
# state backends.
#
state.checkpoints.dir: s3://flink-rt/flink/checkpoints/
#state.checkpoints.dir: file:///home/ec2-user/flink/checkpointDir/flink-checkpoints

# Default target directory for savepoints, optional.
#
state.savepoints.dir: s3://flink-rt/flink/savepoints/
#state.savepoints.dir: file:///home/ec2-user/flink/checkpointDir/flink-savepoints
state.backend.incremental: true

```

[Flink官网文档](https://ci.apache.org/projects/flink/flink-docs-master/zh/docs/deployment/filesystems/s3/)
## 3.重启Flink集群
**注意：修改完配置记得同步配置到各个节点**
- 如果是flink集群模式要重启集群
- 如果是flink on yarn(no session mode)重启任务即可。
- 如果是flink on yarn（yarn session mode）需要重启Flink-jobmanager集群任务