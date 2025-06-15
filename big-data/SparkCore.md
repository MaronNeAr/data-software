#### SparkCore

###### SparkRDD

- RDD：弹性分布式数据集，是Spark中最基本的数据抽象
- RDD在代码中是一个抽象类，代表一个不可变、可分区、元素可并行计算的集合
- 数据模型：一种对现实世界数据特征进行抽象、描述和组织的工具
- 数据结构：采用特殊的结构组织和管理数据
- RDD分布式计算模型
  - 一定是一个对象
  - 一定封装了大量方法和属性（计算逻辑）
  - 一定需要适合进行分布式处理（减小数据规模，并行计算）
- 分布式集群中心化基础架构：主从架构（Master-Slave）
- RDD模型可以封装数据的处理逻辑，需要通过大量的RDD对象组合在一起实现复杂的功能
- RDD的功能是分布式操作，功能调用但不会马上执行
- RDD的处理方式和Java IO流完全一样，也采用装饰者设计模式来实现功能的

###### RDD编程

- 从集合中创建RDD、从外部存储创建RDD、从其他RDD创建

- 创建Spark运行环境及RDD数据处理模型

  ```java
  // 创建Spark配置对象
  final SparkConf conf = new SparkConf();
  conf.setMaster("local");
  conf.setAppName("spark");
  // 构建Spark运行时环境
  final JavaSparkContext jsc = new JavaSparkContext();
  // 构建RDD数据处理模型
  final List<String> names = List.of("zhangsan", "lisi", "wangwu");
  final JavaRDD<String> rdd = jsc.parallelize(names); 
  // parallelize(names, N)表示将数据分为3个切片
  final List<String> collect = rdd.collect();
  collect.foreach(System.out.println);
  // 关闭Spark运行时环境
  jsc.close();
  ```
  
- RDD类似于数据管道，可以流转数据，但是不能保存数据

- Kafka可以将数据进行切片（减小规模），也称之为分区，分区操作是底层完成的

- RDD进行数据分区

  ```Java
  conf.setMaster("local[2]");
  // local[*]表示分区数为当前环境CPU的核心个数
  rdd.saveAsTextFile("output");
  // 将数据保存两个分区当中
  ```

- Spark分区数据存储基本原理：平均分

- Spark内存数据分区规则：$partition = [(i * length) / slices, ((i + 1) * length)/slices]$，类似于AVL

- Spark磁盘数据分区规则

  - 分区数据尝试计算时尽可能平均，按字节来计算
  - 分区数据存储考虑业务数据的完整性，按照行来读取，考虑数据偏移量，同一偏移量不重复读取
