---
layout: post
title: Flink Write Data to Clickhouse 遇到的一些问题.
categories: [Flink,Clickhouse]
description:  Flink Write Data to Clickhouse 遇到的一些问题.
keywords: Flink Clickhouse Problems
---

# 简介
由于公司业务和框架因素,目前将Clickhouse作为数仓唯一的存储选择.
现将一些个人在使用Flink写数据到Clickhouse遇到的一些问题做一些笔记.

## Clickhouse简介:
clickhouse是列式分布式存储系统,ClickHouse是一个用于联机分析(OLAP)的列式数据库管理系统(DBMS)。
[官网简介](https://clickhouse.com/docs/zh/)

## 问题一: 如果保证 Flink Sink to Clickhouse 有效一次
这个问题是刚到公司,分配给我的. 这里说一下我简单的思路和方案.
截至目前(2021年11月29日),前几天看到网上阿里已经实现了基于自身Clickhouse的有效一次性入库方案(类似于StreamSinkFile的二次提交)
但是需要结合clickhouse的源码修改才能完成

[阿里巴巴相关文档](https://mp.weixin.qq.com/s/8H2bxYUBmPzHl3Ae6WVWJg)

目前官方版本Clickhouse不具备事务.

### 解决思路
1. 将Clickhouse的表引擎改为:  ReplacingMergeTree (目前是MergeTree)
    - 优点: 满足幂等性,多次写入后数据最终能保证正确性.
    - 缺点: 当表数据很多时,ReplacingMergeTree对CPU和内存消耗很高.不适用于目前业务中.(目前业务将ods层数据都存到CH中,且是单节点.)

2. 将kf_partition,kf_offset写入ch的表中,使topic和表一一对应.offset由kafka维护,不一致时,删除ch中多余数据.
    - 优点: 能基本保证数据正确性.
    - 缺点: 不能使用Flink的自动重启机制,需要每次重启时,比对CH与Kafka的对应分区offset.
    - 缺点: 当CH负载过于高时,重启任务删除数据时,不能正常删除掉.数据准确性受限于CH负载.
3. 当任务重启时,使用kafka中的offset点直接启动任务,下一个小时通过hive或者具备事务性或幂等性的存储结构回插到CH(CH删除当前小时数据或分钟) 
    - 优点: 能保证数据准确性
    - 缺点: 资源消耗更多(需要额外的事务性分布式存储系统或集群资源消耗,目前公司不希望使用更多资源,就是"优化".)
    - 缺点: 当前重启小时CH中数据不够准确,需要下一个小时数据回插后数据才具备准确性.

最终使用了方案2.不要问为啥(成本控制?)

## 问题二: Clickhouse删除数据时提示 空间不足
1. 问题场景:

因为问题一解决方案2是基于保持clickhouse中数据的分区和offset与kafka的分区和offset一致,
当任务重启(失败重启或手动重启)时,回去校验offset一致情况 .
如果有不一致情况会将clickhouse中多余的数据删除其实保持一致.
同时:
users.xml 设置了同步修改数据属性.(因为是单节点所以设置1.多节点和副本情况设置2)
``` xml
 <mutations_sync>1</mutations_sync> 
``` 
导致了任务重启时删除多个按分区删除语句时提示空间不足:
日志如下:
``` log
dealCkAndKfOffsetNoEqual count=[8211] ;exec-sql=[alter table  db01.test_table  delete where toDate(`logdate`)='2021-11-30' and kf_partition=6 and kf_partition>=14152053083 and kf_offset<=14152061943]
ru.yandex.clickhouse.except.ClickHouseException: ClickHouse exception, code: 341, host: 172.34.6.84, port: 8123; Code: 341, e.displayText() = DB::Exception: Exception hap
pened during execution of mutation 'mutation_8222580.txt' with part '20210819_662942_690500_6_8222579' reason: 'Code: 243, e.displayText() = DB::Exception: Cannot reserve
 54.68 GiB, not enough space (version 21.8.11.4 (official build))'. This error maybe retryable or not. In case of unretryable error, mutation can be killed with KILL MUTA
TION query (version 21.8.11.4 (official build))

        at ru.yandex.clickhouse.except.ClickHouseExceptionSpecifier.specify(ClickHouseExceptionSpecifier.java:58)
        at ru.yandex.clickhouse.except.ClickHouseExceptionSpecifier.specify(ClickHouseExceptionSpecifier.java:28)
        at ru.yandex.clickhouse.ClickHouseStatementImpl.checkForErrorAndThrow(ClickHouseStatementImpl.java:876)
        at ru.yandex.clickhouse.ClickHouseStatementImpl.getInputStream(ClickHouseStatementImpl.java:616)
        at ru.yandex.clickhouse.ClickHouseStatementImpl.executeQuery(ClickHouseStatementImpl.java:117)
        at ru.yandex.clickhouse.ClickHouseStatementImpl.executeQuery(ClickHouseStatementImpl.java:100)
        at ru.yandex.clickhouse.ClickHouseStatementImpl.executeQuery(ClickHouseStatementImpl.java:95)
        at ru.yandex.clickhouse.ClickHouseStatementImpl.executeQuery(ClickHouseStatementImpl.java:90)
        at ru.yandex.clickhouse.ClickHouseStatementImpl.execute(ClickHouseStatementImpl.java:226)
        at flink_demo.utils.ClickhouseUtils.execSql(ClickhouseUtils.java:44)
        at flink_demo.utils.ClickhouseUtils.execSql(ClickhouseUtils.java:32)
        at flink_demo.utils.FlinkUtils.dealCkAndKfOffsetNoEqual(FlinkUtils.java:373)
        at flink_demo.FlinkWriteLogOdsCKQueue.main(FlinkWriteLogOdsCKQueue.java:52)
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:498)
        at org.apache.flink.client.program.PackagedProgram.callMainMethod(PackagedProgram.java:355)
        at org.apache.flink.client.program.PackagedProgram.invokeInteractiveModeForExecution(PackagedProgram.java:222)
        at org.apache.flink.client.ClientUtils.executeProgram(ClientUtils.java:114)
        at org.apache.flink.client.cli.CliFrontend.executeProgram(CliFrontend.java:812)
        at org.apache.flink.client.cli.CliFrontend.run(CliFrontend.java:246)
        at org.apache.flink.client.cli.CliFrontend.parseAndRun(CliFrontend.java:1054)
        at org.apache.flink.client.cli.CliFrontend.lambda$main$10(CliFrontend.java:1132)
        at java.security.AccessController.doPrivileged(Native Method)
        at javax.security.auth.Subject.doAs(Subject.java:422)
        at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1730)
        at org.apache.flink.runtime.security.contexts.HadoopSecurityContext.runSecured(HadoopSecurityContext.java:41)
        at org.apache.flink.client.cli.CliFrontend.main(CliFrontend.java:1132)
Caused by: java.lang.Throwable: Code: 341, e.displayText() = DB::Exception: Exception happened during execution of mutation 'mutation_8222580.txt' with part '20210819_662
942_690500_6_8222579' reason: 'Code: 243, e.displayText() = DB::Exception: Cannot reserve 54.68 GiB, not enough space (version 21.8.11.4 (official build))'. This error ma
ybe retryable or not. In case of unretryable error, mutation can be killed with KILL MUTATION query (version 21.8.11.4 (official build))

                at ru.yandex.clickhouse.except.ClickHouseExceptionSpecifier.specify(ClickHouseExceptionSpecifier.java:53)
        ... 28 more
ru.yandex.clickhouse.except.ClickHouseException: ClickHouse exception, code: 341, host: 172.34.6.84, port: 8123; Code: 341, e.displayText() = DB::Exception: Exception hap
pened during execution of mutation 'mutation_8222580.txt' with part '20210819_662942_690500_6_8222579' reason: 'Code: 243, e.displayText() = DB::Exception: Cannot reserve
 54.68 GiB, not enough space (version 21.8.11.4 (official build))'. This error maybe retryable or not. In case of unretryable error, mutation can be killed with KILL MUTA
TION query (version 21.8.11.4 (official build))
```

在执行时CH内存使用较高,如图
![内存使用图](https://i.loli.net/2021/11/30/mnYWsLOuB35Adrj.png)
同时参考了文章: [https://cloud.tencent.com/developer/article/1704570](https://cloud.tencent.com/developer/article/1704570) 

发现执行sql报错提示空间不足,是由因为对指定分区执行了 DELETE WHERE 条件删除，不在删除分区的分区文件，这些分区文件进入了 clone 流程,所以造成提示空间不足

如有其他关键问题.后期再继续更新.