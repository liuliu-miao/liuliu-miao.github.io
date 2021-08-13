---
layout: post
title: Flink-Sink-ElasticSearch部分字段追加数据
categories: [Flink,ElasticSearch]
description: flink sink to elasticsearch add field data 
keywords: Flink ElasticSearch add field data
---

# Flink Flink-Sink-ElasticSearch部分字段追加数据

## 背景需求 
因flink消费kafka，其中数据需要进行补维操作，正常补维的数据都保存了最新属性数据到mysql。当任务正常流通时，从mysql拿取属性数据进行维度补充没有问题。
现有需求：当数据重放时：需要拿到对应时间状态的维度数据。

## 解决方案

### 方案一：
flink 双流join 关联用户ID，或其他关键值进行双流join完成。
[https://ci.apache.org/projects/flink/flink-docs-master/docs/dev/datastream/operators/joining/#interval-join](https://ci.apache.org/projects/flink/flink-docs-master/docs/dev/datastream/operators/joining/#interval-join)

方案一缺点：
1. 存在重放数据中没有属性数据时，关联不上的问题。当然这个也可以依赖外部储存解决
2. 增加程序的关联复杂度。

优点：
1. 利用flink join机制 ，降低对外部存储系统依赖
2. 高效。可以解决大部分的关联数据，但是有部分可能还是关联不上。

### 方案二:
通过保存历史版本数据，当回放数据时，判断当前数据与最新本的版本号是否一致。
不一致时，使用相对应的版本的属性数据。
优点：
1. 不存在数据丢失。
2. 相对程序来说简单一点

缺点：
1. 严重依赖外部存储。特别是数据量巨大的情况。
   
## 最终选择
综合考虑下来，选择了方案二
因公司相关程序都是跑在云（Flink on Yarn-EMR)上，没有自己的分布式存储系统。
本来想使用HBase,但是hbase强依赖hdfs，公司没有自建的HDFS集群，放弃。
选择使用ES。

说了这么多，好像都跟ES没有啥关系。。。。

选择ES来存储属性维度版本数据。
版本号就是属性数据中的timestamp字段（时间戳字段）
分别两个索性
1：存储版本的索性  user_versions
2：存储对应版本的数据索引 user_property_index 

其中索引user_versions专门用来存储用户数据的版本信息
eg:
user_versions
id=pid_uid
source=timestamps:[1628823667951]
``` json
 {
        "_index" : "user_versions",
        "_type" : "_doc",
        "_id" : "1_100007140",
        "_score" : 1.0,
        "_source" : {
          "timestamps" : [
            1628823667951,
            1628823667952,
            1628823668001,
            1628823855026,
            1628823856738,
            1628823873136,
            1628823958514,
            1628824061838
          ]
        }
      }
```
user_property_index
id=pid_uid_timestamp
eg:其余字段省略了。只列出关键数据
``` bash
GET user_property_index/_doc/1_100007140_1628824061838
```
```json
{
  "_index" : "user_property_index",
  "_type" : "_doc",
  "_id" : "1_100007140_1628824061838",
  "_version" : 1,
  "_seq_no" : 3360,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
  "data" : """{
        "pid": "1",
        "uid": "100007140",
        "timestamp": "1628824061838",
        "md5": "f83874b7742c78bd916cc430c44df527"
    }""",
    "update_at" : 1628841996,
    "id" : "1_100007140"
  }
```

当存储用户信息时，选择数据追加方式，
ES中有关于数据追加方式的介绍
[华为云文档](https://www.huaweicloud.com/articles/da7557ae10f7f6153d23b000ec2d4015.html)
简陋的[官网文档](https://www.elastic.co/guide/cn/elasticsearch/php/current/_updating_documents.html)（应该是我找的方式不对）

在flink中使用
``` java 
private static ElasticsearchSink<JSONObject> genESSink(List<HttpHost> httpHosts) {
        ElasticsearchSink.Builder esBuilder = new ElasticsearchSink.Builder(httpHosts, new ElasticsearchSinkFunction<JSONObject>() {

            public UpdateRequest createUserPropertySnapshotRequest(JSONObject element) {
                String pid = element.getString("pid");
                String uid = element.getString("uid");
                Long timestamp = element.getLong("timestamp");

                //保存property 信息
                String propertyId = pid + "_" + uid + "_" + timestamp;
                UpdateRequest updateReq = new UpdateRequest(ESClient.userPropertyIndex, propertyId);
                Map<String, Object> sourceMap = new HashMap<>();
                sourceMap.put("id", pid + "_" + uid);
                sourceMap.put("data", element.toJSONString());
                sourceMap.put("update_at", System.currentTimeMillis() / 1000);

                IndexRequest indexReq = new IndexRequest(ESClient.userPropertyIndex);
                indexReq.id(propertyId);
                indexReq.source(sourceMap);
                indexReq.timeout(TimeValue.timeValueSeconds(60));

                updateReq.doc(sourceMap)
                        .upsert(indexReq)
                        .timeout(TimeValue.timeValueSeconds(60));
                return updateReq;
            }

            @Override
            public void process(JSONObject element, RuntimeContext ctx, RequestIndexer indexer) {
                indexer.add(
                        createUserPropertySnapshotRequest(element),
                        createPropertyVersionRequest(element));
            }

            private UpdateRequest createPropertyVersionRequest(JSONObject element) {
                String pid = element.getString("pid");
                String uid = element.getString("uid");
                Long timestamp = element.getLong("timestamp");

                String versionId = pid + "_" + uid;
                //保存版本信息
                UpdateRequest updateReq = new UpdateRequest(ESClient.versionIndex, versionId);
                Map<String, Object> params = new HashMap<>();
                params.put("new_timestamp", timestamp);
                String idOrCode = "ctx._source.timestamps.add(params.new_timestamp)";
                Script script = new Script(ScriptType.INLINE, Script.DEFAULT_SCRIPT_LANG, idOrCode, params);
                updateReq.script(script);

                Map<String, Object> sourceMap = new HashMap<>();
                sourceMap.put("timestamps", new Long[]{timestamp});
                IndexRequest indexReq = new IndexRequest(ESClient.versionIndex);
                indexReq.id(versionId);
                indexReq.source(sourceMap);
                indexReq.timeout(TimeValue.timeValueSeconds(60));
                updateReq.upsert(indexReq)
                        .timeout(TimeValue.timeValueSeconds(60));

                return updateReq;
            }
        });

        esBuilder.setBulkFlushMaxActions(3);
      
        return esBuilder.build();
    }

    /**
     * 获取离时间参数版本最近的一个版本
     *
     * @param pid              product_id
     * @param uid              userId
     * @param versionTimestamp 时间参数版本号
     * @return user_property dataJson
     */
    public String getUserVersionProperty(String pid, String uid, Long versionTimestamp) {
        String versionId = pid + "_" + uid;
        try (RestHighLevelClient client = new RestHighLevelClient(clientBuilder)) {
            //获取最近版本
            GetRequest getRequest = new GetRequest(versionIndex, versionId);
            GetResponse resp = client.get(getRequest, RequestOptions.DEFAULT);
            if (resp.isExists()) {
                Object timestamps = resp.getSource().get("timestamps");
                ArrayList<Long> versionList = new ArrayList<>();
                if (timestamps != null && timestamps instanceof ArrayList) {
                    versionList = (ArrayList<Long>) timestamps;
                }
                long sub = versionTimestamp;
                long resVersion = -1L;
                HashSet<Long> versionSet = new HashSet<>();
                versionSet.addAll(versionList);
                for (Long version : versionSet) {
                    long tempSub = versionTimestamp - version;//
                    if (tempSub >= 0 && tempSub < sub) {//找到差值最小前一个版本
                        sub = tempSub;
                        resVersion = version;
                    }
                }
                if (resVersion != -1L) {
                    GetRequest proGet = new GetRequest(userPropertyIndex, pid + "_" + uid + "_" + resVersion);
                    resp = client.get(proGet, RequestOptions.DEFAULT);
                    if (resp.isExists()) {
                        Map<String, Object> source = resp.getSource();
                        if (source != null && !source.isEmpty()) {
                            return source.get("data") == null ? null : source.get("data").toString();
                        } else {
                            return null;
                        }
                    }
                }
            }
        } catch (Exception e) {
            log.error("getUserVersionProperty exception : ", e.getMessage(), e);
        }
        return null;
    }
```