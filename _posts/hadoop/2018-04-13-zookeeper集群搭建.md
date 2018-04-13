---
layout: post
title: zookeeper集群搭建
categories: [Hadoop,Zookeeper]
description: zookeeper集群搭建
keywords: zookeeper集群搭建, zookeeper
---

## 准备工作
* jdk环境
* zookeeper安装包(本文使用的是zookeeper-3.4.10 版本) 可到[Zookeeper官网](http://zookeeper.apache.org/releases.html){:target="_blank"}下载

## 步骤
#### 1.解压文件
解压目标路径 /home/admin/module
``` bash 
tar -zxvf zookeeper-3.4.10.tar.gz -C /home/admin/module/
```

#### 2.修改配置
进入到zookeeper-3.4.10目录修改相关位置
``` bash
cd /home/admin/module/zookeeper-3.4.10
mkdir zkData
cd conf
mv zoo_sample.cfg zoo.cfg
vim zoo.cfg
dataDir=/home/admin/module/zookeeper-3.4.10/zkData
保存退出

cd ../zkData
echo 2 > myid
## 最后一行下面增加
## 配置集群服务地址
server.2=hd001:2888:3888
server.3=hd002:2888:3888
server.4=hd003:2888:3888

```
##### 配置参数解读
Server.A=B:C:D。 <br>
A是一个数字，表示这个是第几号服务器； <br>
B是这个服务器的ip地址； <br>
C是这个服务器与集群中的Leader服务器交换信息的端口； <br>
D是万一集群中的Leader服务器挂了，需要一个端口来重新进行选举，选出一个新的 <br>

#### 3.分发文件
完成以上配置后,将此目录分发到其他机器节点
``` bash
rsync -rvl /home/admin/module/zookeeper-3.4.10 hd002:~/module/
rsync -rvl /home/admin/module/zookeeper-3.4.10 hd003:~/module/
```
<font color="red">分发完成后修改对应文件 zookeeper-3.4.10/zkData/myid 中对应的值 ,<br>
其中 myid 文件中的值与 server.3=hd002:2888:3888,server.3中的3对应
</font>


##### <font color="red"> 注意</font> 
``` bash 
[1]scp -r /home/admin/module/zookeeper-3.4.10 hd002:~/module/
[2]scp -r /home/admin/module/zookeeper-3.4.10/ hd002:~/module/
[1],[2]效果一样

<1>/rsync -rvl /home/admin/module/zookeeper-3.4.10 hd002:~/module/
<2>rsync -rvl /home/admin/module/zookeeper-3.4.10/ hd002:~/module/
<1><2> 是不同的
<1>中是将整个目录和目录下所有文件分发给hd002,会在hd002:~/module/下创建一个新的zookeeper-3.4.10目录并将文件同步过去
<2>是将整个目录下所有文件分发给hd001,不会将目录本身分发出去

```


#### 4.启动和查看命令
需要分别到每台服务进行启动
``` bash
# hd001
cd /home/admin/module/zookeeper-3.4.10
bin/zkServer.sh start
bin/zkServer.sh status
#状态结果
[admin@hd001 zookeeper-3.4.10]$ bin/zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /home/admin/module/zookeeper-3.4.10/bin/../conf/zoo.cfg
Mode: follower

# hd002
cd /home/admin/module/zookeeper-3.4.10
bin/zkServer.sh start
bin/zkServer.sh status
#状态结果(因本地已经重启过很多次 .此台机器第一次按顺序启动将会被选举为 leader的)
[admin@hd002 ~]$ ./module/zookeeper-3.4.10/bin/zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /home/admin/module/zookeeper-3.4.10/bin/../conf/zoo.cfg
Mode: follower
# hd003
cd /home/admin/module/zookeeper-3.4.10
bin/zkServer.sh start
bin/zkServer.sh status
#状态结果
[admin@hd003 ~]$ ./module/zookeeper-3.4.10/bin/zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /home/admin/module/zookeeper-3.4.10/bin/../conf/zoo.cfg
Mode: leader
```
