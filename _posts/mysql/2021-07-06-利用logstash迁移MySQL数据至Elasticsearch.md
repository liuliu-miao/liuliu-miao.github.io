---
layout: post
title: 利用logstash迁移MySQL数据至Elasticsearch
categories: [MySql,Elasticsearch]
description: logstash move data from mysql to es
keywords: MySql Elasticsearch MySQL ES
---
# 利用logstash迁移MySQL数据至Elasticsearch
## 背景
公司某个表A，目前时mysql单库单表，对接Flink，主要用于读取和写入操作，后期考虑到量大后，对读写性能要求一定抗压，现将mysql迁移至Elasticsearch.
```
MySql Version: 5.6
Elasticsearch Version： 7.9.3
```

## 安装logstash
下载对应Elasticsearch版本的logstash
``` bash
cd ~/ && mkdir ~/opt && cd opt/
wget https://artifacts.elastic.co/downloads/logstash/logstash-7.9.3.tar.gz

tar -zxvf logstash-7.9.3.tar.gz

cd logstash-7.9.3 && mkdir mysql 

wget https://repo1.maven.org/maven2/mysql/mysql-connector-java/5.1.49/mysql-connector-java-5.1.49.jar -o mysql/mysql-connector-java-5.1.49.jar

cp config/logstash-sample.conf mysql/logstash-mysql-es.conf


bin/logstash-plugin install logstash-input-jdbc
bin/logstash-plugin install logstash-output-elasticsearch

vim mysql/logstash-mysql-es.conf

```
## 配置 logstash-mysql-es.conf

``` yml
input {
    jdbc {
        # 设置 MySql/MariaDB 数据库url以及数据库名称
        jdbc_connection_string => "jdbc:mysql://10.0.xx.xx:3306/dimension?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true"
        # 用户名和密码
        jdbc_user => "xxxxx"
        jdbc_password => "xxxxx"
        # 数据库驱动所在位置，可以是绝对路径或者相对路径
        jdbc_driver_library => "/home/elex/opt/logstash-7.9.3/mysql/mysql-connector-java-5.1.49.jar"
        # 驱动类名
        jdbc_driver_class => "com.mysql.jdbc.Driver"
        # 开启分页
        jdbc_paging_enabled => "true"
        # 分页每页数量，可以自定义
        jdbc_page_size => "500000"
        # 执行的sql文件路径
        #statement_filepath => "/usr/local/logstash-7.9.3/sync/foodie-items.sql"
        statement => "SELECT id,data,update_at,xxx,xxx FROM user"
        # 设置定时任务间隔  含义：分、时、天、月、年，全部为*默认含义为每分钟跑一次任务,配合statement中的语句可以做增量同步。
        #schedule => "* * * * *"
        # 是否开启记录上次追踪的结果，也就是上次更新的时间，这个会记录到 last_run_metadata_path 的文件
        use_column_value => true
        # 记录上一次追踪的结果值
        last_run_metadata_path => "/home/elex/opt/logstash-7.9.3/mysql/track_time"
        # 如果 use_column_value 为true， 配置本参数，追踪的 column 名，可以是自增id或者时间
        tracking_column => "update_at"
        # tracking_column 对应字段的类型
        tracking_column_type => "numeric"
        # 是否清除 last_run_metadata_path 的记录，true则每次都从头开始查询所有的数据库记录
        clean_run => false
        # 数据库字段名称大写转小写
        lowercase_column_names => false
    }
}
filter {
    
  mutate {
       remove_field => ["@timestamp"]
  }
    mutate {
       remove_field => ["@version"]
  }
}


output {
    elasticsearch {
        hosts => ["10.0.xx.xx:9200"]
        # 索引名字，必须小写
        index => "dimension-user"
        document_id => "%{id}"
        action => "index"

    }
}

```
## 执行迁移
执行过程中需要不要终止任务
也可放到后台运行
``` bash
./bin/logstash -f mysql/logstash-mysql-es.conf
# 或者
nohup ./bin/logstash -f mysql/logstash-mysql-es.conf > log.out 2>&1 & 
```

## 总结
1. 在迁移过程中，发现利用logstash迁移时，读取数据库时是全表扫描，对mysql压力极大。这个是问题。
2. ES7版本和6版本有很多不同，需要注意配置的使用。

1的解决方案： 可以通过手动程序，读取mysql数据，导入ES。
由于目前mysql数据量在百万，还能接受。所以就没有写程序。
如有遇到千万或上亿级数据时，最好不要使用logstash同步数据，或者测试通过在使用。