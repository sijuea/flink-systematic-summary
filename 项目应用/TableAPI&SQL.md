


###  1.概念

  **[1]. Table API**：以 Java/Scala 编程语言编写的 DSL（领域特定语言），具备 SQL 类似的表达能力，支持链式操作。
  
  **[2]. SQL API**：用户可通过标准 SQL 查询语言对动态流或静态表执行操作，类似传统数据库系统。
  
  **[3]. TableEnvironment**：统一的入口，用于注册表、执行 SQL、控制上下文行为（如流/批模式）。
  
  **[4]. 动态表（Dynamic Table）模型**：将流数据建模为持续更新的表，实现流式数据的表语义。
  
  **[5]. 统一语义**：Flink 的 Table & SQL 模块实现了批流一体，在语义、API 和优化层面统一处理逻辑。

  **[6]. 自定义函数：**
  
   * **UDF（User-Defined Function）**：标量函数，实现一进一出，扩展内置函数库。
    
   * **UDTF（User-Defined Table Function）**：表函数，一进多出，用于生成多行数据。
    
   * **UDAF（User-Defined Aggregate Function）**：聚合函数，定义聚合计算逻辑。
    

---

### 2.原理

**[1]. 查询解析（Parsing）**：
   * SQL 会被 Flink 内部的 Apache Calcite 解析为逻辑查询计划（Logical Plan）。
     
**[2]. 验证与转换（Validation & Conversion）**：
   * 系统会验证字段、函数等是否合法，并将逻辑计划转换为 Flink 自身的中间表示。
     
**[3]. 查询优化（Query Optimization）**：
   * 使用基于规则的优化器进行查询重写，包括谓词下推、子查询合并、连接重排序等。
     
**[4]. 物理计划生成（Physical Plan）**：
   * 逻辑计划会被转换为物理计划，包含并行度、数据分区、执行算子等实际执行策略。
     
**[5]. 任务生成与执行（Job Generation & Execution）**：
   * 最终生成的物理计划会被翻译成 Flink 的 DataStream 或 Transformation DAG 提交执行。

**[P] 表与流的桥梁 - 动态表：**

Flink 在内部使用** 动态表（Dynamic Table）**模型，将流抽象为持续变化的表格，支持 INSERT/UPDATE/DELETE 三种变更事件，与传统关系型数据库类似。

类型如下：

| 类型                         | 描述                                                     | 典型用途                |
| -------------------------- | ------------------------------------------------------ | ------------------- |
| **Append-only（仅追加表）**      | 只包含 INSERT 操作，不存在 UPDATE 或 DELETE                      | 日志、事件流，如点击流分析       |
| **Upsert（更新插入表）**          | 包含 INSERT 和 UPDATE（以主键为基础）                             | 有主键的状态更新，如统计计数      |
| **Retract（撤回表）**           | 包含 INSERT、UPDATE、DELETE 三种操作，需要携带之前数据的撤回               | 没有主键但仍有更新/删除需求的复杂聚合 |
| **Changelog Table（变更日志表）** | 泛指包含各种变更（INSERT、UPDATE\_BEFORE、UPDATE\_AFTER、DELETE）的表 | 适用于通用场景             |


---

### 3.实现
**[1]. 环境准备**

```java
// Java 示例
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
EnvironmentSettings settings = EnvironmentSettings.newInstance().inStreamingMode().build();
TableEnvironment tableEnv = TableEnvironment.create(settings);
```

---

**[2]. 数据源定义**

```java
// Kafka 源表，使用事件时间和水位线
CREATE TABLE user_events (
    user_id STRING,
    action STRING,
    event_time TIMESTAMP(3),
    WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND
) WITH (
    'connector' = 'kafka',
    'topic' = 'user_topic',
    'properties.bootstrap.servers' = 'localhost:9092',
    'format' = 'json',
    'scan.startup.mode' = 'earliest-offset'
);
```

---

 **[3]. 使用 Table API 执行窗口聚合**

```java
Table result = tableEnv.from("clicks") //表名
    .window(Tumble.over(lit(10).minutes()).on($("event_time")).as("w"))
    .groupBy($("user_id"), $("w"))
    .select(
        $("user_id"),
        $("w").start().as("window_start"),
        $("w").end().as("window_end"),
        $("user_id").count().as("cnt")
    );

```
---
**[4]. 使用 SQL 执行窗口查询**
```java
//滚动窗口
SELECT
    user_id,
    TUMBLE_START(event_time, INTERVAL '10' MINUTE) AS window_start,
    TUMBLE_END(event_time, INTERVAL '10' MINUTE) AS window_end,
    COUNT(*) AS cnt
FROM user_events
GROUP BY user_id, TUMBLE(event_time, INTERVAL '10' MINUTE);
//滑动窗口
SELECT
    user_id,
    HOP_START(event_time, INTERVAL '5' MINUTE, INTERVAL '15' MINUTE) AS window_start,
    HOP_END(event_time, INTERVAL '5' MINUTE, INTERVAL '15' MINUTE) AS window_end,
    COUNT(*) AS cnt
FROM user_events
GROUP BY user_id, HOP(event_time, INTERVAL '5' MINUTE, INTERVAL '15' MINUTE);

```

**[5]. 将结果输出到 Sink**
```java
//方式1
tableEnv.executeSql("""
    CREATE TABLE sink (
        user_id STRING,
        cnt BIGINT
    ) WITH (
        'connector' = 'print'
    )
""");

result.executeInsert("sink");
//方式2:
CREATE TABLE sink_result (
    user_id STRING,
    cnt BIGINT,
    window_start TIMESTAMP(3),
    window_end TIMESTAMP(3)
) WITH (
    'connector' = 'print'
);

-- 插入数据
INSERT INTO sink_result
SELECT user_id, cnt, window_start, window_end
FROM result_table;

```

**[6]. CDC实时同步实现**
```SQL
//以mysql为例
CREATE TABLE mysql_binlog (
    id INT,
    name STRING,
    description STRING,
    ts TIMESTAMP(3),
    WATERMARK FOR ts AS ts - INTERVAL '5' SECOND,
    PRIMARY KEY (id) NOT ENFORCED
) WITH (
    'connector' = 'mysql-cdc',
    'hostname' = 'localhost',
    'port' = '3306',
    'username' = 'flinkuser',
    'password' = 'flinkpwd',
    'database-name' = 'mydb',
    'table-name' = 'mytable'
);

```
**[7]. SQL Join**

**Regular Join**(常规join)：支持 left join、 right join 、 inner join 、cross join

* **语义：** join时，左表和右表都会被缓存（保留状态），等待可能的匹配，所以这种长久保留的状态可能导致状态无限增长，应配置TTL来回收State

* **场景：** 当数据表规模有限、流量可控，或两个表均为有界表
* **案例**
  ```SQL
   SELECT o.order_id, p.name, p.price
   FROM Orders AS o
   INNER JOIN Product AS p
    ON o.product_id = p.id;
   //orders 表和product 表的状态保存$ttl时长
  ```

**Interval Join**(区间 join)：

* **语义：** 仅支持 等值 join + 时间范围限制，即right.time  between  left.time-low and left.time+up

* **场景：** 顶点与支付，点击与曝光等

* **案例**
    ```SQL
      SELECT
   u.user_id,
   u.event_time    AS click_time,
   p.event_time    AS exposure_time
   FROM click_stream AS u
   INNER JOIN exposure_stream AS p
     ON u.user_id = p.user_id
    AND p.event_time BETWEEN u.event_time - INTERVAL '5' SECOND
                         AND u.event_time + INTERVAL '5' SECOND;

    ```
**Temporal  Join**(维度表 join)：

* 语义：将流数据与“历史快照”做时间关联，流表侧的每一条记录都会按其事件时间到维度表做一次时点查询。

* 关键字：FOR SYSTEM_TIME AS OF

* 场景：将动态费率规则应用到交易事件。

* 案例：
   ```SQL
   -- 维度表需定义为带有版本时间属性的表
  CREATE TABLE user_dim (
    user_id    STRING,
    name       STRING,
    gender     STRING,
    ts         TIMESTAMP(3),
    PRIMARY KEY (user_id) NOT ENFORCED
  ) WITH (
    'connector' = 'jdbc',
    …,
    'lookup.cache.max-rows' = '5000',
    'lookup.cache.ttl' = '10min'
  );
  
  -- 订单流关联到维表
  SELECT
    o.order_id,
    o.user_id,
    d.name,
    d.gender,
    o.ts
  FROM orders AS o
  LEFT JOIN user_dim FOR SYSTEM_TIME AS OF o.ts AS d
    ON o.user_id = d.user_id;
  
   ```
**Lookup Join（异步维表查询）**  

* **语义：** 于异步 I/O，从外部系统（例如 MySQL、Redis、HBase）按需拉取最新维度数据。
* **场景：** 实时点击流中查用户等级
* **案例：**
  
  ```SQL
   CREATE TABLE rate_rule (
    rule_id    STRING,
    factor     DOUBLE,
    ts         TIMESTAMP(3),
    PRIMARY KEY (rule_id) NOT ENFORCED
  ) WITH (
    'connector' = 'jdbc',
    'url' = 'jdbc:mysql://.../db',
    'table-name' = 'rule_table',
    'lookup.cache.max-rows' = '10000',
    'lookup.cache.ttl' = '5min'
  );
  
  SELECT
    t.tx_id,
    t.amount,
    r.factor,
    t.amount * r.factor AS fee
  FROM tx_stream AS t
  LEFT JOIN rate_rule FOR SYSTEM_TIME AS OF t.ts AS r
    ON t.rule_id = r.rule_id;

  ```

**[8]. 自定义函数实现：**

* 继承对应抽象类或接口：`ScalarFunction`（UDF）、`TableFunction`（UDTF）、`AggregateFunction` 或 `TableAggregateFunction`（UDAF）。

* 由 Flink SQL 编译器将函数解析成算子，运行时执行函数调用。

* 函数参数和返回类型通过类型推断和 `TypeInformation`/`DataType` 支持。

**[9]. 窗口 Top案例**
  ```
   -- 1. 在窗口内按 product_id 聚合点击次数
   CREATE VIEW product_counts AS
   SELECT
     product_id,
     TUMBLE_START(ts, INTERVAL '1' HOUR) AS win_start,
     TUMBLE_END(ts,   INTERVAL '1' HOUR) AS win_end,
     COUNT(1)         AS cnt
   FROM clicks
   GROUP BY
     TUMBLE(ts, INTERVAL '1' HOUR),
     product_id;
   
   -- 2. 对聚合结果做 Top-N 排名，保留每个窗口前 3 名
   SELECT
     win_start,
     win_end,
     product_id,
     cnt
   FROM (
     SELECT
       *,
       ROW_NUMBER() OVER (
         PARTITION BY win_start, win_end
         ORDER BY cnt DESC
       ) AS rn
     FROM product_counts
   ) t
   WHERE rn <= 3;

  ```

---

### 4.场景


[1]. **实时指标分析**：
   * 业务场景如 PV/UV、用户活跃度、订单统计等，依赖低延迟数据聚合。
     
[2]. **ETL 管道构建**：
   * 通过 SQL 对接 Kafka、HBase、JDBC 等来源，对数据进行清洗、转换、写入目标存储。
     
[3]. **数据湖查询和管理**：
   * 集成 Apache Iceberg、DeltaLake、Hudi，实现数据湖上的增量处理和批流统一。
     
[4]. **多源 JOIN 分析**：
   * 利用 Temporal Table Join 实现实时与维度表的关联，例如用户画像分析。

---

### 5.常见问题
- **Q：数据类型不匹配**：
    
    >  A：尤其是 JSON 数据源自动推断类型时，建议显式定义 schema。
    > 
- **Q：状态膨胀与 OOM**：
    
    > A：聚合窗口未清理或高基数 key 导致状态无限增长，应结合 TTL 和窗口清理机制。
    > 
- **Q：延迟高或性能低**：
    
    > A：查询未优化或算子未并行。可通过 EXPLAIN 语句分析执行计划，调优算子并行度。
    > 
- **Q：JOIN 出现倾斜或乱序问题**：
    
    > A：特别是维表 JOIN 时，建议使用 BROADCAST 或 HASH JOIN，并结合 watermark 控制乱序。
    > 
- **Q：版本不兼容或连接器异常**：
    
    > A：Flink 不同版本之间 SQL 特性存在差异，连接器需与当前版本匹配，避免使用未经测试版本。
    > 
- **Q：作业无法恢复或状态损坏**：
    
    > A：状态后端未配置或 checkpoint 不完整，建议启用 RocksDB + 高可用配置。
    >
