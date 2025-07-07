## Part I: Apache Iceberg Fundamentals

#### Introducing Apache Iceberg

###### 数据湖的演进与挑战

- **传统数据湖（Hive 表格式）的缺陷：**
  - **分区锁定**：查询必须显式指定分区字段（如 `WHERE dt='2025-07-06'`）。
  - **无原子性**：并发写入导致数据覆盖或部分可见。
  - **低效元数据**：`LIST` 操作扫描全部分区目录（云存储成本高）。
- **Iceberg 的革新目标：**解耦计算引擎与存储格式（支持 Spark/Flink/Trino 等）；提供 ACID 事务、模式演化和分区演化能力。 

###### Iceberg 核心设计目标

- **ACID 事务**
  - **基于快照的隔离**：写入生成新快照，查询读取历史一致性视图。
  - **乐观锁并发控制（OCC**）：冲突时自动重试或报错。
- **模式演化（Schema Evolution）**：安全地新增/重命名/删除列（无需重写数据文件）；向后兼容：旧查询可读新数据。
- **分区演化（Partition Evolution）**：动态修改分区策略（如从 `月` 改为 `天`），旧查询无需改写。
- **隐藏分区（Hidden Partitioning）**
  - 表定义分区变换（如 `hours(timestamp)`）。
  - 写入时自动计算分区值并记录在元数据中。
  - 查询时引擎根据 `WHERE` 条件过滤分区（用户无需知道分区字段名）。

```mermaid
flowchart TD
  subgraph 设计目标
    A[ACID 事务] --> A1[快照隔离]
    A --> A2[乐观并发控制]
    B[模式演化] --> B1[安全列变更]
    C[分区演化] --> C1[动态分区布局]
  end
```

###### Iceberg 架构分层

- **Catalog 层**：存储表的位置和元数据指针（支持 Hive/Nessie/JDBC）。
- **元数据层**：
  - **元数据文件（Metadata JSON）**：记录当前快照指针、模式、分区信息。
  - **清单列表（Manifest List）**：快照包含的所有清单文件及其统计信息。
  - **清单文件（Manifest File）**：列出数据文件路径、分区范围、行数等统计信息。
- **数据层**：实际存储文件（Parquet/ORC/Avro），按分区组织。

```mermaid
graph LR
  Catalog --> Metadata
  subgraph Metadata Layer
    Metadata --> ManifestList
    ManifestList --> ManifestFile
  end
  ManifestFile --> DataFile
  DataFile --> File1[Parquet]
  DataFile --> File2[ORC]
  DataFile --> File3[Avro]
  
  classDef green fill:#D5E8D4,stroke:#82B366;
  classDef yellow fill:#FFF2CC,stroke:#D6B656;
  classDef blue fill:#DAE8FC,stroke:#6C8EBF;
  
  class Catalog green;
  class Metadata,ManifestList,ManifestFile yellow;
  class DataFile,File1,File2,File3 blue;
```

###### Iceberg vs. 其他表格格式

| **特性**         | **Apache Iceberg** | **Delta Lake** | **Apache Hudi**      |
| :--------------- | :----------------- | :------------- | :------------------- |
| **ACID 支持**    | ✅ 快照隔离         | ✅ 事务日志     | ✅ 时间轴管理         |
| **分区演化**     | ✅ 无需重写数据     | ❌ 需重写数据   | ⚠️ 部分支持           |
| **计算引擎耦合** | ❌ 解耦（通用API）  | ⚠️ 强依赖 Spark | ⚠️ 强依赖 Spark/Flink |
| **云存储优化**   | ✅ 避免 LIST 操作   | ⚠️ 依赖元存储   | ⚠️ 依赖元存储         |

#### Iceberg表结构规范

###### 元数据层级结构

- **Catalog**：表入口点（如 Hive/Nessie），存储最新元数据文件位置
- **Metadata File (JSON)**：当前快照ID；表模式（Schema）；分区规范（Partition Spec）；历史快照列表。
- **Manifest List (Avro)**：快照包含的所有清单文件路径；每个清单文件的分区范围统计。
- **Manifest File (Avro)**：数据文件路径列表；文件格式（Parquet/ORC/AVRO）；列级统计（min/max/null计数）；文件所属分区。
- **Data File**：实际数据文件（列式存储）

```mermaid
graph LR
  A[Catalog] -->|指向| B[Metadata File<br><i>v1.metadata.json</i>]
  B -->|包含| C[Current Snapshot ID]
  B -->|包含| D[Schema]
  B -->|包含| E[Partition Spec]
  C -->|引用| F[Manifest List<br><i>snap-123456.avro</i>]
  F -->|列出| G[Manifest File 1.avro]
  F -->|列出| H[Manifest File 2.avro]
  G -->|引用| I[Data File 1.parquet]
  G -->|引用| J[Data File 2.parquet]
  H -->|引用| K[Data File 3.parquet]
```

###### 快照（Snapshot）机制

- **快照**：数据表在特定时间点的完整状态。
- **关键属性**
  - **snapshot-id**：唯一标识符
  - **timestamp-ms**：创建时间戳
  - **manifest-list**：关联的清单列表位置
- **快照生成**：每次写操作（INSERT/UPDATE/DELETE）生成新快照。
- **支持时间旅行**：`SELECT * FROM tbl TIMESTAMP AS OF '2025-07-01 10:00:00'`。

###### Manifest File

- **清单文件**：元数据核心载体（Avro格式）
- **文件位置**：云存储路径。
- **统计信息**：行数（`record_count`）；文件大小（`file_size`）；列级最小值/最大值；空值计数（`null_value_counts`）。
- **分区数据**：实际分区值。

###### 数据文件（Data File）格式

- **Parquet（默认推荐）**：列式存储；高效压缩；谓词下推优化。
- **ORC**：ACID 原生支持；更好的 Hive 兼容性。
- **Avro**：行式存储；模式演化友好。

## Part II: Working with Iceberg Tables

#### 基于Spark的Iceberg表操作

###### 表创建与配置

- **创建方式**
  - **Spark SQL**：`CREATE TABLE db.table...`
  - **DataFrame API**：`df.writeTo('db.table').create()`
  - **DDL命令**：`ALTER TABLE... SET TBLPROPERTIES`
- **高级配置**：
  - **format-version**：指定元数据版本（1/2）。
  - **write.target-file-size-bytes**：控制文件大小（默认512MB）。
  - **write.metadata.delete-after-commit.enabled**：自动清理元数据。

###### 数据写入模式

- **INSERT INTO**：`INSERT INTO table FROM source`

- **MERGE INTO**：

  ```sql
  MERGE INTO table t
  USING updates u ON t.id = u.id
  WHEN MATCHED THEN UPDATE SET *
  WHEN NOT MATCHED THEN INSERT *
  ```

- **COPY FROM**：`COPY INTO table FROM '/path/raw_data.parquet' FILE_FORMAT = (TYPE = PARQUET)`

###### 时间旅行查询

- **VERSION AS OF [sid]**：按快照ID查询。
- **TIMESTAMP AS OF [ts]**：按时间戳查询。

#### 模式演化

###### 模式演化的核心原则

- **向后兼容（Backward Compatibility）**：旧查询能读取新schema写入的数据。
- **向前兼容（Forward Compatibility）**：新查询能读取旧schema写入的数据。
- **Iceberg 保证**：所有演化操作均保持向后兼容。

###### 列操作

- **添加列**：立即生效，无需重写数据文件；旧数据自动填充 `null`；可指定默认值：`ADD COLUMN level INT DEFAULT 1`。

- **重命名列**

  - **操作限制**：不能重命名分区列;需保证向前/向后兼容（新名称和旧名称在元数据中共存）。

  - **底层原理**：在元数据中添加 `rename` 记录（保留原始列ID）；查询时自动映射：旧查询用旧名，新查询用新名。

- **删除列**：标记列为 `dropped`（仍存于元数据）；数据文件保留该列（物理未删除）；运行 `OPTIMIZE` 后物理移除。

- **类型提升（Type Promotion）**
  - **整型**：`int → long`
  - **浮点型**：`float → double`
  - **Decimal**：扩大精度（`decimal(10,2) → decimal(20,2)`）

- **嵌套结构演化**
  - **支持操作**：在 Struct 中添加/重命名/删除字段；修改 Map 的 value 类型；扩展 Array 元素类型。

#### Partitioning and Sorting

###### 分区基础与类型

- **时间分区（最常用）**：根据数据写入时间进行分区，days/months/hours。
- **范围分区（指标）**：根据数据范围进行数据分区，如粉丝数、播放量指标等。
- **桶分区（高基数列）**：对字段值进行哈希映射分桶进行数据分区，尤其是id等高基数字段。
- **值分区（枚举值）**：根据字段枚举值进行数据分区，如性别、年龄段等。
- **截断分区（字符串优化）**：截取字符串子串进行数据分区。

###### 隐藏分区（Hidden Partitioning）

- **传统分区问题**：用户需知道分区列名：`WHERE partition_col='value'`。

- **Iceberg 方案**：

  - **表定义分区转换**：`PARTITIONED BY days(event_time)`。

  - **引擎自动转换**：`WHERE event_time > X` → 分区值过滤。

  - **完全透明：**用户只需用业务列查询。

###### 排序优化技术

- **多维排序（Z-Order）**：将多列值映射到Z形空间曲线，保证多列值相近的行物理相邻。

| **技术**         | **适用场景**   | **优势**            | **限制**         |
| :--------------- | :------------- | :------------------ | :--------------- |
| **单列排序**     | 强过滤单列查询 | 简单高效            | 其他列无序       |
| **Z-Order**      | 2-4列组合过滤  | 多维数据局部性      | 列数增加效果下降 |
| **希尔伯特曲线** | 极高维数据     | 比Z-Order更高维优化 | 计算开销大       |

###### 分区与排序决策树

- **分区选择原则**：一级分区——时间（过滤量最大）；二级分区——桶分区（分散热点）。
- **排序选择原则**：1列——简单排序；2-4列——Z-Order。

#### Table Evolution

###### 快照管理（Snapshot Management）

- **清理快照**：`CALL system.expire_snapshots('db.table', TIMESTAMP [date]);`
- **回滚快照**：`CALL system.rollback_to_snapshot('db.table', [ts]);`
- **保留策略**：
  - **默认保留**：至少1个历史快照.
  - **配置参数**：`snapshot.expire.age.ms`（自动过期时间）

###### 分支（Branch）与标签（Tag）

- **分支操作**

  - **创建开发分支**：`ALTER TABLE table CREATE BRANCH dev AS OF VERSION 1;`

  - **分支写入数据**：`INSERT INTO dev VALUES (...);`

  - **合并到主分支**：`MERGE INTO main t USING dev s ON t.id = s.id WHEN NOT MATCHED THEN INSERT *;`

- **标签管理**
  - **创建生产标签**：`ALTER TABLE table CREATE TAG v1 AS OF VERSION 1;`
  - **查询标签数据**：`SELECT * FROM table VERSION AS OF 'v1';`

###### 数据同步（Hive → Iceberg）

- **元数据转换**：将Hive Metastore元数据转为Iceberg格式。
- **数据注册**：将现有数据文件注册到Iceberg元数据。
- **验证**：数据完整性检查（行数、校验和）。
- **Spark迁移命令**：`ALTER TABLE hive_table CONVERT TO ICEBERG;`

###### 演化操作决策树

```mermaid
graph TD
  A[需要表级变更？] --> B{变更类型}
  B -->|恢复数据| C[快照回滚]
  B -->|实验开发| D[分支操作]
  B -->|版本标记| E[标签管理]
  B -->|Hive迁移| F[表转换]
  B -->|性能调优| G[属性修改]
  B -->|存储优化| H[元数据维护]
  
  C --> I[rollback_to_snapshot]
  D --> J[CREATE/INSERT BRANCH]
  E --> K[CREATE TAG]
  F --> L[CONVERT TO ICEBERG]
  G --> M[SET TBLPROPERTIES]
  H --> N[rewrite_manifests]
```



## Part III: Advanced Features and Optimization

#### Transaction Processing



#### Performance Tuning



#### Iceberg on Cloud Storage



#### Streaming Ingestion with Apache Flink



## Part IV: Ecosystem Integration

#### Query Engines: Trino and Presto



#### Data Governance and Catalog Integration



#### Iceberg in the Modern Data Stack



## Part V: Production Operations

#### Monitoring and Maintenance



#### Scaling Iceberg to Petabyte Scale



#### Security Best Practices