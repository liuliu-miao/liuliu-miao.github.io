---
layout: post
title: Zookeeper_Kafka单节点模式升级为集群模式
categories: [Zookeeper,Kafka]
description: Zookeeper,Kafka upgrade from standlone to cluster mode
keywords:  Zookeeper Kafka Standlone Cluster-Mode
---
# 写在前面
目前集群中zk,kafka为单节点模式（standlone），flume上传数据到kafka中，kafka依赖zk，flink相关任务消费kafka数据。
将zk和kafka升级为集群模式时，最好将flume停止和flink相关任务停止。
且最后升级完成后，需要对zk和kafka进行测试。
需要将kafka中的topic重新分配到不同broker中，然后再启动Flume，观察每个broker中的流量。
最后再启动Flink任务。观察消费流量。（升级过程中开启kafka的JMX端口进行流量查看）
# 1. 增加zk节点
**1.1  将现有节点停止，编辑配置文件zoo.cfg，将其设为集群模式。**
``` yml
dataDir=/home/elex/zookeeper/data
dataLogDir=/home/elex/zookeeper/log
# the port at which the clients will connect
clientPort=2181
#admin.serverPort=8888
server.1=127.0.0.1:2287:3387
server.2=127.0.0.1:2288:3388
server.3=127.0.0.1:2289:3389

#线上时：------------------------------------------

dataDir=/home/elex/zookeeper/data
dataLogDir=/home/elex/zookeeper/log
# the port at which the clients will connect
clientPort=2181
server.1=ip1:2288:3388
server.2=ip2:2288:3388
server.3=ip3:2288:3388

```
**1.2 添加新节点。配置 myid文件**
同步主节点zookeeper目录到其他节点。修改myid文件
``` bash
# dataDir,dataLogDir目录
mkdir -p /home/elex/zookeeper/data
mkdir -p /home/elex/zookeeper/log
#在节点1，2，3上分别执行 myid中值记得都不同，和server.x保持一致。
echo 1 > /home/elex/zookeeper/data/myid
```

**1.3 增加完新节后，启动集群，分别到每个节点启动**
后期考虑写脚本一件启停。
``` bash
./bin/zkServer.sh start conf/zoo.cfg

./bin/zkServer.sh status

#全部启动后，可以停止leader看，是否其他节点可以主动选举为新的leader
./bin/zkServer.sh stop
```

# 2.增加kafka节点
(以下配置为测试环境，单机模拟集群模式，线上环境需要换上真实ip和端口)
**2.1 修改现有kafka节点，将其设为集群模式（如果已经为集群模式，跳过）。**
```yml
#broker.id每个节点不一样，记得修改
broker.id=0
#数据目录，记得改为真实的数据目录。
log.dirs=/home/test1/kafkalog,/home/test2/kafkalog
#zk地址改成zk的集群地址
zookeeper.connect=10.0.3.151:2181,10.0.3.151:2182,10.0.3.151:2183
```

**2.2 增加新的kafka节点** **broker.id不能重复**
将kafka目录拷贝到其他节点，然后删除新加节点的dataDir和dataLogDir。
```bash
scp kafka root@ip1:xxx
cd kafka 
rm -rf data/*
rm -rf log/*
# 修改 
vim conf/server.properties
broker.id=1
broker.id=2
#根据新节点的实际IP和目录修改
listeners=PLAINTEXT://10.0.3.151:9092
zookeeper.connect=10.0.3.151:2181,10.0.3.151:2182,10.0.3.151:2183
log.dirs=/home/test1/kafkalog,/home/test2/kafkalog
```
增加完成后，启动每一个节点的kafka进程,并启用JMX监控
```bash
env JMX_PORT=9999 bin/kafka-server-start.sh -daemon ./config/server.properties &
# 或者 修改kafka-run-class.sh脚本，第一行增加JMX_PORT=9988开启监控。
```

**2.3将现有的topic分配到不同的broker中。**
```bash 
vim event-topic.json
cat << EOF > event-topic.json
{"topics":  [{"topic": "event"}],
"version":1
}

EOF

```
**注意： 后期增加的topic不用指定，会自动分配到不同的broker中。**

```bash 
# 查看当前topic分布
./bin/kafka-topics.sh --describe --zookeeper 10.0.3.151:2181 --topic event
Topic: event    PartitionCount: 3       ReplicationFactor: 1    Configs: 
        Topic: event    Partition: 0    Leader: 0       Replicas: 0     Isr: 0
        Topic: event    Partition: 1    Leader: 1       Replicas: 0     Isr: 1
        Topic: event    Partition: 2    Leader: 2       Replicas: 0     Isr: 2

# 生成计划
./bin/kafka-reassign-partitions.sh --zookeeper 10.0.3.151:2181  --topics-to-move-json-file event-topic.json --broker-list "0,1,2" --generate
Current partition replica assignment
{"version":1,"partitions":[{"topic":"event","partition":2,"replicas":[0],"log_dirs":["any"]},{"topic":"event","partition":1,"replicas":[0],"log_dirs":["any"]},{"topic":"event","partition":0,"replicas":[0],"log_dirs":["any"]}]}

Proposed partition reassignment configuration
#将这部分信息 得到一个  event-move.json 文件用于执行计划
{"version":1,"partitions":[{"topic":"event","partition":0,"replicas":[1],"log_dirs":["any"]},{"topic":"event","partition":2,"replicas":[0],"log_dirs":["any"]},{"topic":"event","partition":1,"replicas":[2],"log_dirs":["any"]}]}

#执行计划
./bin/kafka-reassign-partitions.sh --zookeeper 10.0.3.151:2181 --reassignment-json-file event-move.json --execute

Current partition replica assignment
                                                                                                
{"version":1,"partitions":[{"topic":"event","partition":2,"replicas":[0],"log_dirs":["any"]},{"topic":"event","partition":1,"replicas":[0],"log_dirs":["any"]},{"topic":"event","partition":0,"repl
icas":[0],"log_dirs":["any"]}]}   
                                                                                                
Save this to use as the --reassignment-json-file option during rollback
Successfully started reassignment of partitions.

#校验执行计划
./bin/kafka-reassign-partitions.sh --zookeeper 10.0.3.151:2181 --reassignment-json-file event-move.json  --verify
Status of partition reassignment: 
Reassignment of partition event-2 completed successfully
Reassignment of partition event-1 completed successfully
Reassignment of partition event-0 completed successfully

#再次查看，分区分布在不同的broker上了
./bin/kafka-topics.sh --describe --zookeeper 10.0.3.151:2181 --topic event

Topic: event    PartitionCount: 3       ReplicationFactor: 1    Configs: 
    Topic: event    Partition: 0    Leader: 0       Replicas: 0     Isr: 0
    Topic: event    Partition: 1    Leader: 1       Replicas: 1     Isr: 1
    Topic: event    Partition: 2    Leader: 2       Replicas: 2     Isr: 2

```
- kafka迁移校验
``` bash
./bin/kafka-reassign-partitions.sh --zookeeper 10.0.3.151:2181 --reassignment-json-file event-move.json  --verify
#输出
Status of partition reassignment: 
Reassignment of partition event-2 completed successfully
Reassignment of partition event-1 completed successfully
Reassignment of partition event-0 completed successfully
```
**注意： 在执行kafka迁移计划验证时，视topic数据量大小，可能需要很长时间。**
需要等待结果：Reassignment of xxx completed successfully均为Sucessfully才算完成，
如果有Progress的，需要等待。
上线环境时，数据量过大，topic数量也比较多，等待了大约2个小时。

**2.4 增加topic分区**（若后期数据量过大，效率低的情况再酌情增加分区）
```bash
./bin/kafka-topics.sh --zookeeper zk01:2181,zk02:2181,zk03:2181 --alter --topic track_pc --partitions 3
# 或使用kafka-manager进行修改

``` 

# 其他
1. 项目经验之Kafka机器数量计算
    Kafka机器数量（经验公式）=2*（峰值生产速度*副本数/100）+1

    峰值生产速度，再根据设定的副本数，就能预估出需要部署Kafka的数量。
    比如我们的峰值生产速度是50M/s。副本数为2。
    Kafka机器数量=2*（50*2/100）+ 1=3台

2. 项目经验值Kafka分区数计算
    创建一个只有1个分区的topic
    测试这个topic的producer吞吐量和consumer吞吐量。
    假设他们的值分别是Tp和Tc，单位可以是MB/s。
    然后假设总的目标吞吐量是Tt，那么分区数=Tt / min（Tp，Tc）

