---
layout: post
title: Flink Streaming Write HDFS(parquet)
categories: [Flink]
description: Flink Streaming 写入hdfs文件对应hive中间层dwd
keywords: Flink HDFS
---

# Flink Streaming写入hdfs
- 需求背景:

   因公司大数据后台升级,将kafka中日志流数据,实时解析,
   写入指定hdfs中(parquet格式,snapy压缩),以此,往实时数仓方向转.

- 相关软件版本:
    ``` text
    Flink:1.11.1
    Java: 1.8
    Scala: 2.12
    Kafka:2.2.0
    Hive: 2.1.1
    Hadoop: 3.0.0
    ```
## 代码示例
- 主类
    
    ``` java
     
    package com.dz.bigdata.writer.hdfs;
            
        import java.util.Objects;
        import java.util.Properties;
        
        import org.apache.flink.api.common.serialization.SimpleStringSchema;
        import org.apache.flink.core.fs.Path;
        import org.apache.flink.streaming.api.datastream.DataStreamSource;
        import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
        import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
        import org.apache.flink.streaming.api.functions.sink.filesystem.OutputFileConfig;
        import org.apache.flink.streaming.api.functions.sink.filesystem.StreamingFileSink;
        import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer;
        import org.apache.parquet.hadoop.metadata.CompressionCodecName;
        import org.slf4j.Logger;
        import org.slf4j.LoggerFactory;
        
        import com.alibaba.fastjson.JSON;
        import com.alibaba.fastjson.JSONObject;
        import com.dz.bigdata.assinger.CommonEventTimeBucketAssigner;
        import com.dz.bigdata.common.Constants;
        import com.dz.bigdata.config.Config;
        import com.dz.bigdata.format.ParquetAvroWriters;
        import com.dz.bigdata.pojo.ServerStandardLog;
        import com.dz.bigdata.writer.BaseEnv;
        
        public class ServerLogHdfsWriter extends BaseEnv {
            private final static Logger LOG = LoggerFactory.getLogger(ServerLogHdfsWriter.class);
            private final static int maxParallelism = 6;
        
            public static void main(String[] args) throws Exception {
                Config.initArgs(args);
        
                String topic = Config.getArgsRequiredValue("topic");
                String groupId = Config.getArgsRequiredValue("groupId");
                String checkpointInterval = Config.getArgsRequiredValue("checkpointInterval");
                String location = Config.getArgsRequiredValue("path");
                int checkTime = Integer.parseInt(checkpointInterval);
        
                StreamExecutionEnvironment env = getStreamEnv(checkTime, "/flink/checkpoints/flink_hdfs_writer_" + topic);
        
                Properties props = new Properties();
                props.setProperty("bootstrap.servers", Config.getPropValue(Constants.KAFKA_SERVER));
                props.setProperty("group.id", groupId);
                env.setParallelism(maxParallelism);
        
                DataStreamSource<String> source =
                        env.addSource(new FlinkKafkaConsumer<>(topic, new SimpleStringSchema(), props));
        
                SingleOutputStreamOperator<ServerStandardLog> rowLog = source.map(line -> {
                    try {
                        JSONObject jsonObject = JSON.parseObject(line);
                        if (jsonObject.get("data") != null) {
                            jsonObject.put("data", jsonObject.getJSONObject("data"));
                        }
                        ServerStandardLog serverStandardLog =
                                JSON.parseObject(jsonObject.toJSONString(), ServerStandardLog.class);
                        return serverStandardLog;
        
                    } catch (Exception e) {
                        LOG.error("get log msg exception  ", e);
                        e.printStackTrace();
                        return null;
                    }
                }).filter(Objects::nonNull);
                OutputFileConfig config =
                        OutputFileConfig.builder().withPartSuffix("." + topic.trim() + ".snappy.parquet").build();
        
                StreamingFileSink<ServerStandardLog> sink = StreamingFileSink
                                                             .forBulkFormat(new Path("hdfs://" + location),
                                                                     ParquetAvroWriters.forReflectRecord(ServerStandardLog.class,
                                                                             CompressionCodecName.SNAPPY))
                                                             .withBucketAssigner(new CommonEventTimeBucketAssigner<>("dt=0%s%s",
                                                                     e-> e.getReqTime()))
                                                             .withOutputFileConfig(config).build();
        
                if(!"dev".equals(Config.getProfile())){
                    rowLog.addSink(sink);
                }else{
                    rowLog.print();
                }
        
                env.execute("server_log_to_hdfs_dwd_"+topic);
            }
        }
    
    public class BaseEnv {
    
        public static StreamExecutionEnvironment getStreamEnv(int checkpointTime, String checkpointPath) {
            StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
            env.enableCheckpointing(checkpointTime * 60 * 1000, CheckpointingMode.EXACTLY_ONCE);
            env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);
            env.getCheckpointConfig().setMinPauseBetweenCheckpoints(500);
            env.getCheckpointConfig().setCheckpointTimeout(checkpointTime * 2 * 60 * 1000);
    
            if (!"dev".equals(Config.getProfile())) {
                FsStateBackend fsStateBackend = new FsStateBackend("hdfs://" + checkpointPath);
                StateBackend rocksDBBackend = new RocksDBStateBackend(fsStateBackend, TernaryBoolean.TRUE);
                env.setStateBackend(rocksDBBackend);
                env.setRestartStrategy(RestartStrategies.fixedDelayRestart(2, Time.minutes(3)));
    //            env.setRestartStrategy(RestartStrategies.noRestart());
            } else {
                StateBackend fsStateBackend = new FsStateBackend("file://" + checkpointPath);
                env.setStateBackend(fsStateBackend);
                env.setRestartStrategy(RestartStrategies.noRestart());
            }
    
            env.getCheckpointConfig()
                    .enableExternalizedCheckpoints(CheckpointConfig.ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);
    
            return env;
        }
    }
    
    ```
  
- JavaBean 
    
    ```java 
    
    @Data
    public class ServerStandardLog {
        private String logId="";
        private String bline="";
        private String pline="";
        private Long reqTime;
        private String ip="";
        private String country="";
        private String province="";
        private String city="";
        private String uid="";
        private String ua="";
        private String imei="";
        private String imsi="";
        private String idfa="";
        private String idfv="";
        private String key="";
        private String subKey="";
        private Map<String,String> data = new HashMap<>();
    
        public ServerStandardLog() {
        }
    }
    ```
# 总结和问题
- 使用flink-Streaming写入时,对map类型支持.
- sink并行度不为1时,且多个topic往同一个目录写入时,需将sink文件名命名为不同的,否则会造成冲突,目前以topic区分,最好的方式可设置为随机数或uuid.
