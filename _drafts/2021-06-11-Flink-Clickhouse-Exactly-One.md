---
layout: post
title: Flink-Clickhouse-Exactly-One解决方案（任有数据丢失风险，仅仅做记录用）
categories: [Flink,Clickhouse]
description: Flink read kafka write Clickhouse Exactly-One 
keywords: Flink Clickhouse Exactly-One
---
# 前言

了解Clickhouse的都知道.
1. Clickhouse是一个高性能快速写入和查询的列式存储系统,对于数据的修改和删除,官方是非常不建议的,
    官方网站定义修改为 mutation(突变)
    [update更新语句官网](https://clickhouse.tech/docs/en/sql-reference/statements/alter/update/)
    ``` 
    Manipulates data matching the specified filtering expression. Implemented as a mutation.
    ```
2. Clickhouse的Java-Api或其他Api写入clickhouse时底层是通过发送http接口,实现数据的写入过程,这是个异步的过程.

综上两点就造成了,Flink写入Clickhouse时不具备事务,而且在Clickhouse表在使用表引擎不具备合并引擎的时候,数据一致性不能通过官网API得以保证.
此处是Flink的checkpoint机制没法正确保存正确的Offset信息.
底层代码:
``` xml
<!-- clickhouse jar包版本--> 
<dependency>
    <groupId>ru.yandex.clickhouse</groupId>
    <artifactId>clickhouse-jdbc</artifactId>
    <version>0.2.4</version>
</dependency>
```
``` java
//ClickHouseStatementImpl 606行开始
requestEntity = applyRequestBodyCompression(requestEntity);

        HttpEntity entity = null;
        try {
            uri = followRedirects(uri);
            HttpPost post = new HttpPost(uri);
            post.setEntity(requestEntity);

            HttpResponse response = client.execute(post);
            entity = response.getEntity();
            checkForErrorAndThrow(entity, response);

            InputStream is;
            if (entity.isStreaming()) {
                is = entity.getContent();
            } else {
                FastByteArrayOutputStream baos = new FastByteArrayOutputStream();
                entity.writeTo(baos);
                is = baos.convertToInputStream();
            }
            return is;
        } catch (ClickHouseException e) {
            throw e;
        } catch (Exception e) {
            log.info("Error during connection to {}, reporting failure to data source, message: {}", properties, e.getMessage());
            EntityUtils.consumeQuietly(entity);
            log.info("Error sql: {}", sql);
            throw ClickHouseExceptionSpecifier.specify(e, properties.getHost(), properties.getPort());
        }
```

# Flink Streaming Write Data to Clickhouse Exactly-One解决方案

- 背景:
    公司因历史原因和数据原因,存在的Clickhouse表的表引擎时MergeTree(这里叫做A表吧),有历史Flink任务通过读取Kafka数据,最终将数据写入A表中.同时也有任务将数据sink到Kafka中.
    当Flink任务重启或Clickhouse集群因不同原因重启时,导致Flink任务重启时,会对已存入Clickhouse部分的数据进行重复消费问题.
    重复原因: Flink在checkpoint未完成down了,但是数据已经写入到Clikchouse中,此时Offset还未提交.下次启动后,会从未提交的Offset部分开始消费.

- 相关软件版本:
    ``` text
    Flink:1.10.1
    Java: 1.8
    Kafka:2.2.0
    Clickhouse:  21.4.3.21
    ```
- 解决方案
本次方案由个人提出,不涉及公司机密.
1. 修改Clickhouse表引擎为:ReplacingMergeTree
2. 在A表中新增列(kf_partition,kf_offset)保存消费的offset,当重启时,读取每个分区最大的offset,设置到consumer中.保证数据不重复消费.

## 1.修改Clickhouse表引擎为:ReplacingMergeTree 优缺点
* 优点: 
1. 从数据角度解决,不必关心事务性问题.
2. 修复速度极快
* 缺点:
1. ReplacingMergeTree引擎会定期才合并相同数据.对及时数据准确性差.
2. ReplacingMergeTree数据合并时,对Clickhouse产生一定的压力
3. 历史数据问题.

根据优缺点同公司业务结合,最终放弃了此方案.

## 2.在A表中新增列(kf_partition,kf_offset)保存消费的offset
* 优点: 
1. 兼容了历史数据,不涉及表引擎改动
2. Flink任务重启时不必从checkpoint中启动
* 缺点:
1. 开发工作量大
2. Flink不能使用自动重启策略,需要单独有自动监控和重启服务(监本)进行自动化辅助
## 代码示例
注意: 不要启用Flink的自动重启策略
在A表中增加两列 kf_partition, kf_offset
``` sql
ALTER TABLE db.A ADD COLUMN kf_partition Int32

ALTER TABLE db.A ADD COLUMN kf_offset Int64 AFTER kf_partition
```
``` java
flinkEnv.setRestartStrategy(RestartStrategies.noRestart());
FlinkKafkaConsumer<String> kafkaConsumer = KafkaUtils.getKafkaConsumer(Collections.singletonList(topic), offsetInfo, parameterTool, groupId);

```

```java    
   public static FlinkKafkaConsumer<String> getKafkaConsumer(List<String> topics, Map<KafkaTopicPartition, Long> offsetInfo, ParameterTool param, String groupId) {
        Properties kfProps = KafkaUtils.getConsumerProps(param.get("bootstrap.servers"), groupId);
        FlinkKafkaConsumer<String> consumer = new FlinkKafkaConsumer<>(topics, new ConsumerSerializationSchema(), kfProps);
        if (!offsetInfo.isEmpty()) {
            consumer.setStartFromSpecificOffsets(offsetInfo);
        } else {
            String offset_time = param.get("offset_time");//指定了offset时间时,使用offset_time,否则使用 groupOffset
            if(StringUtils.isNotBlank(offset_time)){
                DateTimeFormatter dtf = DateTimeFormatter.ofPattern("yyyy-MM-ddHH:mm:ss");
                LocalDateTime localDateTime = LocalDateTime.parse(offset_time, dtf);
                long milliSecond = localDateTime.toInstant(ZoneOffset.ofHours(0)).toEpochMilli();
                //写入kafka的时间....
                consumer.setStartFromTimestamp(milliSecond);
            }else {
                consumer.setStartFromGroupOffsets();
            }

        }
        return consumer;
    }

    public static Map<KafkaTopicPartition, Long> getOffsetInfo(Properties parameters, String topic, String envProfile) throws Exception {
        Map<KafkaTopicPartition, Long> offsetInfo = new HashMap<>();
        ClickHouseClient ch = new ClickHouseClient(parameters);
        KFTopicCKTBRelation ttr = KFTopicCKTBRelation.valueOfTopic(topic);
        if (ttr == null) {
            System.out.printf("do not find topic and ck table relation topic=[%s]", topic);
            return offsetInfo;
        }
        String[] ckTables = ttr.ckTables;
        String partitionCol = ttr.partitionCol;
        String sql = getTablesOffsetSql(envProfile, partitionCol, ttr.colFormat, ckTables);
        System.out.printf("getOffsetSql=[%s]\n", sql);
        List<Map<String, String>> list = ch.querySql(sql);
        if (!list.isEmpty()) {
            for (Map<String, String> map : list) {
                long kfOffset = Long.parseLong(map.get("kf_offset"));
                if (kfOffset > 0L) {
                    offsetInfo.put(new KafkaTopicPartition(topic, Integer.parseInt(map.get("kf_partition"))), kfOffset + 1);
                }
            }
        }

        System.out.printf("start_get_offset,time=[%s]", System.currentTimeMillis());
        return offsetInfo;

    }

    public static String getTablesOffsetSql(String profile, String partitionCol, String colFormat, String... tables) {
        int dbSum = 10;
        if (Arrays.asList("win", "locale").contains(profile)) {
            dbSum = 2;
        }
        DateTimeFormatter dtf = DateTimeFormatter.ofPattern(colFormat);
        String today = LocalDateTime.now().format(dtf);
        String yesterday = LocalDateTime.now().minusDays(1).format(dtf);
        String sql = "select a.kf_partition as kf_partition, max(a.kf_offset) as kf_offset from (";

        StringBuilder subSql = new StringBuilder();
        for (int i = 1; i < dbSum + 1; i++) {
            for (int j = 0; j < tables.length; j++) {
                String table = tables[j];
                String tbName = "bi_" + i + "." + table;
                if (Arrays.asList("win", "locale").contains(profile)) {
                    subSql.append(String.format("select kf_partition,max(kf_offset) as kf_offset from %s where `%s` <='%s' group by kf_partition", tbName, partitionCol, today));
                } else {
                    subSql.append(String.format("select kf_partition,max(kf_offset) as kf_offset from %s where `%s` in (%s) group by kf_partition", tbName, partitionCol, "'" + today + "','" + yesterday + "'"));
                }
                if (j < tables.length - 1 || i < dbSum) {
                    subSql.append(" union all ");
                }
            }
        }
        sql += subSql + " ) a group by kf_partition ";

        return sql;
    }
    
```

# 总结和问题.
- 按类拆分,逐个解决
将原有任务(Flink一个app中同事sink到Kafka和Clickhouse)进行任务拆分,拆分为具有事务性,和不具备事务性的两个任务.
- 充分利用Clickhouse快速插入和查询功能,实现实施数据查询.
- 对于不具备事务性的,可以借助外部存储系统将kafka-offset保存,在重启时,读取最新的offset