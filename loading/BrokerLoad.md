# 从 HDFS 或外部云存储系统导入数据

StarRocks 提供基于 MySQL 协议的 Broker Load 导入方式，帮助您从 HDFS 或外部云存储系统导入几十到数百 GB 的数据量。

Broker Load 是一种异步的导入方式。您提交导入作业以后，StarRocks 会异步地执行导入作业。您需要通过 [SHOW LOAD](/sql-reference/sql-statements/data-manipulation/SHOW%20LOAD.md) 语句或者 curl 命令来查看导入作业的结果。

Broker Load 支持一次导入多个数据文件，并且能够保证单次导入事务的原子性，即单次导入的多个数据文件都成功或者都失败，而不会出现部分导入成功、部分导入失败的情况。

Broker Load 还支持在导入过程中做数据的转换，具体请参见[导入过程中实现数据转换](/loading/Etl_in_loading.md)。

## 背景信息

在 StarRocks v2.4 及以前版本，Broker Load 需要借助 Broker 访问外部存储系统。语法里需要写 `WITH BROKER "<broker_name>"` 来指定使用哪组 Broker 组。Broker 是一个独立的无状态服务，封装了文件系统接口。通过 Broker，StarRocks 能够访问和读取外部存储系统上的数据文件，并利用自身的计算资源对数据文件中的数据进行预处理和导入。

自 StarRocks v2.5 起，Broker Load 不再需要借助 Broker 即可访问外部存储系统。语法里也不再需要提供 `broker_name`，但继续保留 `WITH BROKER` 关键字。

> **说明**
>
> 无 Broker 进程的导入在某些场景下会受限。比如如果您配置了多套 HA 或者多个 Kerberos 配置时，不支持无 Broker 进程的导入。这种情况下，您可以继续通过 Broker 进程执行导入。

如果您继续通过 Brokers 来执行导入，则必须确保您的 StarRocks 集群中已部署 Broker。您可以通过 [SHOW BROKER](/sql-reference/sql-statements/Administration/SHOW%20BROKER.md) 语句来查看集群中已经部署的 Broker。如果集群中没有部署 Broker，请参见[部署 Broker 节点](/administration/deploy_broker.md)完成 Broker 部署。

## 支持的数据文件格式

Broker Load 支持如下数据文件格式：

- CSV

- Parquet

- ORC

> **说明**
>
> 对于 CSV 格式的数据，StarRocks 支持设置长度最大不超过 50 个字节的 UTF-8 编码字符串作为列分隔符，包括常见的逗号 (,)、Tab 和 Pipe (|)。

## 支持的外部存储系统

Broker Load 支持从如下外部存储系统导入数据：

- HDFS

- Amazon S3

- Google GCS

- 阿里云 OSS

- 腾讯云 COS

## 基本原理

提交导入作业以后，FE 会生成对应的查询计划，并根据目前 BE 的个数和待导入数据文件的大小，将查询计划分配给多个 BE 执行。每个 BE 负责执行一部分导入任务。BE 在执行过程中，会通过 Broker 拉取数据，并且会在对数据进行预处理之后将数据导入到 StarRocks 中。所有 BE 均完成导入后，由 FE 最终判断导入作业是否成功。

下图展示了 Broker Load 的主要流程：

![Broker Load 原理图](/assets/4.3-1-zh.png)

## 基本操作

### 创建导入作业

这里以导入 CSV 格式的数据为例介绍如何创建导入作业。有关如何导入其他格式的数据、以及 Broker Load 的详细语法和参数说明，请参见 [BROKER LOAD](/sql-reference/sql-statements/data-manipulation/BROKER%20LOAD.md)。

注意在 StarRocks 中，部分文字是 SQL 语言的保留关键字，不能直接用于 SQL 语句。如果想在 SQL 语句中使用这些保留关键字，必须用反引号 (`) 包含起来。参见[关键字](../sql-reference/sql-statements/keywords.md)。

#### 数据样例

1. 在 StarRocks 数据库 `test_db` 中创建 StarRocks 表。

   a. 创建一张名为 `table1` 的主键模型表。表包含 `id`、`name` 和 `score` 三列，分别代表用户 ID、用户名称和用户得分，主键为 `id` 列，如下所示：

   ```SQL
      MySQL [test_db]> CREATE TABLE `table1`
      (
          `id` int(11) NOT NULL COMMENT "用户 ID",
          `name` varchar(65533) NULL DEFAULT "" COMMENT "用户姓名",
          `score` int(11) NOT NULL DEFAULT "0" COMMENT "用户得分"
      )
          ENGINE=OLAP
          PRIMARY KEY(`id`)
          DISTRIBUTED BY HASH(`id`) BUCKETS 10;
      ```

   b. 创建一张名为 `table2` 的主键模型表。表包含 `id` 和 `city` 两列，分别代表城市 ID 和城市名称，主键为 `id` 列，如下所示：

   ```SQL
      MySQL [test_db]> CREATE TABLE `table2`
      (
          `id` int(11) NOT NULL COMMENT "城市 ID",
          `city` varchar(65533) NULL DEFAULT "" COMMENT "城市名称"
      )
          ENGINE=OLAP
          PRIMARY KEY(`id`)
          DISTRIBUTED BY HASH(`id`) BUCKETS 10;
   ```

2. 在本地文件系统中创建 CSV 格式的数据文件。

   a. 创建一个名为 `file1.csv` 的数据文件。文件一共包含三列，分别代表用户 ID、用户姓名和用户得分，如下所示：

   ```Plain
   1,Lily,23
   2,Rose,23
   3,Alice,24
   4,Julia,25
   ```

   b. 创建一个名为 `file2.csv` 的数据文件。文件一共包含两列，分别代表城市 ID 和城市名称，如下所示：

   ```Plain
   200,'北京'
   ```

3. 把创建好的数据文件 `file1.csv` 和 `file2.csv` 分别上传到 HDFS 集群的 `/user/starrocks/` 路径下、Amazon S3 存储空间 `bucket_s3` 里的 `input` 文件夹下、 Google GCS 存储空间 `bucket_gcs` 里的 `input` 文件夹下、阿里云 OSS 存储空间 `bucket_oss` 里的 `input` 文件夹下、以及腾讯云 COS 存储空间 `bucket_cos` 里的 `input` 文件夹下。

#### 从 HDFS 导入

可以通过如下语句，把 HDFS 集群 `/user/starrocks/` 路径下的 CSV 文件 `file1.csv` 和 `file2.csv` 分别导入到 StarRocks 表 `table1` 和 `table2` 中：

```SQL
LOAD LABEL test_db.label1
(
    DATA INFILE("hdfs://<hdfs_host>:<hdfs_port>/user/starrocks/file1.csv")
    INTO TABLE table1
    COLUMNS TERMINATED BY ","
    (id, name, score)

    DATA INFILE("hdfs://<hdfs_host>:<hdfs_port>/user/starrocks/file2.csv")
    INTO TABLE table2
    COLUMNS TERMINATED BY ","
   (id, city)
)
WITH BROKER
(
    "username" = "hdfs_username",
    "password" = "hdfs_password"
)
PROPERTIES
(
    "timeout" = "3600"
);
```

#### 从 Amazon S3 导入

可以通过如下语句，把 Amazon S3 存储空间 `bucket_s3` 里 `input` 文件夹内的 CSV 文件 `file1.csv` 和 `file2.csv` 分别导入到 StarRocks 表 `table1` 和 `table2` 中：

```SQL
LOAD LABEL test_db.label2
(
    DATA INFILE("s3a://bucket_s3/input/file1.csv")
    INTO TABLE table1
    COLUMNS TERMINATED BY ","
    (id, name, score)
    
    DATA INFILE("s3a://bucket_s3/input/file2.csv")
    INTO TABLE table2
    COLUMNS TERMINATED BY ","
    (id, city)
)
WITH BROKER
(
    "fs.s3a.access.key" = "xxxxxxxxxxxxxxxxxxxx",
    "fs.s3a.secret.key" = "yyyyyyyyyyyyyyyyyyyy",
    "fs.s3a.endpoint" = "s3.ap-northeast-1.amazonaws.com"
);
```

> **说明**
>
> - 由于 Broker Load 只支持通过 S3A 协议访问 AWS S3，因此当从 AWS S3 导入数据时，`DATA INFILE` 中传入的目标文件的 S3 URI，前缀必须将 `s3://` 修改为 `s3a://`。
> - 如果您的 Amazon EC2 实例上绑定的 IAM 角色可以访问您的 Amazon S3 存储空间，那么您不需要提供 `fs.s3a.access.key` 和 `fs.s3a.secret.key` 配置，留空即可。

#### 从 Google GCS 导入

可以通过如下语句，把 Google GCS 存储空间 `bucket_gcs` 里 `input` 文件夹内的 CSV 文件 `file1.csv` 和 `file2.csv` 分别导入到 StarRocks 表 `table1` 和 `table2` 中：

```SQL
LOAD LABEL test_db.label3
(
    DATA INFILE("s3a://bucket_gcs/input/file1.csv")
    INTO TABLE table1
    COLUMNS TERMINATED BY ","
    (id, name, score)
    
    DATA INFILE("s3a://bucket_gcs/input/file2.csv")
    INTO TABLE table2
    COLUMNS TERMINATED BY ","
    (id, city)
)
WITH BROKER
(
    "fs.s3a.access.key" = "xxxxxxxxxxxxxxxxxxxx",
    "fs.s3a.secret.key" = "yyyyyyyyyyyyyyyyyyyy",
    "fs.s3a.endpoint" = "storage.googleapis.com"
);
```

> **说明**
>
> 由于 Broker Load 只支持通过 S3A 协议访问 Google GCS，因此当从 Google GCS 导入数据时，`DATA INFILE` 中传入的目标文件的 GCS URI，前缀必须修改为 `s3a://`。

#### 从 阿里云 OSS 导入

可以通过如下语句，把阿里云 OSS 存储空间 `bucket_oss` 里 `input` 文件夹内的 CSV 文件 `file1.csv` 和 `file2.csv` 分别导入到 StarRocks 表 `table1` 和 `table2` 中：

```SQL
LOAD LABEL test_db.label4
(
    DATA INFILE("oss://bucket_oss/input/file1.csv")
    INTO TABLE table1
    COLUMNS TERMINATED BY ","
    (id, name, score)
    
    DATA INFILE("oss://bucket_oss/input/file2.csv")
    INTO TABLE table2
    COLUMNS TERMINATED BY ","
    (id, city)
)
WITH BROKER
(
    "fs.oss.accessKeyId" = "xxxxxxxxxxxxxxxxxxxxxxxxxx",
    "fs.oss.accessKeySecret" = "yyyyyyyyyyyyyyyyyyyy",
    "fs.oss.endpoint" = "oss-cn-zhangjiakou-internal.aliyuncs.com"
);
```

#### 从腾讯云 COS 导入

可以通过如下语句，把腾讯云 COS 存储空间 `bucket_cos` 里 `input` 文件夹内的 CSV 文件 `file1.csv` 和 `file2.csv` 分别导入到 StarRocks 表 `table1` 和 `table2` 中：

```SQL
LOAD LABEL test_db.label5
(
    DATA INFILE("cosn://bucket_cos/input/file1.csv")
    INTO TABLE table1
    COLUMNS TERMINATED BY ","
    (id, name, score)
    
    DATA INFILE("cosn://bucket_cos/input/file2.csv")
    INTO TABLE table2
    COLUMNS TERMINATED BY ","
    (id, city)
)
WITH BROKER
(
    "fs.cosn.userinfo.secretId" = "xxxxxxxxxxxxxxxxx",
    "fs.cosn.userinfo.secretKey" = "yyyyyyyyyyyyyyyy",
    "fs.cosn.bucket.endpoint_suffix" = "cos.ap-beijing.myqcloud.com"
);
```

#### 从 华为云 OBS 导入

可以通过如下语句，把华为云 OBS 存储空间 `bucket_obs` 里 `input` 文件夹内的 CSV 文件 `file1.csv` 和 `file2.csv` 分别导入到 StarRocks 表 `table1` 和 `table2` 中：

```SQL
LOAD LABEL test_db.label6
(
    DATA INFILE("obs://bucket_obs/input/file1.csv")
    INTO TABLE table1
    COLUMNS TERMINATED BY ","
    (id, name, score)
    
    DATA INFILE("obs://bucket_obs/input/file2.csv")
    INTO TABLE table2
    COLUMNS TERMINATED BY ","
    (id, city)
)
WITH BROKER "mybroker"
(
    "fs.obs.access.key" = "xxxxxxxxxxxxxxxxxxxxxxxxxx",
    "fs.obs.secret.key" = "yyyyyyyyyyyyyyyyyyyy",
    "fs.obs.endpoint" = "obs.cn-east-3.myhuaweicloud.com"
);
```

> **说明**
>
> 从华为云 OBS 导入数据时，需要先下载[依赖库](https://github.com/huaweicloud/obsa-hdfs/releases/download/v45/hadoop-huaweicloud-2.8.3-hw-45.jar)添加到 **$BROKER_HOME/lib/** 路径下并重启 Broker。

#### 查询数据

从 HDFS、Amazon S3、Google GCS、阿里云 OSS、腾讯云 COS、或者华为云 OBS 导入完成后，您可以使用 SELECT 语句来查看 StarRocks 表的数据，验证数据已经成功导入。

1. 查询 `table1` 表的数据，如下所示：

      ```SQL
      MySQL [test_db]> SELECT * FROM table1;
      +------+-------+-------+
      | id   | name  | score |
      +------+-------+-------+
      |    1 | Lily  |    23 |
      |    2 | Rose  |    23 |
      |    3 | Alice |    24 |
      |    4 | Julia |    25 |
      +------+-------+-------+
      4 rows in set (0.00 sec)
      ```

2. 查询 `table2` 表的数据，如下所示：

      ```SQL
      MySQL [test_db]> SELECT * FROM table2;
      +------+--------+
      | id   | city   |
      +------+--------+
      | 200  | 北京    |
      +------+--------+
      4 rows in set (0.01 sec)
      ```

#### 使用说明

以上导入操作以导入多个数据文件到多个目标表为例。您还可以指定导入一个数据文件或者一个路径下所有数据文件到一个目标表。这里假设您的 Amazon S3 存储空间 `bucket_s3` 里 `input` 文件夹下包含多个数据文件，其中一个数据文件名为 `file1.csv`。这些数据文件与目标表 `table1` 包含的列数相同、并且这些列能按顺序一一对应到目标表 `table1` 中的列。

如果要把数据文件 `file1.csv` 导入到目标表 `table1` 中，可以执行如下语句:

```SQL
LOAD LABEL test_db.label_7
(
    DATA INFILE("s3a://bucket_s3/input/file1.csv")
    INTO TABLE table1
    COLUMNS TERMINATED BY ","
    FORMAT AS "CSV"
)
WITH BROKER 
(
    "fs.s3a.access.key" = "xxxxxxxxxxxxxxxxxxxx",
    "fs.s3a.secret.key" = "yyyyyyyyyyyyyyyyyyyy",
    "fs.s3a.endpoint" = "s3.ap-northeast-1.amazonaws.com"
)；
```

如果要把 `input` 文件夹下的所有数据文件都导入到目标表 `table1` 中，可以执行如下语句:

```SQL
LOAD LABEL test_db.label_8
(
    DATA INFILE("s3a://bucket_s3/input/*")
    INTO TABLE table1
    COLUMNS TERMINATED BY ","
    FORMAT AS "CSV"
)
WITH BROKER 
(
    "fs.s3a.access.key" = "xxxxxxxxxxxxxxxxxxxx",
    "fs.s3a.secret.key" = "yyyyyyyyyyyyyyyyyyyy",
    "fs.s3a.endpoint" = "s3.ap-northeast-1.amazonaws.com"
)；
```

### 查看导入作业

Broker Load 支持通过 SHOW LOAD 语句和 curl 命令两种方式来查看导入作业的执行情况。

#### 使用 SHOW LOAD 语句

请参见 [SHOW LOAD](/sql-reference/sql-statements/data-manipulation/SHOW%20LOAD.md)。

#### 使用 curl 命令

命令语法如下：

```Bash
curl --location-trusted -u root: \
    'http://<fe_host>:<fe_http_port>/api/<database_name>/_load_info?label=<label_name>'
```

例如，可以通过如下命令查看 `db1` 数据库中标签为 `label1` 的导入作业的执行情况：

```Bash
curl --location-trusted -u root: \
    'http://<fe_host>:<fe_http_port>/api/db1/_load_info?label=label1'
```

命令执行后，以 JSON 格式返回导入作业的结果信息 `jobInfo`，如下所示：

```JSON
{"jobInfo":{"dbName":"default_cluster:db1","tblNames":["table1_simple"],"label":"label1","state":"FINISHED","failMsg":"","trackingUrl":""},"status":"OK","msg":"Success"}%
```

`jobInfo` 中包含如下参数：

| **参数**    | **说明**                                                     |
| ----------- | ------------------------------------------------------------ |
| dbName      | 目标 StarRocks 表所在的数据库的名称。                               |
| tblNames    | 目标 StarRocks 表的名称。                        |
| label       | 导入作业的标签。                                             |
| state       | 导入作业的状态，包括：<ul><li>`PENDING`：导入作业正在等待执行中。</li><li>`QUEUEING`：导入作业正在等待执行中。</li><li>`LOADING`：导入作业正在执行中。</li><li>`PREPARED`：事务已提交。</li><li>`FINISHED`：导入作业成功。</li><li>`CANCELLED`：导入作业失败。</li></ul>请参见[异步导入](/loading/Loading_intro.md#异步导入)。 |
| failMsg     | 导入作业的失败原因。当导入作业的状态为`PENDING`，`LOADING`或`FINISHED`时，该参数值为`NULL`。当导入作业的状态为`CANCELLED`时，该参数值包括 `type` 和 `msg` 两部分：<ul><li>`type` 包括如下取值：</li><ul><li>`USER_CANCEL`：导入作业被手动取消。</li><li>`ETL_SUBMIT_FAIL`：导入任务提交失败。</li><li>`ETL-QUALITY-UNSATISFIED`：数据质量不合格，即导入作业的错误数据率超过了 `max-filter-ratio`。</li><li>`LOAD-RUN-FAIL`：导入作业在 `LOADING` 状态失败。</li><li>`TIMEOUT`：导入作业未在允许的超时时间内完成。</li><li>`UNKNOWN`：未知的导入错误。</li></ul><li>`msg` 显示有关失败原因的详细信息。</li></ul> |
| trackingUrl | 导入作业中质量不合格数据的访问地址。可以使用 `curl` 命令或 `wget` 命令访问该地址。如果导入作业中不存在质量不合格的数据，则返回空值。 |
| status      | 导入请求的状态，包括 `OK` 和 `Fail`。                        |
| msg         | HTTP 请求的错误信息。                                        |

### 取消导入作业

当导入作业状态不为 **CANCELLED** 或 **FINISHED** 时，可以通过 [CANCEL LOAD](/sql-reference/sql-statements/data-manipulation/CANCEL%20LOAD.md) 语句来取消该导入作业。

例如，可以通过以下语句，撤销 `db1` 数据库中标签为 `label1` 的导入作业：

```SQL
CANCEL LOAD
FROM db1
WHERE LABEL = "label";
```

## 作业拆分与并行执行

一个 Broker Load 作业会拆分成一个或者多个子任务并行处理，一个作业的所有子任务作为一个事务整体成功或失败。作业的拆分通过 `LOAD LABEL` 语句中的 `data_desc` 参数来指定：

- 如果声明多个 `data_desc` 参数对应导入多张不同的表，则每张表数据的导入会拆分成一个子任务。

- 如果声明多个 `data_desc` 参数对应导入同一张表的不同分区，则每个分区数据的导入会拆分成一个子任务。

每个子任务还会拆分成一个或者多个实例，然后这些实例会均匀地被分配到 BE 上并行执行。实例的拆分由以下 [FE 配置](/administration/Configuration.md#配置-fe-动态参数)决定：

- `min_bytes_per_broker_scanner`：单个实例处理的最小数据量，默认为 64 MB。

- `max_broker_concurrency`：单个子任务允许的最大并发实例数，默认为 100 个。

- `load_parallel_instance_num`：单个 BE 上每个作业允许的并发实例数，默认为 1 个。

    可以使用如下公式计算单个子任务的实例总数：

    单个子任务的实例总数 = min（单个子任务待导入数据量的总大小/`min_bytes_per_broker_scanner`，`max_broker_concurrency`，`load_parallel_instance_num` x BE 总数）

一般情况下，一个导入作业只有一个 `data_desc`，只会拆分成一个子任务，子任务会拆分成与 BE 总数相等的实例。

## 相关配置项

[FE 配置项](../administration/Configuration.md#fe-配置项) `async_load_task_pool_size` 指定了 Broker Load 任务线程池的大小，即 StarRocks 集群中可以并行执行的 Broker Load 任务的最大数量。

StarRocks v2.4 及以前版本中，如果某一时间段内提交的 Broker Load 作业的任务总数超过最大数量，则超出作业会按照各自的提交时间放到队列中排队等待调度。

StarRocks v2.5 版本中，如果某一时间段内提交的 Broker Load 作业的任务总数超过最大数量，则超出的作业会按照作业创建时指定的优先级被放到队列中排队等待调度。参见 [BROKER LOAD](../sql-reference/sql-statements/data-manipulation/BROKER%20LOAD.md#opt_properties) 文档中的可选参数 `priority`。您可以使用 [ALTER LOAD](../sql-reference/sql-statements/data-manipulation/ALTER%20LOAD.md) 语句修改处于 **QUEUEING** 状态或者 **LOADING** 状态的 Broker Load 作业的优先级。

## 常见问题

请参见 [Broker Load 常见问题](/faq/loading/Broker_load_faq.md)。
