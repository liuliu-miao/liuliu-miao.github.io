---
layout: post
title: zookeeper集群搭建
categories: [Hadoop,Zookeeper]
description: zookeeper集群搭建
keywords: zookeeper集群搭建, zookeeper
---

## 准备工作
* jdk环境
* zookeeper安装包(本文使用的是zookeeper-3.4.10 版本) 可到!(官网)[http://zookeeper.apache.org/releases.html]下载

## 步骤
1.将zookeeper-3.4.10解压到路径 /home/admin/module
``` bash 
tar -zxvf zookeeper-3.4.10.tar.gz -C /home/admin/module/
```

2.进入到zookeeper-3.4.10目录修改配置
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

完成以上配置后,将此目录分发到其他机器节点
``` bash
rsync -rvl /home/admin/module/zookeeper-3.4.10 hd002:~/module/
rsync -rvl /home/admin/module/zookeeper-3.4.10 hd003:~/module/
```
###### <font color="red"> 注意</font> 
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
