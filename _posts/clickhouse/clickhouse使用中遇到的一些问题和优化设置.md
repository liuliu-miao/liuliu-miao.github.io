
# 简介
记录自己在使用clickhouse中遇到的一些问题和优化设置

## detach partition或drop partition时,size limit 50GB

修改 config.xml 不需要重启clickhoue-server.

``` xml
    <!-- 默认50GB  -->
    <!-- 400GB -->
    <max_table_size_to_drop>429496729600</max_table_size_to_drop>
    <!-- 200GB -->
    <max_partition_size_to_drop>214748364800</max_partition_size_to_drop>
```
文档参考:
[文档说明不用重启](https://clickhouse.com/docs/en/whats-new/changelog/2020/#bug-fix_49)
[Bug修复](https://clickhouse.com/docs/zh/whats-new/changelog/#bug-fix_8)
[设置文档-默认值](https://clickhouse.com/docs/en/operations/server-configuration-parameters/settings/?#max-table-size-to-drop)