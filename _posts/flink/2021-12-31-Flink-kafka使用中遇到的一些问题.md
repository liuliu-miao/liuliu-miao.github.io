---
layout: post
title: Flink消费Kafka时遇到的一些问题汇总
categories: [Flink,Kafka]
description:  Flink消费Kafka时遇到的一些问题汇总,不定时更新.
keywords: Flink-Kafka-Problems
---

# 简介
记录一些在使用flink消费kafka时,遇到一些优化问题和框架问题,不定时更新一些新的问题.

## 软件版本
``` markdown
flink: 1.13.2
kafka: 2.5.0
hadoop: 3.2.1
centos 7
```

## flink checkpoint警告日志
- 日志1:

    ``` log
    [DataStreamer for file /flink/checkpoints/xxx083408/097a5dc4d32c739760d33baea57f9679/shared/7832d571-5039-4c15-ae7f-99c874c64c55] WARN  org.apache.hadoop.hdfs.DataStreamer  - Caught exception
    java.lang.InterruptedException: null
            at java.lang.Object.wait(Native Method)
            at java.lang.Thread.join(Thread.java:1252)
            at java.lang.Thread.join(Thread.java:1326)
            at org.apache.hadoop.hdfs.DataStreamer.closeResponder(DataStreamer.java:986)
    ```
- 原因:

    hadoop自身BUG
    文档说明: https://issues.apache.org/jira/browse/HDFS-10429	

- 日志2:

    ``` log
    block BP-1644493283-172.34.242.137-1636089673161:blk_1076991017_3256384] WARN  org.apache.hadoop.hdfs.DataStreamer  - DataStreamer Exception
    java.nio.channels.ClosedByInterruptException: null
            at java.nio.channels.spi.AbstractInterruptibleChannel.end(AbstractInterruptibleChannel.java:202)
            at sun.nio.ch.SocketChannelImpl.write(SocketChannelImpl.java:477)
            at org.apache.hadoop.net.SocketOutputStream$Writer.performIO(SocketOutputStream.java:63)
            at org.apache.hadoop.net.SocketIOWithTimeout.doIO(SocketIOWithTimeout.java:142)
            at org.apache.hadoop.net.SocketOutputStream.write(SocketOutputStream.java:159)
            at org.apache.hadoop.net.SocketOutputStream.write(SocketOutputStream.java:117)
            at java.io.BufferedOutputStream.flushBuffer(BufferedOutputStream.java:82)
            at java.io.BufferedOutputStream.flush(BufferedOutputStream.java:140)
            at java.io.DataOutputStream.flush(DataOutputStream.java:123)
            at org.apache.hadoop.hdfs.DataStreamer.run(DataStreamer.java:775)
    ```

    - 原因: 
    flink论坛给出由于hadoop bug造成.
    link: https://issues.apache.org/jira/browse/FLINK-13228

## flink消费kafka事务超时问题
日志现象:
``` log 
Map -> Sink: late_log_to_kafka)-xxxxx-51, producerId=19575, epoch=9071] has been open for 304327 ms. This is close to or even exceeding the transaction timeout of 300000 ms.
```
- flink官网文档建议
增加超时时间: 
link: [https://nightlies.apache.org/flink/flink-docs-release-1.13/docs/connectors/datastream/kafka/#kafka-producers-and-fault-tolerance](https://nightlies.apache.org/flink/flink-docs-release-1.13/docs/connectors/datastream/kafka/#kafka-producers-and-fault-tolerance)


## flink-checkpoint失败优化
* 现象
flink exactly_one 消费kafka时,kafka峰值流量过大,且分区较多时(且机器资源有限),容易导致checkpoint-data过大,checkpoint失败或超时.
* 解决方法: 
增加 kafkaProducersPoolSize 大小,默认5.根据实际情况增大其值.

* 参考源码:

    flink-kafka-producer
    ``` java
    /**
        * Creates a FlinkKafkaProducer for a given topic. The sink produces its input to the topic. It
        * accepts a {@link KafkaSerializationSchema} and possibly a custom {@link
        * FlinkKafkaPartitioner}.
        *
        * @param defaultTopic The default topic to write data to
        * @param serializationSchema A serializable serialization schema for turning user objects into
        *     a kafka-consumable byte[] supporting key/value messages
        * @param producerConfig Configuration properties for the KafkaProducer. 'bootstrap.servers.' is
        *     the only required argument.
        * @param semantic Defines semantic that will be used by this producer (see {@link
        *     FlinkKafkaProducer.Semantic}).
        * @param kafkaProducersPoolSize Overwrite default KafkaProducers pool size (see {@link
        *     FlinkKafkaProducer.Semantic#EXACTLY_ONCE}).
        */
        public FlinkKafkaProducer(
                String defaultTopic,
                KafkaSerializationSchema<IN> serializationSchema,
                Properties producerConfig,
                FlinkKafkaProducer.Semantic semantic,
                int kafkaProducersPoolSize) {
            this(
                    defaultTopic,
                    null,
                    null, /* keyed schema and FlinkKafkaPartitioner */
                    serializationSchema,
                    producerConfig,
                    semantic,
                    kafkaProducersPoolSize);
        }
    ```
    Semantic.EXACTLY_ONCE
    ``` java
    /**
        * Semantics that can be chosen.
        * <li>{@link #EXACTLY_ONCE}
        * <li>{@link #AT_LEAST_ONCE}
        * <li>{@link #NONE}
        */
        public enum Semantic {

            /**
            * Semantic.EXACTLY_ONCE the Flink producer will write all messages in a Kafka transaction
            * that will be committed to Kafka on a checkpoint.
            *
            * <p>In this mode {@link FlinkKafkaProducer} sets up a pool of {@link
            * FlinkKafkaInternalProducer}. Between each checkpoint a Kafka transaction is created,
            * which is committed on {@link FlinkKafkaProducer#notifyCheckpointComplete(long)}. If
            * checkpoint complete notifications are running late, {@link FlinkKafkaProducer} can run
            * out of {@link FlinkKafkaInternalProducer}s in the pool. In that case any subsequent
            * {@link FlinkKafkaProducer#snapshotState(FunctionSnapshotContext)} requests will fail and
            * {@link FlinkKafkaProducer} will keep using the {@link FlinkKafkaInternalProducer} from
            * the previous checkpoint. To decrease the chance of failing checkpoints there are four
            * options:
            * <li>decrease number of max concurrent checkpoints
            * <li>make checkpoints more reliable (so that they complete faster)
            * <li>increase the delay between checkpoints
            * <li>increase the size of {@link FlinkKafkaInternalProducer}s pool
            */
            EXACTLY_ONCE,

            /**
            * Semantic.AT_LEAST_ONCE the Flink producer will wait for all outstanding messages in the
            * Kafka buffers to be acknowledged by the Kafka producer on a checkpoint.
            */
            AT_LEAST_ONCE,

            /**
            * Semantic.NONE means that nothing will be guaranteed. Messages can be lost and/or
            * duplicated in case of failure.
            */
            NONE
        }
    ```