

### 1.概念

**[1]. 核心抽象：**

   * **DataStream**
     `DataStream` 是 Flink 中的核心抽象，代表了一个不可变的分布式数据集合。可以通过各种操作符进行转换，且它本身可以是有界的或无界的。

   * **Transformation**
     `Transformation` 表示对 `DataStream` 进行的操作。这些操作包括如 `map`、`filter`、`keyBy`、`flatMap`、`window` 等，它们是编写流处理程序的基本单元。

   * **Operator**
     `Operator` 是 `Transformation` 在运行时的物理实现。在 Flink 的执行模型中，`Transformation` 被编译成一系列的 `Operator`，这些 `Operator` 执行具体的数据处理逻辑，运行在不同的节点上。

**[2]. 定义：**
   DataStream API 是基于流式计算模型的编程接口，用于处理无界（infinite）或有界（finite）数据流。

**[3]. 数据抽象类型：**

   * `DataStream<T>`：表示一条元素类型为 T 的数据流。
   * `SingleOutputStreamOperator<T>`：表示经过操作符处理后的结果流。

**[4]. 与 Table API 的区别：**

   * DataStream API 偏向底层、灵活；
   * Table API 更偏向声明式、SQL 风格。

---

### 2.原理

**[1]. 流式数据模型：**
   Flink 内部处理的数据为不断到达的事件（Event），事件以数据流的形式在任务中流动。

**[2]. 任务执行图（JobGraph 和 StreamGraph）：**
   用户编写的逻辑被转换为 StreamGraph，进一步优化后转为 JobGraph，由 JobManager 执行。

**[3]. 事件时间与时间语义：**
   支持三种时间语义：**处理时间（Processing Time）**、 **摄取时间（Ingestion Time）**、**事件时间（Event Time）**
   事件时间可配合 watermark 实现乱序事件处理。

**[4]. 状态管理与容错机制：** 存储任务状态（如 RocksDB、HashMap 等）

**[5]. 并行计算与数据分区：** Operator 并行度控制任务性能；
    数据流可通过 keyBy、rebalance、rescale 等方式进行分区重组。

---

### 3.实现

**[1]. 基本结构：**
   一个标准 Flink DataStream 应用包括：

   * 创建执行环境（`StreamExecutionEnvironment`）
   * 定义数据源（Source）
   * 进行转换操作（Transformation）
   * 输出结果（Sink）
   * 启动执行（`env.execute()`）
**[2]. 环境配置：**
    
  启用checkpoint，设置状态后端，时间语义，TTL 
    ```
   // 启用 Checkpoint
    env.enableCheckpointing(5000);  // 每5秒钟执行一次检查点
    
    // 设置 checkpoint 的容错模式
    env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
    
    //状态后端设置rocksdb
    env.setStateBackend(new RocksDBStateBackend("hdfs://checkpoint-dir"));
    
    //事件时间需要设置乱序程度和引用字段
    stream.assignTimestampsAndWatermarks(
    WatermarkStrategy
        .forBoundedOutOfOrderness(Duration.ofSeconds(5))  // 允许最大 5 秒的乱序
        .withTimestampAssigner((event, timestamp) -> event.getTimestamp())  // 设置事件的时间戳
    )
    
    //设置ttl【 对 状态 存储进行过期清理的机制】
    stateTtlConfig = StateTtlConfig.newBuilder(Time.minutes(5))
    .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)//创建和写入时更新
    .build();
    ```
**[3]. 常见 Source 类型：**

   * 内建 Source：如 `fromElements`, `socketTextStream`
   * Kafka、Pulsar 等外部连接器
   * 自定义 Source：实现 `SourceFunction` 或 `Source` 接口

**[4]. 不同时间语义机制的实现方式：**
  **处理时间（Processing Time）**
  **摄取时间（Ingestion Time）**
  **事件时间（Event Time）**

**[5]. 转换操作符（Transformations）：**
   * Map、FlatMap、Filter、KeyBy、Window、Reduce、Process、Connect、Union、CoFlatMap 等
  具体含义暂略，日常开发，可随用随查(个人观点)

**[6]. 窗口操作（Window）：**

   * **时间窗口：** 滚动窗口(Tumbling Window)、滑动窗口(Sliding Window)、会话窗口(Session Window)
   * **窗口分配器和触发器：** 通过 `WindowAssigner` 和 `Trigger` 精细控制窗口行为

**[7]. 状态操作（State）：**

   * Keyed State 与 Operator State
   * 使用 `ValueState`、`ListState` 等接口处理有状态逻辑
   * 需通过 `RuntimeContext` 获取 State 句柄

**[8]. Sink 操作：**

   * 标准输出
   * Kafka、HDFS、Elasticsearch、JDBC 等外部系统
   * 自定义 Sink（继承 `SinkFunction`）
   * Sink端两阶段提交 
  ``` java
    public class MyTwoPhaseCommitSink extends TwoPhaseCommitSinkFunction<MyDataType, String, String> {

     // Step 1: 开始事务
    @Override
    protected String beginTransaction() throws Exception {
        // 创建事务 ID 或获取外部系统中的事务上下文
        return "tx_123";
    }

    // Step 2: 写入数据到外部系统
    @Override
    protected void invoke(MyDataType value, String transaction) throws Exception {
        // 在此执行写入操作，写入到外部系统，但不提交
        writeToExternalSystem(value, transaction);
    }

    // Step 3: 提交事务
    @Override
    protected void commitTransaction(String transaction) throws Exception {
        // 提交事务到外部系统
        commitToExternalSystem(transaction);
    }

    // Step 4: 回滚事务
    @Override
    protected void rollbackTransaction(String transaction) throws Exception {
        // 如果失败，回滚事务
        rollbackInExternalSystem(transaction);
    }
    
    private void writeToExternalSystem(MyDataType value, String transaction) {
        // 执行写入逻辑，写入数据，但不提交
    }

    private void commitToExternalSystem(String transaction) {
        // 提交事务，完成数据写入
    }

    private void rollbackInExternalSystem(String transaction) {
        // 回滚事务，撤销数据写入
    }
    }

    // 使用 TwoPhaseCommitSinkFunction
    DataStream<MyDataType> stream = ...;
    stream.addSink(new MyTwoPhaseCommitSink());

  ```

---

### 4.场景

**[1]. 实时日志处理：**

   * 对日志流进行解析、清洗、聚合，如 Nginx、业务日志
   * 支持动态规则、状态管理

**[2]. 实时监控告警系统：**

   * 异常检测：如指标突变、访问频次异常
   * 结合 CEP（复杂事件处理）构建事件模式匹配

**[3] 实时ETL（Extract-Transform-Load）：**

   * 从 Kafka 等采集系统获取数据
   * 进行格式转换、字段清洗后写入数仓或 OLAP 系统


---

### 5.常见问题

**Q：Watermark 设置不合理：**

  > A：延迟太小导致数据丢失，太大会影响计算延迟
  > 解决方式：合理评估数据乱序程度，使用 `BoundedOutOfOrdernessWatermarks`

**Q：状态膨胀与 OOM：**

  > A：Key 数量过多或状态未清理
  > 解决：TTL 设置、状态压缩、聚合优化

**Q：窗口未触发：**

  > A：Watermark 不推进或触发器未触发
   >检查数据是否持续到来，触发器配置是否正确

**Q：Checkpoint 失败：**

  > A：外部系统阻塞、状态太大
  >使用异步 Checkpoint、优化 Sink、切分大状态

