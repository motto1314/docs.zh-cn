# StarRocks version 2.3

## 2.3.7

发布日期： 2022 年 12 月 30 日

### 问题修复

修复了如下问题：

- 当 StarRocks 表某些列支持为 NULL，但是当基于该表创建视图 时，视图中该列会被错误设置为 NOT NULL。 [#15749](https://github.com/StarRocks/starrocks/pull/15749)
- 导入时生成了新的 Tablet 版本，但是 FE 可能尚未感知新的 tablet 版本，因此此时 FE 下发的查询执行计划中依然要求 BE 读取该 Tablet 的历史版本。如果此时垃圾回收机制回收了该历史版本，则该时间点发起的查询无法读取该历史版本，从而查询报错 “Not found: get_applied_rowsets(version xxxx) failed tablet:xxx #version:x [xxxxxx]”。 [#15726](https://github.com/StarRocks/starrocks/pull/15726)
- 高频导入时 FE 占用较多内存。 [#15377](https://github.com/StarRocks/starrocks/pull/15377)
- 对于聚合和多表 JOIN 的查询，统计信息收集不准确，出现 CROSS JOIN，导致查询耗时过长。 [#12067](https://github.com/StarRocks/starrocks/pull/12067) [#14780](https://github.com/StarRocks/starrocks/pull/14780)

## 2.3.6

发布日期： 2022 年 12 月 22 日

### 功能优化

- Pipeline 执行引擎支持 INSERT INTO 语句。如果需要启用，则需要设置 FE 配置项 `enable_pipeline_load_for_insert` 为 `true`。 [#14723](https://github.com/StarRocks/starrocks/pull/14723)
- 优化主键模型 Compaction 阶段所占内存。[#13861](https://github.com/StarRocks/starrocks/pull/13861) 、[#13862](https://github.com/StarRocks/starrocks/pull/13862)

### 问题修复

修复了如下问题：

- 修复开启资源隔离后，多个资源组同时执行查询，可能会导致 BE 挂死的问题。[#14905](https://github.com/StarRocks/starrocks/pull/14905)
- 创建物化视图 `CREATE MATERIALIZED VIEW AS SELECT`，如果 `SELECT` 查询中未使用聚合函数，使用 GROUP BY，例如`CREATE MATERIALIZED VIEW test_view AS` `select a,b from test group by b,a order by a;`，则 BE 节点全部崩溃。[#13743](https://github.com/StarRocks/starrocks/pull/13743)
- 执行 INSERT INTO 高频导入至主键模型的表，进行数据变更后，立即重启 BE，重启缓慢。[#15128](https://github.com/StarRocks/starrocks/pull/15128)
- 如果环境中只安装 JRE 未安装 JDK，则重启 FE 后查询失败。修复后，在该环境无法成功重启 FE，会直接报错 `Error: JAVA_HOME can not be jre`，您需要在环境中安装 JDK。[#14332](https://github.com/StarRocks/starrocks/pull/14332)
- 查询导致 BE 崩溃。[#14221](https://github.com/StarRocks/starrocks/pull/14221)
- 设置 exec_mem_limit 时不支持使用表达式。[#13647](https://github.com/StarRocks/starrocks/pull/13647)
- 根据子查询结果创建同步刷新的物化视图，创建失败。[#13507](https://github.com/StarRocks/starrocks/pull/13507)
- 手动刷新 Hive 外表后列的 comment 注释被清空。[#13742](https://github.com/StarRocks/starrocks/pull/13742)
- 关联查询表 A 和 B 时，先计算右表 B 再计算左表 A，如果右表 B 特别大，在计算右表 B 过程中，左表 A 的数据执行了 Compaction，导致 BE 节点崩溃。[#14070](https://github.com/StarRocks/starrocks/pull/14070)
- 如果 Parquet 文件列名大小写敏感，此时查询条件中使用了 Parquet 文件中的大写列名，则查询结果为空。[#13860](https://github.com/StarRocks/starrocks/pull/13860) 、[#14773](https://github.com/StarRocks/starrocks/pull/14773)
- 批量导入时，Broker 的连接数过多，超过默认最大连接数，导致 Broker 连接断开，最终导入失败，并且报错 list path error。[#13911](https://github.com/StarRocks/starrocks/pull/13911)
- BE 负载很高时，资源组的监控指标 starrocks_be_resource_group_running_queries 统计错误。[#14043](https://github.com/StarRocks/starrocks/pull/14043)
- 如果查询语句使用 OUTER JOIN，可能会导致 BE 节点崩溃。[#14840](https://github.com/StarRocks/starrocks/pull/14840)
- 2.4 版本使用了异步物化视图后回滚到 2.3 版本，导致 FE 无法启动。[#14400](https://github.com/StarRocks/starrocks/pull/14400)
- 主键模型的表部分地方使用了delete_range，如果性能不佳可能会导致后续从 RocksDB 读取数据过慢，导致 CPU 资源占用率过高。[#15130](https://github.com/StarRocks/starrocks/pull/15130)

## 2.3.5

发布日期： 2022 年 11 月 30 日

### 功能优化

- Colocate Join 支持等值 Join。[#13546](https://github.com/StarRocks/starrocks/pull/13546)
- 解决高频导入时可能不断追加 WAL 记录而导致主键索引文件过大的问题。[#12862](https://github.com/StarRocks/starrocks/pull/12862)
- 优化 FE 后台线程在全量检测 tablet 时持有 DB 锁时间太长的问题。[#13070](https://github.com/StarRocks/starrocks/pull/13070)

### 问题修复

修复了如下问题：

- 修复在 UNION ALL 之上建立的 view 中，当 UNION ALL 输入列有常量 NULL，则 view 的 schema 错误，正常情况下列的类型是 UNION ALL 所有输入列的类型，但实际上是 NULL_TYPE 的问题。[#13917](https://github.com/StarRocks/starrocks/pull/13917)
- 修复 `SELECT * FROM ...` 和 `SELECT * FROM ... LIMIT ...` 查询结果不一致的问题。[#13585](https://github.com/StarRocks/starrocks/pull/13585)
- 修复 FE 同步外表 tablet 元数据时可能会覆盖本地 tablet 元数据导致从 Flink 导入数据失败的问题。[#12579](https://github.com/StarRocks/starrocks/pull/12579)
- 修复 Runtime Filter 中 null filter 在处理常量字面量的时候导致 BE 节点崩溃的问题。[#13526](https://github.com/StarRocks/starrocks/pull/13526)
- 修复执行 CTAS 报错的问题。[#12388](https://github.com/StarRocks/starrocks/pull/12388)
- 修复 audit log 中 pipeline engine 收集的 `ScanRows` 有误的问题。[#12185](https://github.com/StarRocks/starrocks/pull/12185)
- 修复查询压缩格式的 HIVE 数据返回结果不正确的问题。[#11546](https://github.com/StarRocks/starrocks/pull/11546)
- 修复一个 BE 节点崩溃后，查询超时并且响应缓慢的问题。[#12955](https://github.com/StarRocks/starrocks/pull/12955)
- 修复使用 Broker Load 导入数据时报错 Kerberos 认证失败的问题。[#13355](https://github.com/StarRocks/starrocks/pull/13355)
- 修复过多的 OR 谓词导致统计信息估算耗时过久的问题。[#13086](https://github.com/StarRocks/starrocks/pull/13086)
- 修复 Broker Load 导入包含大写列名的 ORC 文件 ( Snappy 压缩) 后 BE 节点崩溃的问题。[#12724](https://github.com/StarRocks/starrocks/pull/12724)
- 修复导出或者查询 Primary Key 表时间超过 30 分钟报错的问题。[#13403](https://github.com/StarRocks/starrocks/pull/13403)
- 修复使用 broker 备份至 HDFS 时，大数据量场景下备份失败的问题。[#12836](https://github.com/StarRocks/starrocks/pull/12836)
- 修复 `parquet_late_materialization_enable` 参数引起的读取 Iceberg 数据正确性的问题。[#13132](https://github.com/StarRocks/starrocks/pull/13132)
- 修复创建视图报错 `failed to init view stmt` 的问题。[#13102](https://github.com/StarRocks/starrocks/pull/13102)
- 修复通过 JDBC 连接 StarRock 执行 SQL 报错的问题。[#13526](https://github.com/StarRocks/starrocks/pull/13526)
- 修复分桶过多时查询时使用 tablet hint 导致查询超时的问题。[#13272](https://github.com/StarRocks/starrocks/pull/13272)
- 修复一个 BE 节点崩溃无法重新启动，并且此时建表后导入作业报错的问题。[#13701](https://github.com/StarRocks/starrocks/pull/13701)
- 修复创建物化视图导致全部 BE 节点崩溃的问题。[#13184](https://github.com/StarRocks/starrocks/pull/13184)
- 修复执行 ALTER ROUTINE LOAD 更新消费分区的 offset，可能会报错 `The specified partition 1 is not in the consumed partitions`，并且最终 Follower 崩溃的问题。[#12227](https://github.com/StarRocks/starrocks/pull/12227)

## 2.3.4

发布日期： 2022 年 11 月 10 日

### 功能优化

- 优化 Routine Load 创建失败时的报错提示。[#12204]( https://github.com/StarRocks/starrocks/pull/12204)
- 查询 Hive 时解析 CSV 数据失败后会直接报错。[#13013](https://github.com/StarRocks/starrocks/pull/13013)

### 问题修复

修复了如下问题：

- 修复 HDFS 文件路径带有 `()` 导致查询报错的问题。[#12660](https://github.com/StarRocks/starrocks/pull/12660)
- 修复子查询带有 LIMIT，并且使用 ORDER BY ... LIMIT ... OFFSET 对结果集进行排序后查询结果错误的问题。[#9698](https://github.com/StarRocks/starrocks/issues/9698)
- 修复查询 ORC 文件时大小写不敏感的问题。[#12724](https://github.com/StarRocks/starrocks/pull/12724)
- 修复 RuntimeFilter 没有正常关闭导致 BE 崩溃问题。[#12895](https://github.com/StarRocks/starrocks/pull/12895)
- 修复新增列后立即删除数据可能导致查询结果错误的问题。[#12907](https://github.com/StarRocks/starrocks/pull/12923)
- 修复当 StarRocks 和 MySQL 客户端不在同一局域网时，执行一次 KILL 无法成功终止 INSERT INTO SELECT 导入作业的问题。[#11879](https://github.com/StarRocks/starrocks/pull/11897)
- 修复 audit log 中 pipeline engine 收集的 `ScanRows` 有误的问题。[#12185](https://github.com/StarRocks/starrocks/pull/12185)

## 2.3.3

发布日期： 2022 年 9 月 27 日

### 问题修复

修复了如下问题：

- 修复查询文件格式为 Textfile 的 Hive 表，结果不准确的问题。[#11546](https://github.com/StarRocks/starrocks/pull/11546)
- 修复查询 Parquet 格式的文件时不支持查询嵌套的 ARRAY 的问题。[#10983](https://github.com/StarRocks/starrocks/pull/10983)
- 修复一个查询读取外部数据源和 StarRocks 表，或者并行查询读取外部数据源和 StarRocks 表，并且查询路由至一个资源组，则可能会导致查询超时的问题。[#10983](https://github.com/StarRocks/starrocks/pull/10983)
- 修复默认开启 Pipeline 执行引擎后，参数 parallel_fragment_exec_instance_num 变为 1，导致 INSERT INTO 导入变慢的问题。[#11462](https://github.com/StarRocks/starrocks/pull/11462)
- 修复表达式在初始阶段发生错误时可能导致 BE 停止服务的问题。[#11396](https://github.com/StarRocks/starrocks/pull/11396)
- 修复执行 ORDER BY LIMIT 时导致堆溢出的问题。 [#11185](https://github.com/StarRocks/starrocks/pull/11185)
- 修复重启 Leader FE，会造成 schema change 执行失败的问题。 [#11561](https://github.com/StarRocks/starrocks/pull/11561)

## 2.3.2

发布日期： 2022 年 9 月 7 日

### 新特性

- 支持延迟物化，提升小范围过滤场景下 Parquet 外表的查询性能。 [#9738](https://github.com/StarRocks/starrocks/pull/9738)
- 新增 SHOW AUTHENTICATION 语句，用于查询用户认证相关的信息。 [#9996](https://github.com/StarRocks/starrocks/pull/9996)

### 功能优化

- 查询 HDFS 集群里配置了分桶的 Hive 表时，支持设置是否递归遍历该分桶 Hive 表的所有数据文件。 [#10239](https://github.com/StarRocks/starrocks/pull/10239)
- 修改资源组类型 `realtime` 为 `short_query`。 [#10247](https://github.com/StarRocks/starrocks/pull/10247)
- 优化外表查询机制，在查询 Hive 外表时默认忽略大小写。 [#10187](https://github.com/StarRocks/starrocks/pull/10187)

### 问题修复

修复了如下问题：

- 修复 Elasticsearch 外表在多个 Shard 下时查询可能意外退出的问题。 [#10369](https://github.com/StarRocks/starrocks/pull/10369)
- 修复重写子查询为公用表表达式 (CTE) 时报错的问题。 [#10397](https://github.com/StarRocks/starrocks/pull/10397)
- 修复导入大批量数据时报错的问题。 [#10370](https://github.com/StarRocks/starrocks/issues/10370) [#10380](https://github.com/StarRocks/starrocks/issues/10380)
- 修复多个 Catalog 配置为相同 Thrift 服务地址时，删除其中一个 Catalog 会导致其他 Catalog 的增量元数据同步失效的问题。 [#10511](https://github.com/StarRocks/starrocks/pull/10511)
- 修复 BE 内存占用统计不准确的问题。 [#9837](https://github.com/StarRocks/starrocks/pull/9837)
- 修复完整克隆 (Full Clone) 后，查询主键模型表时报错的问题。[#10811](https://github.com/StarRocks/starrocks/pull/10811)
- 修复拥有逻辑视图的 SELECT 权限但无法查询的问题。[#10563](https://github.com/StarRocks/starrocks/pull/10563)
- 修复逻辑视图命名无限制的问题。逻辑视图的命名规范同数据库表的命名规范。[#10558](https://github.com/StarRocks/starrocks/pull/10558)

### 行为变更

- 增加 BE 配置项 `max_length_for_bitmap_function` （默认为 1000000 字节）和 `max_length_for_to_base64`（默认为 200000 字节），以控制 bitmap 函数和to_base64() 函数的输入值的最大长度。 [#10851](https://github.com/StarRocks/starrocks/pull/10851)

## 2.3.1

发布日期： 2022 年 8 月 22 日

### 功能优化

- Broker Load 支持将 Parquet 文件的 List 列转化为非嵌套 ARRAY 数据类型。[#9150](https://github.com/StarRocks/starrocks/pull/9150)
- 优化 JSON 类型相关函数（json_query、get_json_string 和 get_json_int）性能。[#9623](https://github.com/StarRocks/starrocks/pull/9623)
- 优化报错信息：查询 Hive、Iceberg、Hudi 外表时，如果查询的数据类型未被支持，系统针对相应的列报错。[#10139](https://github.com/StarRocks/starrocks/pull/10139)
- 降低资源组调度延迟，优化资源隔离性能。[#10122](https://github.com/StarRocks/starrocks/pull/10122)

### 问题修复

修复了如下问题：

- 查询 Elasticsearch 外表时，`limit` 算子下推问题导致返回错误结果。[#9952](https://github.com/StarRocks/starrocks/pull/9952)
- 使用 `limit` 算子查询 Oracle 外表失败。[#9542](https://github.com/StarRocks/starrocks/pull/9542)
- Routine Load 中，Kafka Broker 停止工作导致 BE 卡死。[#9935](https://github.com/StarRocks/starrocks/pull/9935)
- 查询与外表数据类型不匹配的 Parquet 文件导致 BE 停止工作。[#10107](https://github.com/StarRocks/starrocks/pull/10107)
- 外表 Scan 范围为空，导致查询阻塞超时。[#10091](https://github.com/StarRocks/starrocks/pull/10091)
- 子查询中包含 ORDER BY 子句时，系统报错。[#10180](https://github.com/StarRocks/starrocks/pull/10180)
- 并发重加载 Hive 元数据导致 Hive Metastore 挂起。[#10132](https://github.com/StarRocks/starrocks/pull/10132)

## 2.3.0

发布日期： 2022 年 7 月 29 日

### 新增特性

- 主键模型支持完整的 DELETE WHERE 语法。相关文档，请参见 [DELETE](../sql-reference/sql-statements/data-manipulation/DELETE.md#delete-与主键模型)。
- 主键模型支持持久化主键索引，基于磁盘而不是内存维护索引，大幅降低内存使用。相关文档，请参见[主键模型](../table_design/Data_model.md#使用说明-3)。
- 全局低基数字典优化支持实时数据导入，实时场景下字符串数据的查询性能提升一倍。
- 支持以异步的方式执行 CTAS，并将结果写入新表。相关文档，请参见 [CREATE TABLE AS SELECT](../sql-reference/sql-statements/data-definition/CREATE%20TABLE%20AS%20SELECT.md)。
- 资源组相关功能：
  - 支持监控资源组：可在审计日志中查看查询所属的资源组，并通过相关 API 获取资源组的监控信息。相关文档，请参见[监控指标](../administration/Monitor_and_Alert.md#监控指标)。
  - 支持限制大查询的 CPU、内存、或 I/O 资源；可通过匹配分类器将查询路由至资源组，或者设置会话变量直接为查询指定资源组。相关文档，请参见[资源隔离](../administration/Resource_Group.md)。
- 支持 JDBC 外表，可以轻松访问Oracle、PostgreSQL、MySQL、SQLServer、ClickHouse 等数据库，并且查询时支持谓词下推，提高查询性能。相关文档，请参见 [更多数据库（JDBC）的外部表](../data_source/External_table.md#更多数据库jdbc的外部表)。
- 【Preview】发布全新数据源 Connector 框架，支持创建外部数据目录（External Catalog），从而无需创建外部表，即可直接查询 Apache Hive™。相关文档，请参见[使用 Catalog 管理内部和外部数据](https://docs.starrocks.io/zh-cn/2.3/data_source/Manage_data)。
- 新增如下函数：
  - [window_funnel](../sql-reference/sql-functions/aggregate-functions/window_funnel.md)
  - [ntile](../sql-reference/sql-functions/Window_function.md)
  - [bitmap_union_count](../sql-reference/sql-functions/bitmap-functions/bitmap_union_count.md)、[base64_to_bitmap](../sql-reference/sql-functions/bitmap-functions/base64_to_bitmap.md)、[array_to_bitmap](../sql-reference/sql-functions/array-functions/array_to_bitmap.md)
  - [week](../sql-reference/sql-functions/date-time-functions/week.md)、[time_slice](../sql-reference/sql-functions/date-time-functions/time_slice.md)

### 功能优化

- 优化合并机制（Compaction），对较大的元数据进行合并操作，避免因数据高频更新而导致短时间内元数据挤压，占用较多磁盘空间。
- 优化导入 Parquet 文件和压缩文件格式的性能。
- 优化创建物化视图的性能，在部分场景下创建速度提升近 10 倍。
- 优化算子性能：
  - TopN，sort 算子。
  - 包含函数的等值比较运算符下推至 scan 算子时，支持使用 Zone Map 索引。
- 优化 Apache Hive™ 外表功能。
  - 当 Apache Hive™ 的数据存储采用 Parquet、ORC、CSV 格式时，支持 Hive 表执行 ADD COLUMN、REPLACE COLUMN 等表结构变更（Schema Change）。相关文档，请参见 [Hive 外部表](../data_source/External_table.md#hive-外表)。
  - 支持 Hive 资源修改 `hive.metastore.uris`。相关文档，请参见 [ALTER RESOURCE](../sql-reference/sql-statements/data-definition/ALTER%20RESOURCE.md)。
- 优化 Apache Iceberg 外表功能，创建 Iceberg 资源时支持使用自定义目录（Catalog）。相关文档，请参见 [Apache Iceberg 外表](../data_source/External_table.md#apache-iceberg-外表)。
- 优化 Elasticsearch 外表功能，支持取消探测 Elasticsearch 集群数据节点的地址。相关文档，请参见 [Elasticsearch 外部表](../data_source/External_table.md#elasticsearch-外部表)。
- 当 sum() 中输入的值为 STRING 类型且为数字时，则自动进行隐式转换。
- year、month、day 函数支持 DATE 数据类型。

### 问题修复

修复了如下问题：

- Tablet 过多导致 CPU 占用率过高的问题。
- 导致出现"fail to prepare tablet reader"报错提示的问题。
- FE 重启失败的问题。[#5642](https://github.com/StarRocks/starrocks/issues/5642 )、[#4969](https://github.com/StarRocks/starrocks/issues/4969 )、[#5580](https://github.com/StarRocks/starrocks/issues/5580)
- CTAS 语句中调用 JSON 函数时报错的问题。[#6498](https://github.com/StarRocks/starrocks/issues/6498)

### 其他

- 【Preview】提供集群管理工具 StarGo，提供集群部署、启停、升级、回滚、多集群管理等多种能力。相关文档，请参见[通过 StarGo 部署 StarRocks 集群](../administration/stargo.md)。
- 部署或者升级至 2.3 版本，默认开启 Pipeline 执行引擎，预期在高并发小查询、复杂大查询场景下获得明显的性能优势。如果使用 2.3 版本时遇到明显的性能回退，则可以通过设置 `SET GLOBAL enable_pipeline_engine = false;`，关闭 Pipeline 执行引擎。
- [SHOW GRANTS](../sql-reference/sql-statements/account-management/SHOW%20GRANTS.md) 语句兼容 MySQL语法，显示授权 GRANT 语句。
- 建议单个 schema change 任务数据占用内存上限memory_limitation_per_thread_for_schema_change( BE 配置项) 保持为默认值 2 GB，数据超过上限后写入磁盘。因此如果您之前调大该参数，则建议恢复为 2 GB，否则可能会出现单个 schema change 任务数据占用大量内存的问题。  
