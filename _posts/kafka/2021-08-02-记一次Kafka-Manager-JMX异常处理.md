---
layout: post
title: 记一次Kafka-Manager JMX异常处理
categories: [Kafka]
description: Kafka-Manager JMX 127.0.0.1 Connection refused 
keywords:  Kafka-Manager Kafka JMX Connection refused 9999
---
# 记一次Kafka-Manager JMX异常处理

## 现象
直接上日志：
``` log
2021-08-02 03:49:15,548 - [ERROR] k.m.a.c.BrokerViewCacheActor - Failed to get broker metrics for BrokerIdentity(0,172.34.10.2,9999,false,true,Map(PLAINTEXT -> 9092))
java.rmi.ConnectException: Connection refused to host: 127.0.0.1; nested exception is: 
        java.net.ConnectException: Connection refused (Connection refused)
        at sun.rmi.transport.tcp.TCPEndpoint.newSocket(TCPEndpoint.java:623)
        at sun.rmi.transport.tcp.TCPChannel.createConnection(TCPChannel.java:216)
        at sun.rmi.transport.tcp.TCPChannel.newConnection(TCPChannel.java:202)
        at sun.rmi.server.UnicastRef.invoke(UnicastRef.java:131)
        at java.rmi.server.RemoteObjectInvocationHandler.invokeRemoteMethod(RemoteObjectInvocationHandler.java:235)
        at java.rmi.server.RemoteObjectInvocationHandler.invoke(RemoteObjectInvocationHandler.java:180)
        at com.sun.proxy.$Proxy5.newClient(Unknown Source)
        at javax.management.remote.rmi.RMIConnector.getConnection(RMIConnector.java:2430)
        at javax.management.remote.rmi.RMIConnector.connect(RMIConnector.java:308)
        at javax.management.remote.JMXConnectorFactory.connect(JMXConnectorFactory.java:270)
Caused by: java.net.ConnectException: Connection refused (Connection refused)
        at java.net.PlainSocketImpl.socketConnect(Native Method)
        at java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:350)
        at java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:206)
        at java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:188)
        at java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392)
        at java.net.Socket.connect(Socket.java:606)
        at java.net.Socket.connect(Socket.java:555)
        at java.net.Socket.<init>(Socket.java:451)
        at java.net.Socket.<init>(Socket.java:228)
2021-08-02 03:49:15,559 - [ERROR] k.m.j.KafkaJMX$ - Failed to connect to service:jmx:rmi:///jndi/rmi://172.34.56.7:9999/jmxrmi
java.rmi.ConnectException: Connection refused to host: 127.0.0.1; nested exception is: 
        java.net.ConnectException: Connection refused (Connection refused)
        at sun.rmi.transport.tcp.TCPEndpoint.newSocket(TCPEndpoint.java:623)
        at sun.rmi.transport.tcp.TCPChannel.createConnection(TCPChannel.java:216)
        at sun.rmi.transport.tcp.TCPChannel.newConnection(TCPChannel.java:202)
        at sun.rmi.server.UnicastRef.invoke(UnicastRef.java:131)
        at java.rmi.server.RemoteObjectInvocationHandler.invokeRemoteMethod(RemoteObjectInvocationHandler.java:235)
        at java.rmi.server.RemoteObjectInvocationHandler.invoke(RemoteObjectInvocationHandler.java:180)
        at com.sun.proxy.$Proxy5.newClient(Unknown Source)
        at javax.management.remote.rmi.RMIConnector.getConnection(RMIConnector.java:2430)
        at javax.management.remote.rmi.RMIConnector.connect(RMIConnector.java:308)
        at javax.management.remote.JMXConnectorFactory.connect(JMXConnectorFactory.java:270)
Caused by: java.net.ConnectException: Connection refused (Connection refused)
        at java.net.PlainSocketImpl.socketConnect(Native Method)
        at java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:350)
        at java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:206)
        at java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:188)
        at java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392)
        at java.net.Socket.connect(Socket.java:606)
        at java.net.Socket.connect(Socket.java:555)
        at java.net.Socket.<init>(Socket.java:451)
        at java.net.Socket.<init>(Socket.java:228)

```
启动时已经启动了JMX_PORT端口监控
``` bash
env JMX_PORT=9999 bin/kafka-server-start.sh -daemon ./config/server.properties &
```

## 原因： 
172.34.56.7，172.34.10.2为broker地址，其中 Connection refused to host: 127.0.0.1 中的127.0.0.1是Broker的JMX上报的地址，当kafka-manager去127.0.0.1连接JMX获取数据时，被拒绝了。实际上应该是去broker的ip:172.34.56.7，172.34.10.2中去获取。

网上找了很多帖子，发现配置kafka的conf
-Djava.rmi.server.hostname=172.34.56.7
-Dcom.sun.management.jmxremote.local.only=false

这种方式可行，但是想了想不太友好。
受此篇文章启发：[https://blog.csdn.net/chenchaofuck1/article/details/51558995](https://blog.csdn.net/chenchaofuck1/article/details/51558995)
通过hostname -i 发现
返回了127.0.0.1
猜测JMX上报时，java.rmi.server.hostname 地址上报的应该是hostname -i得到的值

## 解决方法：
修改/etc/hosts
增加 
172.34.56.7 `hostname`

``` bash
cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost6 localhost6.localdomain6
172.34.56.7  ip-172-34-56-7.compute.internal
```
再hostname -i 返回了机器的内网地址。
然后重启kafka-broker，kafka-manager可以正常监控了。

## 附录：hostname相关知识
[https://www.huaweicloud.com/articles/e06cc282064a70fb8101ca48944ab64a.html](https://www.huaweicloud.com/articles/e06cc282064a70fb8101ca48944ab64a.html)

对于-Djava.rmi.server.hostname orcale-jdk文档说明：
[https://docs.oracle.com/javase/9/management/monitoring-and-management-using-jmx-technology.htm#JSMGM-GUID-F08985BB-629A-4FBF-A0CB-8762DF7590E0](https://docs.oracle.com/javase/9/management/monitoring-and-management-using-jmx-technology.htm#JSMGM-GUID-F08985BB-629A-4FBF-A0CB-8762DF7590E0)