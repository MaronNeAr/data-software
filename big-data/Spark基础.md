#### Spark基础

###### Spark基础概念

- Spark-分布式计算引擎（框架），基于MR开发的
  - 单机：单进程、单节点处理任务，内存资源利用率较低
  - 伪分布式：多进程、单节点处理任务，可以申请更多的资源，抢占CPU的能力更强
  - 分布式：多进程、多节点处理任务，资源更丰富，内存资源利用率高
- 计算：让计算机执行一些操作，都称之为计算，例如将数据存储到文件中
- Spark支持绝大部分的离线数据分析场景以及部分实时数据分析场景
- RDD分布式计算模型
  - 一定是一个对象
  - 一定封装了大量方法和属性（计算逻辑）
  - 一定需要适合进行分布式处理（减小数据规模，并行计算）
- 分布式计算的核心是切分数据，减小数据规模
- 分布式存储HDFS（切分）、分布式消息传输Kafka（切分）
- 集群中心化：NameNode做集群中心、DataNode进行数据存储、备份节点standby
- 集群去中心化：节点均为DataNode，其中一个当中心节点，另外做备份节点
- Spark分布式集群也采用的是集群中心化，干活的是Executor，中心节点是Driver
- 系统：完整的计算机程序（HDFS、Kafka）；引擎：核心功能
- 框架：不完整的计算机程序（核心功能已经开发完毕，但是和业务相关的代码未开发）
- Spark是基于MR开发的
  - MR：Java，不适合大量的数据处理
  - Spark：Scala，适合大量的数据处理
  - Hadoop出现的早，只考虑单一计算的操作：File -> Mapper、Reducer -> File -> Mapper、Reducer -> File
  - Spark优化了计算过程：File -> Mapper、Reducer -> Memory（内存） -> Mapper、Reducer -> File

###### Spark概述

- Spark是一种基于内存的快速、通用、可扩展的大数据分析计算引擎
  - 主要解决海量数据的存储和海量数据的分析计算
- Hadoop MR框架是从数据源获取数据，经过分析计算后，将结果输出到指定位置，核心是一次计算，不适合迭代计算
- Spark框架支持迭代式计算，图形计算，Spark更快的原因是中间结果不落盘（Spark的shuffle也是落盘的）

###### Spark内置模块

- Spark Core：实现了Spark的基本功能，包含任务调度、内存管理、错误恢复、与存储系统交互模块、RDD的API定义
- SparkSQL结构化数据：Spark用来操作结构化数据的程序包，支持多种数据源
- Spark Streaming实时计算：Spark提供的对实时数据进行流式计算的组件，与Spark Core中的RDD API高度对应
- Spark Mlib机器学习、Spark GraphX图计算

###### Spark运行模式

- 部署：将软件安装到什么位置，部署Spark指的是Spark的程序逻辑在谁提供的资源中执行
- 部署模式：单机模式和集群模式
- 资源调度：Yarn + Spark（Standalone），也称之为Spark On Yarn
- Local模式：在本地部署单个Spark任务
- Standalone模式：Spark自带的任务调度模式
- Yarn模式：Spark使用Hadoop的Yarn组件进行资源与任务调度
- Mesos模式：Spark使用Mesos平台进行资源与任务的调度