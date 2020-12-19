---
layout: post
title: Flink Streaming Write Hive Table
categories: [Flink]
description: Flink Streaming 写入hive表中
keywords: Flink Hive
---

# Flink Streaming写入Hive表
- 需求背景:

   因公司大数据后台升级,将kafka中日志流数据,实时解析,
   写入指定hive table不同分区中,以此,往实时数仓方向转.
   (在写入hdfs基础上更一步,因为写入hdfs-dwd层需要手动去增加分区,直接写入hive中更直接)

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
```java 
package com.dz.bigdata.writer.hive;

import java.time.Instant;
import java.time.LocalDateTime;
import java.time.ZoneId;
import java.time.ZoneOffset;
import java.time.format.DateTimeFormatter;
import java.util.Objects;
import java.util.Optional;
import java.util.Properties;

import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.api.java.typeutils.RowTypeInfo;
import org.apache.flink.configuration.CoreOptions;
import org.apache.flink.connectors.hive.HiveOptions;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer;
import org.apache.flink.table.api.DataTypes;
import org.apache.flink.table.api.EnvironmentSettings;
import org.apache.flink.table.api.SqlDialect;
import org.apache.flink.table.api.Table;
import org.apache.flink.table.api.TableResult;
import org.apache.flink.table.api.TableSchema;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
import org.apache.flink.table.catalog.DataTypeFactory;
import org.apache.flink.table.catalog.hive.HiveCatalog;
import org.apache.flink.table.functions.FunctionKind;
import org.apache.flink.table.functions.ScalarFunction;
import org.apache.flink.table.functions.TableFunction;
import org.apache.flink.table.functions.UserDefinedFunction;
import org.apache.flink.table.types.DataType;
import org.apache.flink.table.types.inference.TypeInference;
import org.apache.flink.types.Row;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import com.dz.bigdata.common.Constants;
import com.dz.bigdata.config.Config;
import com.dz.bigdata.pojo.ServerStandardLog;
import com.dz.bigdata.utils.ObjectUtils;
import com.dz.bigdata.writer.BaseEnv;

public class ServerLogHiveWriter extends BaseEnv {
    private final static Logger LOG = LoggerFactory.getLogger(ServerLogHiveWriter.class);
    private final static int maxParallelism = 6;

    public static void main(String[] args) throws Exception {
        Config.initArgs(args);

        String topic = Config.getArgsRequiredValue("topic");
        String groupId = Config.getArgsRequiredValue("groupId");
        String checkpointInterval = Config.getArgsRequiredValue("checkpointInterval");
        int checkTime = Integer.parseInt(checkpointInterval);

        StreamExecutionEnvironment env = getStreamEnv(checkTime, "/flink/checkpoints/flink_hive_writer_" + topic);

        Properties props = new Properties();
        props.setProperty("bootstrap.servers", Config.getPropValue(Constants.KAFKA_SERVER));
        props.setProperty("group.id", groupId);
        env.setParallelism(maxParallelism);

        DataStreamSource<String> source =
                env.addSource(new FlinkKafkaConsumer<>(topic, new SimpleStringSchema(), props));

        RowTypeInfo typeInfo = ObjectUtils.getTypeInfo(ServerStandardLog.class);

        SingleOutputStreamOperator<Row> rowLog = source.map(line -> {
            try {
                JSONObject jsonObject = JSON.parseObject(line);
                if (jsonObject.get("data") != null) {
                    jsonObject.put("data", jsonObject.getJSONObject("data"));
                }
                ServerStandardLog serverStandardLog =
                        JSON.parseObject(jsonObject.toJSONString(), ServerStandardLog.class);
                if (serverStandardLog != null) {
                    Object[] values = ObjectUtils.getFieldValues(serverStandardLog);
                    return Row.of(values);
                } else {
                    return null;
                }

            } catch (Exception e) {
                LOG.error("get log msg exception  ", e);
                e.printStackTrace();
                return null;
            }
        }).filter(Objects::nonNull).map(e -> e, typeInfo);

        EnvironmentSettings settings = EnvironmentSettings.newInstance().inStreamingMode().build();
        StreamTableEnvironment tbEnv = StreamTableEnvironment.create(env, settings);
        tbEnv.getConfig().getConfiguration().set(CoreOptions.DEFAULT_PARALLELISM, maxParallelism);
        tbEnv.getConfig().getConfiguration().set(HiveOptions.TABLE_EXEC_HIVE_INFER_SOURCE_PARALLELISM, true);
        tbEnv.getConfig().getConfiguration()
                .set(HiveOptions.TABLE_EXEC_HIVE_INFER_SOURCE_PARALLELISM_MAX, maxParallelism);
        if ("dev".equals(Config.getProfile())) {
            Table cslTable = tbEnv.fromDataStream(rowLog);
            TableSchema schema = cslTable.getSchema();
            Optional<DataType> data = schema.getFieldDataType("data");
            tbEnv.createTemporaryView("TmpLog", rowLog);
            Table table = tbEnv.sqlQuery("SELECT data as dataMap FROM TmpLog");
            TableResult result = table.execute();
            result.print();
        } else {

            String tbName = "dwd.dwd_hkx_hkx_server_standard_log_direct";
            String defaultDatabase = "dwd";
            String hiveConfDir = "/etc/hive/conf"; // a local path
            String caName = defaultDatabase;

            HiveCatalog hive = new HiveCatalog(caName, defaultDatabase, hiveConfDir);
            tbEnv.registerCatalog(caName, hive);

            tbEnv.useCatalog(caName);
            tbEnv.getConfig().setSqlDialect(SqlDialect.HIVE);

            tbEnv.createTemporaryFunction("DTFormat", new DtFormatUDF());
            tbEnv.createTemporaryView("TmpLog", rowLog);

            tbEnv.useDatabase(defaultDatabase);

            String insertSql =
                    "INSERT INTO " + tbName + " SELECT logId as  log_id,bline,pline,reqTime as req_time,ip,country,"
                            + "province,city,uid,ua,imei,imsi,idfa,idfv,key,subKey as sub_key,data,"
                            + " DTFormat(reqTime,'0yyyyMMddHH') as dt " + " FROM TmpLog";

            tbEnv.executeSql(insertSql);
        }

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
- DtFormatUDF,ObjectUtils类
```java 
package com.dz.bigdata.writer.hive;

import java.time.Instant;
import java.time.LocalDateTime;
import java.time.ZoneOffset;
import java.time.format.DateTimeFormatter;

import org.apache.flink.table.functions.ScalarFunction;

public class DtFormatUDF extends ScalarFunction {
    public String eval(Long time, String format) {
        return LocalDateTime.ofInstant(Instant.ofEpochMilli(time), ZoneOffset.ofHours(8))
                       .format(DateTimeFormatter.ofPattern(format));
    }
}
package com.dz.bigdata.utils;

import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.lang.reflect.Modifier;
import java.util.Arrays;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

import org.apache.commons.lang3.StringUtils;
import org.apache.flink.api.common.typeinfo.TypeInformation;
import org.apache.flink.api.common.typeinfo.Types;
import org.apache.flink.api.java.typeutils.RowTypeInfo;

public class ObjectUtils {

    public static RowTypeInfo getTypeInfo(Class<?> cls){

        List<Field> declaredFields = Arrays.stream(cls.getDeclaredFields()).filter(e -> !Modifier.isStatic(e.getModifiers()))
                                      .collect(Collectors.toList());
        int fieldLength = declaredFields.size();
        String[] fieldNames = new String[fieldLength];
        TypeInformation[] typeInformation = new TypeInformation[fieldLength];

        for (int i = 0; i < declaredFields.size(); i++) {
            Field field = declaredFields.get(i);
            fieldNames[i] =  field.getName();
            Class<?> type = field.getType();
            if(type == Long.class){
                typeInformation[i] = Types.LONG;
            }else if(type == Map.class){
                typeInformation[i] = Types.MAP(Types.STRING,Types.STRING);
            }else {
                typeInformation[i] = Types.STRING;
            }
        }
        RowTypeInfo outputType = new RowTypeInfo(typeInformation,fieldNames);
        return outputType;
    }
    public static Object[] getFieldValues(Object obj){
        Class<?> csClass = obj.getClass();
         List<Field> declaredFields =
                 Arrays.stream(csClass.getDeclaredFields()).filter(e -> !Modifier.isStatic(e.getModifiers()))
                                                                     .collect(Collectors.toList());
        Object[] values = new Object[declaredFields.size()];
        int index = 0;
        try{
            for (Field field : declaredFields) {
                if(Modifier.isStatic(field.getModifiers())){
                    continue;
                }
                String fieldName = field.getName();
                Method method = csClass.getMethod("get" + StringUtils.upperCase(fieldName.substring(0, 1)) + fieldName.substring(1));
                Object value = method.invoke(obj);
                values[index]=value;
                index++;
            }
        }catch (Exception e){
            e.printStackTrace();
        }

        return values;
    }
}

```

# 总结和问题.
- 写入hive时并行度为1,不能设置,导致写入速度很慢,公司部分日志数据量过大,导致消息积压不能消费完成
 (1.11.x版本存在这个问题,据论坛显示,1.12后续版本将解决此问题),所以目前还是使用写入hdfs-dwd层的方式运行中

- 因为flink转为临时表是map类型字段转换的不是对应hive Map,是Any,导致一开始写入失败,解决方案:手动指定临时表各个字段数据类型.ObjectUtils类做此操作