# CREATE ROUTINE LOAD

## 功能

Routine Load 是一种基于 MySQL 协议的异步导入方式，支持持续消费 Apache Kafka® 的消息并导入至 StarRocks 中。

Routine Load 支持消费 Kafka 中 CSV 或 JSON 格式数据。Routine Load 支持通过无安全认证、SSL 加密和认证、或者 SASL 认证机制访问 Kafka。

本文介绍 CREATE ROUTINE LOAD 的语法、参数说明和示例。

> **说明**
>
> Routine Load 的应用场景、基本原理和基本操作，请参见 [从 Apache Kafka® 持续导入](../../../loading/RoutineLoad.md)。

## 语法

```SQL
CREATE ROUTINE LOAD <database_name>.<job_name> ON <table_name>
[load_properties]
[job_properties]
FROM data_source
[data_source_properties];
```

## 参数说明

### `database_name`、`job_name`、`table_name`

`database_name`

选填，目标数据库的名称。

`job_name`

必填，导入作业的名称。一张表可能有多个导入作业，建议您利用具有辨识度的信息（例如 Kafka Topic 名称、创建导入作业的大致时间等）来设置具有意义的导入作业名称，用于区分多个导入作业。同一数据库内，导入作业的名称必须唯一。

`table_name`

必填，目标表的名称。

### `load_properties`

选填。待导入数据的属性。语法：

```SQL
[COLUMNS TERMINATED BY '<terminator>'],
[COLUMNS ([<column_name>[,...]][,column_assignment[,...]])],
[WHERE <expr>],
[PARTITION ([<partition_name>[,...]])]
COLUMNS TERMINATED BY
```

`COLUMNS TERMINATED BY`
如果导入 CSV 格式的数据，则可以指定列分隔符，默认为`\t`，即 Tab。例如可以输入 `COLUMNS TERMINATED BY ","`。指定列分隔符为逗号(,)。

> **说明**
>
> - 必须确保这里指定的列分隔符与待导入数据文件中的列分隔符一致。
> - StarRocks 支持设置长度最大不超过 50 个字节的 UTF-8 编码字符串作为列分隔符，包括常见的逗号 (,)、Tab 和 Pipe (|)。

`ROWS TERMINATED BY`

用于指定待导入数据文件中的行分隔符。如果不指定该参数，则默认为 `\n`。

`COLUMNS`

待导入数据和目标表之间的列映射和转换关系。详细说明，请参见[列映射和转换关系](#列映射和转换关系)。

- `column_name`：映射列，待导入数据中这类列的值可以直接落入目标表的列中，不需要进行计算。
- `column_assignment`：衍生列，格式为 `column_name = expr`，源数据中这类列的值需要基于表达式 `expr` 进行计算后，才能落入目标表的列中。 建议将衍生列排在映射列之后，因为 StarRocks 先解析映射列，再解析衍生列。

> **说明**
>
> 以下情况不需要设置 `COLUMNS` 参数：
>
> - 待导入 CSV 数据中的列与目标表中列的数量和顺序一致。
> - 待导入 JSON 数据中的 Key 名与目标表中的列名一致。

`WHERE`

设置过滤条件，只有满足过滤条件的数据才会导入到 StarRocks 中。例如只希望导入 `col1` 大于 `100` 并且 `col2` 等于 `1000` 的数据行，则可以输入 `WHERE col1 > 100 and col2 = 1000;`。

> **说明**
>
> 过滤条件中指定的列可以是待导入数据中本来就存在的列，也可以是基于待导入数据的列生成的衍生列。

`PARTITION`

将数据导入至目标表的指定分区中。如果不指定分区，则会将数据自动导入至其对应的分区中。 示例：

```SQL
PARTITION(p1, p2, p3)
```

### `job_properties`

必填。导入作业的的属性。语法：

```SQL
PROPERTIES ("<key1>" = "<value1>"[, "<key2>" = "<value2>" ...])
```

参数说明如下：

| **参数**                  | **是否必选** | **说明**                                                     |
| ------------------------- | ------------ | ------------------------------------------------------------ |
| desired_concurrent_number | 否           | 单个 Routine Load 导入作业的**期望**任务并发度，表示期望一个导入作业最多被分成多少个任务并行执行。默认值为 `3`。 但是**实际**任务并行度由如下多个参数组成的公式决定，并且实际任务并行度的上限为 BE 节点的数量或者消费分区的数量。`min(alive_be_number, partition_number, desired_concurrent_number, max_routine_load_task_concurrent_num)``alive_be_number`：存活的 BE 节点数量。`partition_number`：消费分区数量。`desired_concurrent_number`：单个Routine Load 导入作业的期望任务并发度。默认值为 `3`。`max_routine_load_task_concurrent_num`：Routine Load 导入作业的默认最大任务并行度，默认值为 `5`。该参数为 [FE 动态参数](../../../administration/Configuration.md)。 |
| max_batch_interval        | 否           | 任务的调度间隔，即任务多久执行一次。单位：秒。取值范围：`5`～`60`。默认值：`10`。建议取值为导入间隔 10s 以上，否则会因为导入频率过高可能会报错版本数过多。 |
| max_batch_rows            | 否           | 该参数只用于定义错误检测窗口范围，错误检测窗口范围为单个 Routine Load 导入任务所消费的 `10 * max-batch-rows` 行数据，默认为 `10 * 200000 = 2000000`。导入任务时会检测窗口中数据是否存在错误。错误数据是指 StarRocks 无法解析的数据，比如非法的 JSON。 |
| max_error_number          | 否           | 错误检测窗口范围内允许的错误数据行数的上限。当错误数据行数超过该值时，导入作业会暂停，此时您需要执行 [SHOW ROUTINE LOAD](../data-manipulation/SHOW%20ROUTINE%20LOAD.md)，根据 `ErrorLogUrls`，检查 Kafka 中的消息并且更正错误。默认为 `0`，表示不允许有错误行。错误行不包括通过 WHERE 子句过滤掉的数据。 |
| strict_mode               | 否           | 是否开启严格模式。取值范围：`TRUE` 或者 `FALSE`。默认值：`FALSE`。开启后，如果待导入数据某列的值为 `NULL`，但是目标表中该列不允许为 `NULL`，则该行数据会被过滤掉。 |
| timezone                  | 否           | 该参数的取值会影响所有导入涉及的、跟时区设置有关的函数所返回的结果。受时区影响的函数有 strftime、alignment_timestamp 和 from_unixtime 等，具体请参见[设置时区](../../../administration/timezone.md)。导入参数 `timezone` 设置的时区对应[设置时区](../../../administration/timezone.md)中所述的会话级时区。 |
| merge_condition           | 否           | 用于指定作为更新生效条件的列名。这样只有当导入的数据中该列的值大于当前值的时候，更新才会生效。参见[通过导入实现数据变更](../../../loading/PrimaryKeyLoad.md)。指定的列必须为非主键列，且仅主键模型表支持条件更新。 |
| format                    | 否           | 待导入数据的格式，取值范围：`CSV` 或者 `JSON`。默认值：`CSV`。 |
| strip_outer_array         | 否           | 是否裁剪 JSON 数据最外层的数组结构。取值范围：`TRUE` 或者 `FALSE`。默认值：`FALSE`。真实业务场景中，待导入的 JSON 数据可能在最外层有一对表示数组结构的中括号 `[]`。这种情况下，一般建议您指定该参数取值为 `true`，这样 StarRocks 会剪裁掉外层的中括号 `[]`，并把中括号 `[]` 里的每个内层数组都作为一行单独的数据导入。如果您指定该参数取值为 `false`，则 StarRocks 会把整个 JSON 数据文件解析成一个数组，并作为一行数据导入。例如，待导入的 JSON 数据为 `[ {"category" : 1, "author" : 2}, {"category" : 3, "author" : 4} ]`，如果指定该参数取值为 `true`，则 StarRocks 会把 `{"category" : 1, "author" : 2}` 和 `{"category" : 3, "author" : 4}` 解析成两行数据，并导入到目标表中对应的数据行。 |
| jsonpaths                 | 否           | 导入时指定待导入 JSON 数据的 Key，该模式称为匹配模式，适用于 JSON 数据的 Key 名与目标表的列名不一致的场景。具体请参见本文提供的示例[目标表的列名与 JSON 数据的 key 一致]。如果 JSON 数据的 Key 名与目标表的列名一致，则可以不设置 `jsonpaths` ，该模式称为简单模式。例如 `{"category": 1, "author": 2, "price": "3"}` 中，`category`、`author`、`price` 是字段的名称，按名称直接对应目标表中的 `category`、`author`、`price` 三列。具体请参见本文提供的示例[目标表存在基于 JSON 数据进行计算生成的衍生列] |
| json_root                 | 否           | 如果不需要导入整个 JSON 数据，则指定实际待导入 JSON 数据的根节点。参数取值为合法的 JsonPath。默认值为空，表示会导入整个 JSON 数据。具体请参见本文提供的示例[指定实际待导入 JSON 数据的根节点]。 |

### `data_source`、`data_source_properties`

数据源和数据源属性。语法:

```Bash
FROM <data_source>
 ("<key1>" = "<value1>"[, "<key2>" = "<value2>" ...])
data_source
```

`data_source`

必填。指定数据源，目前仅支持取值为 `KAFKA`。

`data_source_properties`

必填。数据源属性，参数以及说明如下：

| **参数**          | **说明**                                                     |
| ----------------- | ------------------------------------------------------------ |
| kafka_broker_list | Kafka 的 Broker 连接信息。格式为 `<kafka_broker_ip>:<kafka port>`，多个 Broker 之间以英文逗号 (,) 分隔。 Kafka Broker 默认端口号为 `9092`。示例：`"kafka_broker_list" = "broker1:9092,broker2:9092"` |
| kafka_topic       | Kafka Topic 名称。一个导入作业仅支持消费一个 Topic 的消息。  |
| kafka_partitions  | 待消费的分区。如果不配置该参数，则默认消费所有分区。`"kafka_partitions" = "0, 1, 2, 3"` |
| kafka_offsets     | 待消费分区的起始消费位点，必须完全对应 `kafka_partitions` 中指定分区。如果不配置该参数，则默认为从分区的末尾开始消费。取值为具体消费位点，或者：`OFFSET_BEGINNING`：从分区中有数据的位置开始消费。`OFFSET_END`：从分区的末尾开始消费。`"kafka_offsets" = "1000, OFFSET_BEGINNING, OFFSET_END, 2000"` |

**更多数据源相关参数**

支持设置更多数据源 Kafka 相关参数，功能等同于 Kafka 命令行 `--property`， 支持参数，请参见 [librdkafka 配置项文档](https://github.com/edenhill/librdkafka/blob/master/CONFIGURATION.md)中适用于客户端的配置项。

> **说明**
>
> 当参数的取值是文件时，则值前加上关键词 `FILE:`。关于如何创建文件，请参见 [CREATE FILE](../Administration/CREATE%20FILE.md) 命令文档。

**指定所有待消费分区的默认起始消费位点。**

```Plaintext
"property.kafka_default_offsets" = "OFFSET_BEGINNING"
```

`property.kafka_default_offsets` 的取值为具体的消费位点，或者：

- `OFFSET_BEGINNING`：从分区中有数据的位置开始消费。
- `OFFSET_END`：从分区的末尾开始消费。

**指定导入任务消费 Kafka 时所基于 Consumer Group 的 `group.id`**

```Plaintext
"property.group.id" = "group_id_0"
```

如果没有指定 `group.id`，StarRocks 会根据 Routine Load 的导入作业名称生成一个随机值，具体格式为`{job_name}_{random uuid}`，如 `simple_job_0a64fe25-3983-44b2-a4d8-f52d3af4c3e8`。

**指定 BE 访问 Kafka 时使用的认证机制**

- **使用SSL 加密和 CA 证书认证访问 Kafka**

```Plaintext
"property.security.protocol" = "ssl", --使用 SSL 加密
"property.ssl.ca.location" = "FILE:ca-cert",--CA 证书的位置
--如果 Kafka server 端开启了 client 认证，则还需设置如下三个参数：
"property.ssl.certificate.location" = "FILE:client.pem", -- client 的 public key 的位置
"property.ssl.key.location" = "FILE:client.key", -- client 的 private key 的位置
"property.ssl.key.password" = "abcdefg",-- client 的 private key 的密码
```

- **使用 SASL 认证机制访问 Kafka**

```Plaintext
"property.security.protocol"="SASL_PLAINTEXT",--使用 SASL 认证机制
"property.sasl.mechanism"="PLAIN",-- SASL 的 认证方式为 PLAIN
"property.sasl.username"="admin",--SASL 的用户名
"property.sasl.password"="admin"--SASL 的密码
```

### FE 和 BE 配置项

Routine Load 相关配置项，请参见[配置参数](../../../administration/Configuration.md)。

## 列映射和转换关系

### 导入 CSV 数据

**如果 CSV 格式的数据中的列与目标表中的列的数量或顺序不一致**，则需要通过 `COLUMNS` 参数来指定待导入数据文件和目标表之间的列映射和转换关系。一般包括如下两种场景：

- **待导入数据文件中的列与目标表中的列数量一致，但是顺序不一致**。并且数据不需要通过函数计算、可以直接落入目标表中对应的列。
  您需要在 `COLUMNS` 参数中按照待导入数据中的列顺序、使用目标表中对应的列名来配置列映射和转换关系。

  例如，目标表中有三列，按顺序依次为 `col1`、`col2` 和 `col3`；待导入数据中也有三列，按顺序依次对应目标表中的 `col3`、`col2` 和 `col1`。这种情况下，需要指定 `COLUMNS(col3, col2, col1)`。
- **待导入数据文件中的列与目标表中的列数量不一致，甚至某些列的数据需要通过转换（函数计算以后）才能落入目标表中对应的列。**
  您不仅需要在 `COLUMNS` 参数中按照待导入数据文件中的列顺序、使用目标表中对应的列名来配置列映射关系，还需要指定参与数据计算的函数。以下为两个示例：

  - **待导入数据比目标表多列**。
    比如目标表中有三列，按顺序依次为 `col1`、`col2` 和 `col3` ；待导入数据文件中有四列，前三列按顺序依次对应目标表中的 `col1`、`col2` 和 `col3`，第四列在目标表中无对应的列。这种情况下，需要指定 `COLUMNS(col1, col2, col3, temp)`，其中，最后一列可随意指定一个名称（如 `temp`）用于占位即可。
  - **目标表存在基于待导入数据的列进行计算后生成的衍生列**。
    例如待导入数据中只有一个包含时间数据的列，格式为 `yyyy-mm-dd hh:mm:ss`。目标表中有三列，按顺序依次为 `year`、`month` 和 `day`，均是基于待导入数据中包含时间数据的列进行计算后生成的衍生列。这种情况下，可以指定 `COLUMNS(col, year = year(col), month=month(col), day=day(col)`。其中，`col` 是待导入数据文件中所包含的列的临时命名，`year = year(col)`、`month=month(col)` 和 `day=day(col)` 用于指定从待导入数据文件中的 `col` 列提取对应的数据并落入目标表中对应的衍生列，如 `year = year(col)` 表示通过 `year` 函数提取待导入数据中 `col` 列的 `yyyy` 部分的数据并落入目标表中的 `year` 列。更多示例，请参见[设置列的映射和转换关系]xxx

### 导入 JSON 数据

**如果 JSON 格式的数据中的 Key 名与目标表中的列名不一致**，则需要使用匹配模式导入 JSON 数据，即通过 `jsonpaths` 和 `COLUMNS` 两个参数来指定待导入数据和目标表之间的列映射和转换关系：

- `jsonpaths` 参数指定待导入 JSON 数据的 Key，并进行排序（就像新生成了 CSV 数据）。
- `COLUMNS`参数指定待导入 JSON 数据的 Key 与目标表的列的映射关系和数据转换关系。
  - 与 `jsonpaths`中指定的 Key 按顺序保持一一对应。
  - 与目标表中的列按名称保持一一对应。

详细示例，请参见[目标表存在基于 JSON 数据进行计算生成的衍生列]

> **说明**
>
> 如果待导入 JSON 数据中 Key 名（Key的顺序和数量不需要对应）都能对应目标表中列名，则可以使用简单模式导入 JSON 数据，无需配置 `jsonpaths` 和 `COLUMNS`。

## 示例

### 导入 CSV 数据

本小节以 CSV 格式的数据为例，重点阐述在创建导入作业的时候，如何运用各种参数配置来满足不同业务场景下的各种导入要求。

**数据集**

假设 Kafka 集群的 Topic `ordertest1` 存在如下 CSV 格式的数据，其中 CSV 数据中列的含义依次是订单编号、支付日期、顾客姓名、国籍、性别、支付金额。

```Plain
2020050802,2020-05-08,Johann Georg Faust,Deutschland,male,895
2020050802,2020-05-08,Julien Sorel,France,male,893
2020050803,2020-05-08,Dorian Grey,UK,male,1262
2020050901,2020-05-09,Anna Karenina,Russia,female,175
2020051001,2020-05-10,Tess Durbeyfield,US,female,986
2020051101,2020-05-11,Edogawa Conan,japan,male,8924
```

**目标数据库和表**

根据 CSV 数据中需要导入的列， 在 StarRocks 集群的目标数据库 `example_db` 中创建表 `example_tbl1`，建表语句如下：

```SQL
CREATE TABLE example_db.example_tbl1 ( 
    `order_id` bigint NOT NULL COMMENT "订单编号",
    `pay_dt` date NOT NULL COMMENT "支付日期", 
    `customer_name` varchar(26) NULL COMMENT "顾客姓名", 
    `nationality` varchar(26) NULL COMMENT "国籍", 
    `gender` varchar(26) NULL COMMENT "性别", 
    `price`double NULL COMMENT "支付金额") 
ENGINE=OLAP 
PRIMARY KEY (order_id,pay_dt) 
DISTRIBUTED BY HASH(`order_id`) BUCKETS 5; 
```

#### 从 Topic 指定分区和起始位点开始消费

如果需要指定待消费分区，例如为 0、1、2、3、4，消费分区对应的起始位点分别为 1000、从分区中有数据的位置开始、分区末尾、Offset 为 2000， 则需要配置参数 `kafka_partitions`、`kafka_offsets`。

```SQL
CREATE ROUTINE LOAD example_db.example_tbl1_ordertest1 ON example_tbl1
COLUMNS TERMINATED BY ",",
COLUMNS (order_id, pay_dt, customer_name, nationality, gender, price)
FROM KAFKA
(
    "kafka_broker_list" ="<kafka_broker1_ip>:<kafka_broker1_port>,<kafka_broker2_ip>:<kafka_broker2_port>",
    "kafka_topic" = "ordertest1",
    "kafka_partitions" ="0,1,2,3,4",--指定分区
    "kafka_offsets" = "1000, OFFSET_BEGINNING, OFFSET_END, 2000"--指定起始位点
);
```

#### 调整导入性能

如果需要提高导入性能，避免出现消费积压等情况，则可以通过设置单个 Routine Load 导入作业的期望任务并发度`desired_concurrent_number`，增加实际任务并行度，将一个导入作业拆分成尽可能多的导入任务并行执行。

> 更多提升导入性能的方式，请参见 [Routine Load常见问题]

请注意，实际任务并行度由如下多个参数组成的公式决定，上限为 BE 节点的数量或者消费分区的数量。

```SQL
min(alive_be_number, partition_number, desired_concurrent_number, max_routine_load_task_concurrent_num)
```

因此当消费分区和 BE 节点数量较多，并且大于其余两个参数时，如果您需要增加实际任务并行度，则可以提高`max_routine_load_task_concurrent_num`、 `desired_concurrent_number` 的值。

假设消费分区数量为 `7`，存活 BE 数量为 `5`，`max_routine_load_task_concurrent_num` 为默认值 `5`。此时如果需要增加实际任务并发度至上限，则需要将 `desired_concurrent_number` 设置为 `5`（默认值为 `3`），则计算实际任务并行度 `min(5,7,5,5)` 为 `5`。

```SQL
CREATE ROUTINE LOAD example_db.example_tbl1_ordertest1 ON example_tbl1
COLUMNS TERMINATED BY ",",
COLUMNS (order_id, pay_dt, customer_name, nationality, gender, price)
PROPERTIES
(
"desired_concurrent_number" = "5" --设置单个 Routine Load 导入作业的期望任务并发度
)
FROM KAFKA
(
    "kafka_broker_list" ="<kafka_broker1_ip>:<kafka_broker1_port>,<kafka_broker2_ip>:<kafka_broker2_port>",
    "kafka_topic" = "ordertest1"
);
```

#### 设置列的映射和转换关系

如果 CSV 格式的数据中的列与目标表中的列的数量或顺序不一致，假设无需导入 CSV 数据的第五列至目标表，则需要通过 `COLUMNS` 参数来指定待导入数据文件和目标表之间的列映射和转换关系。

**目标数据库和表**

根据 CSV 数据中需要导入的几列（例如除第五列性别外的其余五列需要导入至 StarRocks）， 在 StarRocks 集群的目标数据库 `example_db` 中创建表 `example_tbl1`。

```SQL
CREATE TABLE example_db.example_tbl2 ( 
    `order_id` bigint NOT NULL COMMENT "订单编号",
    `pay_dt` date NOT NULL COMMENT "支付日期", 
    `customer_name` varchar(26) NULL COMMENT "顾客姓名", 
    `nationality` varchar(26) NULL COMMENT "国籍", 
    `price`double NULL COMMENT "支付金额"
) 
ENGINE=OLAP 
PRIMARY KEY (order_id,pay_dt) 
DISTRIBUTED BY HASH(`order_id`) BUCKETS 5; 
```

**导入作业**

本示例中，由于无需导入 CSV 数据的第五列至目标表，因此`COLUMNS`中把第五列临时命名为 `temp_gender` 用于占位，其他列都直接映射至表 `example_tbl2` 中。

```SQL
CREATE ROUTINE LOAD example_db.example_tbl2_ordertest1 ON example_tbl2
COLUMNS TERMINATED BY ",",
COLUMNS (order_id, pay_dt, customer_name, nationality, temp_gender, price)
FROM KAFKA
(
    "kafka_broker_list" ="<kafka_broker1_ip>:<kafka_broker1_port>,<kafka_broker2_ip>:<kafka_broker2_port>",
    "kafka_topic" = "ordertest1"
);
```

#### 设置过滤条件 筛选待导入的数据

如果仅导入满足条件的数据，则可以在 WHERE 子句中设置过滤条件，例如`price > 100`。

```Plain
CREATE ROUTINE LOAD example_db.example_tbl2_ordertest1 ON example_tbl2
COLUMNS TERMINATED BY ",",
COLUMNS (order_id, pay_dt, customer_name, nationality, gender, price)
WHERE price > 100
FROM KAFKA
(
    "kafka_broker_list" ="<kafka_broker1_ip>:<kafka_broker1_port>,<kafka_broker2_ip>:<kafka_broker2_port>",
    "kafka_topic" = "ordertest1"
);
```

#### 设置导入任务为严格模式

在`PROPERTIES`中设置`"strict_mode" = "true"`，表示导入作业为严格模式。如果待导入数据某列的值为 NULL，但是目标表中该列不允许为 NULL，则该行数据会被过滤掉。

```SQL
CREATE ROUTINE LOAD example_db.example_tbl1_ordertest1 ON example_tbl1
COLUMNS TERMINATED BY ",",
COLUMNS (order_id, pay_dt, customer_name, nationality, gender, price)
PROPERTIES
(
"strict_mode" = "true" --设置导入作业为严格模式
)
FROM KAFKA
(
    "kafka_broker_list" ="<kafka_broker1_ip>:<kafka_broker1_port>,<kafka_broker2_ip>:<kafka_broker2_port>",
    "kafka_topic" = "ordertest1"
);
```

#### 设置导入任务的容错率

如果业务场景对数据质量的有要求，则需要设置参数`max_batch_rows`和`max_error_number`设置错误检测窗口的范围和允许的错误数据行数的上限，当错误数据行数超过该值时，导入作业会暂停。

```SQL
CREATE ROUTINE LOAD example_db.example_tbl1_ordertest1 ON example_tbl1
COLUMNS TERMINATED BY ",",
COLUMNS (order_id, pay_dt, customer_name, nationality, gender, price)
PROPERTIES
(
"max_batch_rows" = "100000",--错误检测窗口范围为单个 Routine Load 导入任务所消费的 10 * max-batch-rows 行数。
"max_error_number" = "100" --错误检测窗口范围内允许的错误数据行数的上限
)
FROM KAFKA
(
    "kafka_broker_list" ="<kafka_broker1_ip>:<kafka_broker1_port>,<kafka_broker2_ip>:<kafka_broker2_port>",
    "kafka_topic" = "ordertest1"
);
```

#### 设置增加 SSL 认证

如果需要指定 BE 访问 Kafka 时使用的认证机制，例如进行 SSL 加密和 CA 证书认证，则需要配置自定义参数`property.security.protocol`、`property.ssl.ca.location`，指定使用 SSL 加密以及 CA 证书的位置。

```SQL
CREATE ROUTINE LOAD example_db.example_tbl1_ordertest1 ON example_tbl1
COLUMNS TERMINATED BY ",",
COLUMNS (order_id, pay_dt, customer_name, nationality, gender, price)
PROPERTIES
(
"property.security.protocol" = "ssl", --使用 SSL 加密
"property.ssl.ca.location" = "FILE:ca-cert",--CA 证书的位置
--如果 Kafka Server 端开启了 Client 身份认证，则还需设置如下三个参数：
"property.ssl.certificate.location" = "FILE:client.pem", -- Client 的 Public Key 的位置
"property.ssl.key.location" = "FILE:client.key", -- Client 的 Private Key 的位置
"property.ssl.key.password" = "abcdefg",-- Client 的 Private Key 的密码
)
FROM KAFKA
(
    "kafka_broker_list" ="<kafka_broker1_ip>:<kafka_broker1_port>,<kafka_broker2_ip>:<kafka_broker2_port>",
    "kafka_topic" = "ordertest1"
);
```

### 导入 JSON 格式数据

#### **目标表的列名与** **JSON** **数据的** **Key 一致**

可以使用简单模式导入数据，即创建导入作业时无需使用 `jsonpaths` 和 `COLUMNS` 参数。StarRocks 会按照目标表的列名去对应 JSON 数据的 Key。

**数据集**

假设 Kafka 集群的 Topic `ordertest2` 中存在如下 JSON 数据。

```JSON
{"commodity_id": "1", "customer_name": "Mark Twain", "country": "US","pay_time": 1589191487,"price": 875}
{"commodity_id": "2", "customer_name": "Oscar Wilde", "country": "UK","pay_time": 1589191487,"price": 895}
{"commodity_id": "3", "customer_name": "Antoine de Saint-Exupéry","country": "France","pay_time": 1589191487,"price": 895}
```

> 注意
>
> 这里每行一个 JSON 对象必须在一个 Kafka 消息中，否则会出现“JSON 解析错误”的问题。

**目标数据库和表**

在 StarRocks 集群的目标数据库 `example_db` 中创建表 `example_tbl3` ，并且列名与 JSON 数据中需要导入的 Key 一致。

```SQL
CREATE TABLE example_db.example_tbl3 ( 
    `commodity_id` varchar(26) NULL COMMENT "品类ID", 
    `customer_name` varchar(26) NULL COMMENT "顾客姓名", 
    `country` varchar(26) NULL COMMENT "顾客国籍", 
    `pay_time` bigint(20) NULL COMMENT "支付时间", 
    `price`double SUM NULL COMMENT "支付金额") 
AGGREGATE KEY(`commodity_id`,`customer_name`,`country`,`pay_time`) 
ENGINE=OLAP
DISTRIBUTED BY HASH(`commodity_id`) BUCKETS 5; 
```

**导入作业**

提交导入作业时使用简单模式，即无需使用`jsonpaths` 和 `COLUMNS` 参数，就可以将 Kafka 集群的 Topic `ordertest2` 中的 JSON 数据导入至目标表 `example_tbl3` 中。

```SQL
CREATE ROUTINE LOAD example_db.example_tbl3_ordertest2 ON example_tbl3
PROPERTIES
(
    "format" ="json"
 )
FROM KAFKA
(
    "kafka_broker_list" ="<kafka_broker1_ip>:<kafka_broker1_port>,<kafka_broker2_ip>:<kafka_broker2_port>",
    "kafka_topic" = "ordertest2"
);
```

> **说明**
>
> - 如果 JSON 数据最外层是数组结构，则需要在`PROPERTIES`设置`"strip_outer_array"="true"`，表示裁剪最外层的数组结构。并且需要注意在设置 `jsonpaths` 时，整个 JSON 数据的根节点是裁剪最外层的数组结构后**展平的 JSON 对象**。
> - 如果不需要导入整个 JSON 数据，则需要使用 `json_root` 指定实际所需导入的 JSON 数据根节点。

#### 目标表**存在基于** **JSON** **数据进行计算生成的衍生列**

需要使用匹配模式导入数据，即需要使用 `jsonpaths` 和 `COLUMNS` 参数，`jsonpaths`指定待导入 JSON 数据的 Key，`COLUMNS` 参数指定待导入 JSON 数据的 Key 与目标表的列的映射关系和数据转换关系。

**数据集**

假设 Kafka 集群的 Topic `ordertest2` 中存在如下 JSON 格式的数据。

```JSON
{"commodity_id": "1", "customer_name": "Mark Twain", "country": "US","pay_time": 1589191487,"price": 875}
{"commodity_id": "2", "customer_name": "Oscar Wilde", "country": "UK","pay_time": 1589191487,"price": 895}
{"commodity_id": "3", "customer_name": "Antoine de Saint-Exupéry","country": "France","pay_time": 1589191487,"price": 895}
```

**目标数据库和表**

假设在 StarRocks 集群的目标数据库 `example_db` 中存在目标表 `example_tbl3` 其中有一列衍生列 `pay_dt`，是基于 JSON 数据的Key `pay_time` 进行计算后的数据。其建表语句如下：

```SQL
CREATE TABLE example_db.example_tbl3 ( 
    `commodity_id` varchar(26) NULL COMMENT "品类ID", 
    `customer_name` varchar(26) NULL COMMENT "顾客姓名", 
    `country` varchar(26) NULL COMMENT "顾客国籍", 
    `pay_dt` date NULL COMMENT "支付日期", 
    `price`double SUM NULL COMMENT "支付金额") 
AGGREGATE KEY(`commodity_id`,`customer_name`,`country`,`pay_time`,`pay_dt`) 
ENGINE=OLAP
DISTRIBUTED BY HASH(`commodity_id`) BUCKETS 5; 
```

**导入作业**

提交导入作业时使用匹配模式。使用 `jsonpaths` 指定待导入 JSON 数据的 Key。并且由于 JSON 数据中 key `pay_time` 需要转换为 DATE 类型，才能导入到目标表的列 `pay_dt`，因此 `COLUMNS` 中需要使用函数`from_unixtime`进行转换。JSON 数据的其他 key 都能直接映射至表 `example_tbl3` 中。

```SQL
CREATE ROUTINE LOAD example_db.example_tbl3_ordertest2 ON example_tbl3
COLUMNS(commodity_id, customer_name, country, pay_time, pay_dt=from_unixtime(pay_time, '%Y%m%d'), price)
PROPERTIES
(
    "format" ="json",
    "jsonpaths" ="[\"$.commodity_id\",\"$.customer_name\",\"$.country\",\"$.pay_time\",\"$.price\"]"
 )
FROM KAFKA
(
    "kafka_broker_list" ="<kafka_broker1_ip>:<kafka_broker1_port>,<kafka_broker2_ip>:<kafka_broker2_port>",
    "kafka_topic" = "ordertest2"
);
```

> **说明**
>
> - 如果 JSON 数据最外层是数组结构，则需要在`PROPERTIES`设置`"strip_outer_array"="true"`，表示裁剪最外层的数组结构。并且需要注意在设置 `jsonpaths` 时，整个 JSON 数据的根节点是裁剪最外层的数组结构后**展平的 JSON 对象**。
> - 如果不需要导入整个 JSON 数据，则需要使用 `json_root` 指定实际所需导入的 JSON 数据根节点。

#### 指定实际待导入 JSON 数据的根节点

如果不需要导入整个 JSON 数据，则需要使用 `json_root` 指定实际上所需导入的 JSON 数据的根对象，参数取值为合法的 JsonPath。

**数据集**

假设 Kafka 集群的 Topic `ordertest2` 中存在如下 JSON 格式的数据，实际导入时仅需要导入 key `RECORDS`的值。

```JSON
{
"RECORDS":[
{"commodity_id": "1", "customer_name": "Mark Twain", "country": "US","pay_time": 1589191487,"price": 875}
{"commodity_id": "2", "customer_name": "Oscar Wilde", "country": "UK","pay_time": 1589191487,"price": 895}
{"commodity_id": "3", "customer_name": "Antoine de Saint-Exupéry","country": "France","pay_time": 1589191487,"price": 895}
]
}
```

**目标数据库和表**

假设在 StarRocks 集群的目标数据库 `example_db` 中存在目标表`example_tbl3` ，其建表语句如下：

```SQL
CREATE TABLE example_db.example_tbl3 ( 
    `commodity_id` varchar(26) NULL COMMENT "品类ID", 
    `customer_name` varchar(26) NULL COMMENT "顾客姓名", 
    `country` varchar(26) NULL COMMENT "顾客国籍", 
    `pay_time` bigint(20) NULL COMMENT "支付时间", 
    `price`double SUM NULL COMMENT "支付金额") 
AGGREGATE KEY(`commodity_id`,`customer_name`,`country`,`pay_time`) 
ENGINE=OLAP
DISTRIBUTED BY HASH(`commodity_id`) BUCKETS 5; 
```

**导入作业**

提交导入作业，设置`"json_root" = "$.RECORDS"`指定实际待导入的json 数据的根节点。并且由于实际待导入的 JSON 数据是数组结构，因此还需要设置`"strip_outer_array" = "true"`，裁剪外层的数组结构。

```SQL
CREATE ROUTINE LOAD example_db.example_tbl3_ordertest2 ON example_tbl3
PROPERTIES
(
    "format" ="json"
    "strip_outer_array" = "true",
    "json_root" = "$.RECORDS"
 )
FROM KAFKA
(
    "kafka_broker_list" ="<kafka_broker1_ip>:<kafka_broker1_port>,<kafka_broker2_ip>:<kafka_broker2_port>",
    "kafka_topic" = "ordertest2"
);
```
