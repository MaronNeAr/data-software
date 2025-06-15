## 第一章 Flink概述

#### 流处理与批处理的统一架构

###### 流处理（Stream Processing）为核心

- Flink 将批处理（Batch）视为流处理（Streaming）的特例（即有限数据流），通过统一的运行时引擎支持流批一体
- 核心抽象：**DataStream API**（流处理）和 **DataSet API**（批处理，逐步向 **Table/SQL API** 迁移）。

###### 有界流与无界流

- **无界流**（Unbounded Streams）：持续产生、无终点的数据（如 Kafka 消息流）。
- **有界流**（Bounded Streams）：有限、有终点的数据（如静态文件）。

###### 统一的运行时引擎

- 所有任务最终转换为 **流式执行模型**，通过调度策略和资源管理实现批处理的优化。

#### Flink 的核心优势与适用场景

###### Flink核心优势

- **低延迟与高吞吐**
  - 基于流水线（Pipeline）执行模式，避免中间数据落盘，实现毫秒级延迟。
  
  - 异步 Checkpoint 和增量快照减少对吞吐的影响。
  
- **精确一次（Exactly-Once）语义**：通过分布式快照（Checkpoint）和 Barrier 同步机制，确保状态一致性。

- **事件时间（Event Time）处理**：支持基于事件时间的窗口计算，结合 Watermark 机制处理乱序数据。

- **灵活的窗口机制**：提供滚动、滑动、会话窗口，支持自定义窗口逻辑。

- **状态管理与容错**：内置 Keyed State 和 Operator State，支持多种状态后端（如 RocksDB）。

- **动态扩缩容**：支持在运行中调整算子的并行度，适应负载变化。

###### Flink 的适用场景

- **实时数据分析**：实时监控、日志分析、实时大屏（如电商实时交易统计）。

- **事件驱动型应用**：复杂事件处理（CEP）、欺诈检测、实时推荐。

- **数据管道（ETL）**：高效的数据清洗、转换与加载（替代传统批处理框架）。

- **机器学习与迭代计算**：支持迭代计算模型（如梯度下降），与机器学习库（如 Alink）集成。

#### Flink技术栈与生态系统

###### 分层 API 设计

- **底层 API**：DataStream/DataSet API（细粒度控制）。
- **声明式 API**：Table API & SQL（关系型操作，基于 Apache Calcite 解析）。
- **领域库**：
  - **Flink CEP**：复杂事件处理。
  - **Flink ML**：机器学习。
  - **Stateful Functions**：有状态函数服务（Serverless 模式）。

###### 连接器（Connectors）

- 支持 Kafka、HDFS、HBase、JDBC、Elasticsearch 等外部系统。

###### 部署模式

- **独立集群（Standalone）**：快速本地测试。
- **YARN/Kubernetes**：生产级资源管理。
- **云原生集成**：AWS Kinesis、Google Dataflow 等。

###### 生态工具

- **Flink Web Dashboard**：作业监控与诊断。
- **Zeppelin Notebook**：交互式开发。
- **Savepoint 管理**：版本化状态迁移。

## 第二章 Flink 架构设计

#### 分层架构：API 层、运行时层、物理部署层

###### API 层

- **定位**：用户编程接口层，提供多种抽象级别的 API，满足不同开发需求。

- **DataStream API**：面向无界流处理，支持事件时间、状态管理、窗口计算等流处理特性。

  ```java
  DataStream<String> stream = env.socketTextStream("localhost", 9999);
  stream.filter(s -> s.contains("error")).print();
  ```

- **DataSet API**：面向批处理（有界数据流），逐步被 Table API 替代，但仍用于复杂批处理场景。

- **声明式编程**：基于关系型模型，使用 SQL 或类 LINQ 语法操作数据。

- **流批统一**：同一 SQL 可同时处理流和批数据（通过动态表机制）。

  ```sql
  SELECT user, COUNT(*) FROM logs GROUP BY user;
  ```

###### 运行时层（Runtime Layer）

- **定位**：执行引擎核心，负责作业调度、任务执行、资源管理及容错。

- **客户端（Client）**：提交用户程序生成的 JobGraph 到 JobManager。
- **JobManager**：
  - **调度器（Scheduler）**：将 JobGraph 转换为 ExecutionGraph，分配 TaskSlot。
  - **检查点协调器（Checkpoint Coordinator）**：触发 Checkpoint，确保状态一致性。
- **TaskManager**：执行具体任务（Task），管理 TaskSlot 资源。
- **数据流图（DataFlow Graph）**：将逻辑计划（如 StreamGraph）转换为物理执行计划。

- **任务链（Operator Chaining）**：将多个算子合并为一个 Task，减少序列化与网络开销。

- **任务调度**：
  - **贪心策略（Eager Scheduling）**：流处理默认方式，一次性申请所有资源。
  - **延迟调度（Lazy Scheduling）**：批处理优化策略，按阶段申请资源。
- **反压（Backpressure）**：通过动态调整数据生产速率避免下游过载。
- **内存管理**：使用堆外内存（Network Buffers）减少 GC 影响，提升吞吐。

###### 物理部署层（Deployment Layer）

- **定位**：集群资源管理与任务部署层，支持多种分布式环境。

- **Standalone 模式**：
  - 独立集群，手动管理资源，适合测试与小规模场景。
  - 架构：固定数量的 JobManager 和 TaskManager。
- **YARN 模式**：
  - 与 Hadoop 生态集成，按需申请资源（动态扩展）。
  - **Session 模式**：共享集群资源，适合短作业。
  - **Per-Job 模式**：独占集群资源，适合长周期作业。
- **Kubernetes 模式**：
  - 容器化部署，支持弹性扩缩容与高可用。
  - 使用 Operator 管理 Flink 应用生命周期。

- **ResourceManager**：
  - 与外部资源框架（如 YARN、Kubernetes）交互，申请/释放资源。
  - 支持 Slot 共享（同一 TaskSlot 运行多个 Task 的子任务）。

- **高可用（HA）配置**

  - **ZooKeeper 协调**：存储 JobManager 元数据，实现 Leader 选举与状态恢复。

  - **持久化存储**：Checkpoint 和 Savepoint 存储于 HDFS/S3 等分布式存储系统。

#### 核心组件：JobManager、TaskManager、ResourceManager

######  JobManager（作业管理器）

- **定位**：集群的“大脑”，负责作业调度、协调及容错管理。

- **作业调度与执行计划管理**
  - 接收客户端提交的 **JobGraph**（逻辑执行计划），将其转换为 **ExecutionGraph**（物理执行计划）。
  
  - 将 ExecutionGraph 拆分为 **Task**（任务），分配至 TaskManager 的 **TaskSlot**（任务槽）。
  
- **Checkpoint Coordinator**：
  - 触发周期性 Checkpoint，通过 **Barrier 注入** 同步所有算子的状态快照。
  - 确保 Exactly-Once 语义的一致性。

- **故障恢复**：检测 TaskManager 或 Task 故障，重新调度失败的任务，并从最近的 Checkpoint 恢复状态。

- **高可用性（HA）支持**：依赖 **ZooKeeper** 实现 Leader 选举与元数据存储，避免单点故障。

- **Dispatcher**：接收客户端提交的作业，启动 JobMaster。
- **JobMaster**：管理单个作业的生命周期（调度、容错）。
- **ResourceManager**（Flink 内置）：与外部 ResourceManager（如 YARN）交互，申请资源。

###### TaskManager（任务管理器）

- **定位**：集群的“工作者”，负责执行具体任务（Task）并管理本地资源。

- **任务执行**
  - 运行 **Task**（算子的并行实例），每个 Task 对应一个线程。
  
  - 支持 **算子链（Operator Chaining）**：将多个算子合并为一个 Task，减少网络开销。
  
- **网络通信**

  - 管理 **InputGate**（输入数据）和 **ResultSubpartition**（输出数据）。

  - 基于 **Netty** 实现高效异步数据传输，支持反压（Backpressure）机制。

- **内存管理**

  - 分配 **TaskSlot** 内存，包括 Network Buffers（网络传输）、Managed Memory（状态存储）和 JVM 堆内存。

  - 通过堆外内存减少 GC 压力，提升吞吐量。

- **状态存储**：与 **State Backend**（如 RocksDB）交互，持久化本地状态至磁盘或远程存储。

- **TaskSlot**：
  - 资源隔离的最小单位，每个 TaskManager 包含多个 Slot（默认根据 CPU 核数分配）。
  - 支持 **Slot 共享**：同一 Slot 可运行同一作业的多个 Task，提升资源利用率。

###### ResourceManager（资源管理器）

- **定位**：集群资源的管理者，负责与底层资源系统（如 YARN、Kubernetes）交互。

- **资源申请与释放**

  - 根据 JobManager 的请求，向外部资源框架（如 YARN、K8s）申请 TaskManager 资源。

  - 在作业完成或失败时释放资源。

- **Slot 管理**

  - 维护集群 Slot 资源池，跟踪可用/已用 Slot 状态。

  - 支持动态 Slot 分配（如扩缩容场景）。

- **多集群模式适配**

  - **Standalone 模式**：直接管理固定数量的 TaskManager。

  - **YARN 模式**：与 YARN ResourceManager 交互，按需启动容器。

  - **Kubernetes 模式**：通过 Kubernetes API 动态创建 Pod（TaskManager）。

- **资源分配策略**

  - **贪心策略（Eager）**：流处理作业默认一次性申请所有资源。

  - **延迟策略（Lazy）**：批处理作业分阶段申请资源，优化资源利用率。

###### 核心组件的交互流程

- **客户端提交作业**：用户程序通过 `flink run` 提交 JobGraph 至 JobManager 的 Dispatcher。

- **JobManager 解析作业**：JobMaster 将 JobGraph 转换为 ExecutionGraph，向 ResourceManager 申请 Slot。

- **ResourceManager 分配资源**

  - 若 Slot 不足，向外部资源系统（如 YARN）申请启动新的 TaskManager。

  - 将可用 Slot 分配给 JobMaster。

- **TaskManager 执行任务**

  - JobMaster 将 Task 部署到 TaskManager 的 Slot 中。

  - TaskManager 启动 Task 线程，执行数据计算与传输。

- **Checkpoint 与故障恢复**

  - JobManager 触发 Checkpoint，TaskManager 持久化状态。

  - 若 TaskManager 宕机，JobManager 重新调度 Task 至其他 Slot，并从 Checkpoint 恢复状态。

#### 任务提交流程与资源分配机制

###### 客户端提交作业

- 用户通过命令行工具（`flink run`）或编程方式（`StreamExecutionEnvironment.execute()`）提交作业。

- **程序逻辑转换为逻辑执行图**

  - **DataStream/DataSet API**：生成 **StreamGraph**（流处理）或 **Plan**（批处理）。

  - **Table/SQL API**：生成 **Optimized Logical Plan**（基于 Calcite 优化）。

- **生成 JobGraph**
  - 逻辑执行图转换为 **JobGraph**（物理执行计划的中间表示），包含：
    - **JobVertex**：算子节点（如 `map`, `filter`）。
    - **IntermediateDataSet**：数据交换边（如 forward、hash 分区）。

- **算子链优化**：将可合并的算子合并为单个 Task，减少序列化与网络开销。

- **提交 JobGraph 到 JobManager**
  - 客户端通过 REST 或 RPC 将 JobGraph 提交到 JobManager 的 **Dispatcher**。

######  JobManager 处理作业

- **Dispatcher** 接收 JobGraph 后，启动 **JobMaster** 管理作业生命周期。

- **解析 JobGraph 为 ExecutionGraph**

  - **ExecutionGraph** 是 JobGraph 的并行化版本，每个 JobVertex 拆分为多个 **ExecutionVertex**（并行子任务）。

  - 例如：并行度为 3 的 `map` 算子对应 3 个 ExecutionVertex。

- **申请资源**

  - JobMaster 向 **ResourceManager** 申请 TaskManager 的 **TaskSlot**。

  - 若当前资源不足，ResourceManager 向外部系统（如 YARN、K8s）申请启动新的 TaskManager。

- **分配 Slot 并部署 Task**

  - ResourceManager 返回可用 Slot 列表，JobMaster 将 ExecutionVertex 分配到 Slot。

  - 通过 **TaskDeploymentDescriptor** 将 Task 代码、配置、状态信息发送至 TaskManager。

###### TaskManager 执行 Task

- **TaskManager** 接收部署请求后启动 Task。

- **初始化 Task**：加载用户代码（如 `map` 函数），初始化状态后端（如 RocksDB）。

- **建立数据通道**
  - **InputGate**：接收上游数据（如 Kafka 数据源）。
  
  - **ResultSubpartition**：发送下游数据（如聚合结果）。
  
- **启动计算线程**：每个 Task 在一个独立线程中执行，通过事件循环处理数据。

###### 资源分配机制

- **TaskSlot 管理**：**TaskSlot** 是资源分配的最小单位，每个 TaskManager 包含多个 Slot（默认等于 CPU 核数）。
- **Slot 共享**：
  - 同一作业的不同 Task 可共享同一 Slot（如 `source -> map -> sink` 合并为一个 Task）。
  - 提升资源利用率，减少跨 Slot 的网络开销。

- **资源申请策略**

  - **贪心调度（Eager Scheduling）**：一次性申请所有 Task 所需的 Slot，流处理默认策略。

  -	**延迟调度（Lazy Scheduling）**：按阶段（Stage）申请资源，完成前阶段任务后释放资源，批处理优化策略。

- **动态资源扩缩容**
- **资源不足时的扩容**：JobMaster 检测到 Slot 不足时，触发 ResourceManager 申请新 TaskManager。
  
- **资源空闲时的缩容**：若 Slot 长时间空闲，ResourceManager 释放 TaskManager（需配置超时阈值）。

#### 高可用性（HA）设计原理

###### 基于 ZooKeeper 的 Leader 选举

- **角色**：ZooKeeper 是分布式协调服务，用于管理集群元数据、监控节点状态及选举 Leader。

- **选举流程**：

  1. 多个 JobManager 节点启动后，向 ZooKeeper 注册临时节点（如 `/leader`）。
  2. ZooKeeper 通过 **临时顺序节点** 确定 Leader（编号最小的节点为 Leader）。
  3. 若 Leader 宕机，临时节点消失，剩余节点重新选举新 Leader。

- **关键配置**：

  ```
  high-availability: zookeeper
  high-availability.zookeeper.quorum: zk1:2181,zk2:2181,zk3:2181
  high-availability.storageDir: hdfs:///flink/ha/
  ```

###### 元数据持久化

- **元数据类型**：
  - **JobGraph**：作业的逻辑执行计划。
  - **ExecutionGraph**：物理执行计划（包含 Task 分配信息）。
  - **Checkpoint 元数据**：Checkpoint 的路径、状态句柄等。
- **持久化存储**：
  - 元数据存储于 **高可用存储系统**（如 HDFS、S3）。
  - 通过配置 `high-availability.storageDir` 指定存储路径。

###### Checkpoint 与 Savepoint

- **Checkpoint**：
  - **周期性快照**：TaskManager 定期将算子状态持久化到分布式存储。
  - **恢复依据**：故障后，新 JobManager 从最近一次 Checkpoint 恢复作业状态。
- **Savepoint**：
  - **用户触发的全量快照**：用于版本升级、作业迁移或调试。
  - 与 Checkpoint 不同，Savepoint 包含完整的作业状态（即使作业已停止）。

## 第三章 Flink 运行时（Runtime）

#### 作业执行流程：Client 到 JobManager 的交互

###### 客户端生成并提交 JobGraph

- **用户代码转换**：用户通过 `DataStream`/`DataSet`/`Table` API 编写程序，定义数据处理逻辑（如 `map`, `filter`, `window`）。

  ```java
  StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
  DataStream<String> dataStream = env.socketTextStream("localhost", 9999);
  dataStream.map(str -> str.toUpperCase()).print();
  env.execute("Example Job");
  ```

- **生成逻辑执行图：**

  - **StreamGraph**：流处理程序的逻辑图，表示算子的拓扑关系（如 Source → Map → Sink）。
  - **Optimized Plan**：Table/SQL 作业经 Calcite 优化后的逻辑计划。

- **优化与生成 JobGraph：**

  - **算子链（Operator Chaining）**：将相邻算子合并为单个 Task（如 `Source → Map` 合并）。
  - **数据分区优化**：选择最优数据分发策略（如 Hash、Rebalance）。
  - **JobGraph**：逻辑执行图的物理表示，包含 `JobVertex`（算子节点）、`IntermediateDataSet`（数据边）。
  - **提交单元**：Client 将 JobGraph 序列化后提交至 JobManager。

######  Client 提交 JobGraph 到 JobManager

- **连接 JobManager**：
  - Client 通过 **REST API** 或 **RPC**（Akka）连接到 JobManager 的 **Dispatcher**。
  - **Per-Job 模式**：每个作业独占一个 JobManager（生产环境推荐）。
  - **Session 模式**：多个作业共享一个 JobManager（测试环境常用）。
- **提交 JobGraph**：
  - Client 发送 JobGraph 和依赖的 JAR 包到 Dispatcher。
  - **提交确认**：Dispatcher 返回 `JobID`，标识作业的唯一性。

###### JobManager 处理作业请求

- **解析 JobGraph 为 ExecutionGraph**：

  - **ExecutionGraph**：JobGraph 的并行化版本，每个 `JobVertex` 拆分为多个 **ExecutionVertex**（并行子任务）。

  - **并行度分配**：根据用户配置（`setParallelism(4)`）或默认值生成子任务。

- **申请资源**：

  - JobMaster 向 **ResourceManager** 申请所需数量的 **TaskSlot**。

  - **资源不足时的处理**：
    - **Standalone 模式**：直接失败（需预先启动足够 TaskManager）。
    - **YARN/Kubernetes 模式**：ResourceManager 动态申请新 TaskManager。

- **分配 Slot 并部署 Task**：

  - ResourceManager 返回可用 Slot 列表。

  - JobMaster 将 **ExecutionVertex** 分配到 Slot，生成 **TaskDeploymentDescriptor**（包含代码、配置、状态信息）。

  - 通过 **RPC** 将 Task 部署到对应 TaskManager。

###### TaskManager 执行 Task

- **接收部署请求**：TaskManager 的 **TaskExecutor** 接收 `TaskDeploymentDescriptor`。
- **初始化 Task**：
  - **加载用户代码**：通过 Child-first 类加载机制隔离不同作业的类。
  - **初始化状态后端**：如 RocksDB 状态后端加载本地磁盘路径。
- **启动计算线程**：
  - **InputGate** 接收上游数据（如 Kafka 消息）。
  - **ResultSubpartition** 发送处理结果到下游。
- **注册心跳机制**：TaskManager 定期向 JobManager 发送心跳，上报 Task 状态和 Slot 使用情况。

###### 关键机制与设计

- **算子链（Operator Chaining）优化**：减少网络传输和线程切换开销。

- **资源调度策略**

  - **流处理（Eager 调度）**：一次性申请所有资源，快速启动全链路计算【低延迟要求的实时处理】。

  - **批处理（Lazy 调度）**：分阶段申请资源，释放已完成阶段的 Slot【资源受限的离线计算】。

- **容错与状态恢复**

  - **Checkpoint 触发**：JobManager 定期向所有 Task 发送 Barrier，触发状态快照。

  - **故障恢复**：JobManager 从持久化的 ExecutionGraph 和 Checkpoint 恢复作业。

#### 任务调度策略：贪心调度、延迟调度

###### 贪心调度（Eager Scheduling）

- **核心思想**：

  - **一次性分配所有资源**：在作业启动时，立即申请所有 Task 所需的资源（TaskSlot），并同时部署所有 Task。

  - **流处理默认策略**：适用于无界数据流处理，追求低延迟和持续计算能力。

- **资源申请**：
  - JobMaster 解析 ExecutionGraph 后，立即向 ResourceManager 申请所有 Task 所需的 Slot。
  - 若 Slot 不足，ResourceManager 动态启动新的 TaskManager（如 YARN/K8s 模式）。
- **任务部署**：所有 Task 同时部署到分配的 Slot 中，形成完整的数据处理流水线。
- **立即执行**：Task 启动后直接开始处理数据，无需等待其他阶段完成。

###### 延迟调度（Lazy Scheduling）

- **核心思想**：

  - **分阶段申请资源**：将作业拆分为多个阶段（Stage），按需申请资源，完成前一阶段任务后释放资源。

  - **批处理优化策略**：适用于有界数据流（批处理），优化资源利用率。

- **阶段划分**：

  - 根据数据依赖关系，将 ExecutionGraph 拆分为多个 Stage（如 Source → Map → Reduce）。
  - 每个 Stage 内的 Task 可并行执行。

- **按阶段调度**：

  - 仅当某阶段的所有前置任务完成后，才申请该阶段所需的 Slot。
  - 阶段任务完成后，立即释放 Slot 供后续阶段复用。

- **增量执行**：先调度 Source 和 Map 阶段，待其完成后，再调度 Reduce 阶段。

###### 两种策略的对比

| **特性**         | **贪心调度（Eager）**         | **延迟调度（Lazy）**         |
| :--------------- | :---------------------------- | :--------------------------- |
| **资源分配时机** | 作业启动时一次性分配所有 Slot | 按阶段分配，完成后释放 Slot  |
| **延迟**         | 低（毫秒级）                  | 较高（依赖阶段间隔）         |
| **吞吐量**       | 高（流水线执行）              | 中等（阶段间需落盘中间数据） |
| **资源占用**     | 高（长期占用 Slot）           | 低（Slot 动态复用）          |
| **适用场景**     | 流处理（无界数据）            | 批处理（有界数据）           |
| **容错复杂度**   | 简单（全链路 Checkpoint）     | 复杂（需保存中间结果）       |

#### 任务槽（Task Slot）与资源隔离机制

###### 任务槽（Task Slot）的核心概念

- **Task Slot**：TaskManager 中的资源单元，代表固定比例的内存和（可选）CPU 资源。
- **资源分配**：每个 Slot 可运行一个或多个 Task 的子任务（通过 Slot 共享）。
- **资源隔离**：防止不同 Task 之间因资源竞争（如内存溢出）导致相互干扰。
- **内存划分**：
  - **TaskManager 总内存** = JVM 堆内存 + 堆外内存（Network Buffers、Managed Memory）。
  - **单个 Slot 内存** = TaskManager 总内存 / Slot 数量。
- **CPU 分配**：
  - **默认不隔离**：Slot 间共享 CPU 资源（依赖操作系统调度）。
  - **可选隔离**：通过 cgroups（Linux）或 Kubernetes CPU 限制实现物理核绑定。

###### Slot 的共享机制

- **Slot 共享组（Slot Sharing Group）**：同一作业的所有 Task 默认属于 `default` 共享组，可共享同一 Slot。
- **并行度对齐**：共享同一 Slot 的 Task 必须具有相同的并行度。
- **资源隔离**：不同 Slot 共享组的 Task 不能共享 Slot（如 `group1` 和 `group2`）。

###### 资源隔离机制

- **内存隔离**：
  - **Managed Memory**：用于状态存储（RocksDB）、批处理排序等，按 Slot 数均分。
  - **Network Memory**：用于网络传输的 Buffers，按 Slot 数均分。
  - **JVM 堆内存**：用户代码及 Flink 框架内存，由 JVM 统一管理。
- **CPU 隔离（可选）**
  - **Standalone/YARN 模式**：无严格 CPU 隔离，依赖操作系统时间片轮转调度。
  - **Kubernetes 模式**：通过 `resource.requests.cpu` 和 `limits.cpu` 为每个 TaskManager Pod 分配 CPU 配额。
- **类加载器隔离**
  - **Child-first 类加载**：用户代码（如 UDF）优先使用作业自身的依赖，避免与 Flink 框架类冲突。

###### Slot 的申请与分配流程

- **Slot 申请**

  - **JobManager** 根据 ExecutionGraph 的并行度计算所需 Slot 总数。

  - **ResourceManager** 从 Slot 池中选择可用 Slot，优先复用同一 TaskManager 的 Slot。

- **Slot 分配策略**

  - **就近分配**：优先选择同一 TaskManager 的 Slot 部署同一算子的子任务，减少网络传输。

  - **均衡分配**：避免单个 TaskManager 负载过高。

- **Slot 释放**

  - **作业完成时**：所有 Slot 释放回 ResourceManager。

  - **动态扩缩容**：在 Kubernetes 中，缩容时先驱逐空闲 TaskManager。

#### 内存管理：堆内/堆外内存模型、序列化框架（TypeInformation）

###### 堆内内存（JVM Heap）

- 存储用户代码中的对象（如 `MapFunction` 中的临时变量）。
- Flink 框架自身的堆内数据结构（如部分元数据）。

###### 堆外内存（Off-Heap）

- **Network Buffers**：缓存待发送或接收的网络数据，减少频繁的系统调用。
- **Managed Memory**：预分配的内存池，用于状态存储（RocksDB）、批处理排序等。

###### 序列化框架（TypeInformation）

- **性能问题**：Java 序列化冗长且低效，生成的对象反序列化成本高。
- **类型安全**：泛型类型擦除导致运行时类型信息丢失（如 `Tuple2<String, Integer>`）。
- **类型推断**：自动推导数据流的类型（如 `DataStream<String>`）。
- **生成序列化器**：根据类型创建高效的 `TypeSerializer`。
- **类型检查**：在作业提交前捕获类型不匹配错误。
- **基础类型**：`String`、`Integer`、`Long` 等。
- **复合类型**：
  - **Tuple**：`Tuple2<T0, T1>`。
  - **POJO**：符合字段公开条件的 Java 类（如公共 getter/setter）。
  - **Row**：通用行类型，常用于 Table API。
- **特殊类型**：
  - **Value**：可变对象接口（如 `IntValue`），减少对象创建开销。
  - **Writable**：Hadoop 兼容类型（如 `Text`）。

#### 网络通信模型：基于 Netty 的异步通信与反压机制

###### 基于 Netty 的异步通信

- **Netty 的作用**

  - **异步非阻塞 I/O**：通过事件循环（EventLoop）线程池处理网络请求，避免线程阻塞。

  - **零拷贝优化**：使用堆外内存（`ByteBuf`）减少数据复制开销。

  - **连接管理**：复用 TCP 连接，降低频繁建连的成本。

- **数据传输流程**

  - **发送端（Producer）**：

    - Task 的 `ResultSubpartition` 将数据写入 **NetworkBuffer**（堆外内存）。

    - Netty 的 `Channel` 将 Buffer 封装为消息，异步发送至下游 TaskManager。

  - **接收端（Consumer）**：

    - Netty 接收数据，写入 `InputChannel` 的 **NetworkBuffer**。

    - Task 线程从 `InputGate` 读取 Buffer 并反序列化为对象。

- **内存管理**

  - **NetworkBuffer Pool**：全局共享的堆外内存池，每个 TaskManager 预分配固定数量的 Buffer（如 2048 个）。

  - **动态申请**：当 Buffer 不足时，临时申请额外内存，但可能触发反压。

###### 反压机制

- **反压的作用**

  - **问题场景**：下游处理速度 < 上游生产速度，导致数据堆积（如窗口计算耗时过长）。

  - **解决方案**：动态反馈上游降低数据发送速率，避免系统崩溃或数据丢失。

- **反压触发条件**

  - **本地反压**：`InputChannel` 的 Buffer 使用率超过阈值（如 90%）。

  - **全局反压**：下游 TaskManager 的 Buffer 资源耗尽，向上游传播反压信号。

- **反压实现机制**

  - **基于 TCP 的反压（Flink 1.5 之前）**：利用 TCP 的滑动窗口机制隐式反馈反压；

  - **基于信用值（Credit-Based）的反压（Flink 1.5+）**：显式反馈下游可用 Buffer 数量（信用值），上游按信用值发送数据。
