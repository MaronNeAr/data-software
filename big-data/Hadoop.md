#### Hadoop HDFS

###### 分布式存储

1. **分布式存储模型**：
	- HDFS采用分布式存储模型，将大规模数据集划分成多个数据块，并在集群中的多个节点上进行分布式存储。
	- 每个数据块通常被划分为大小固定的块（默认大小为128MB或256MB），并被分布式存储在集群中的多个DataNode上。
2. **数据块的复制**：
	- 为了提高数据的容错性和可靠性，HDFS采用数据块的复制机制。每个数据块默认会被复制到集群中的三个不同的DataNode上。
	- 复制机制允许系统在某个节点发生故障时，从其他副本中获取数据，保证了数据的可用性和容错性。
3. **数据块的移动**：
	- HDFS允许数据块的移动和复制，以便优化数据的存储和访问性能。当数据块的副本数量不足时，NameNode可以根据需要在集群中的其他节点上创建新的副本。
	- 数据块的移动也可以通过重新分配数据块的副本位置来优化数据的访问性能，使得数据块更接近数据访问的节点。
4. **数据块的位置信息**：
	- NameNode负责维护数据块的位置信息，包括每个数据块的副本位置、所在的DataNode节点等。这些信息被用于数据访问时的块定位和数据读取。
5. **数据的并行读写**：
	- 由于数据被分散存储在集群中的多个节点上，HDFS支持并行的数据读写操作。多个客户端可以同时读取和写入数据，以实现高吞吐量的数据访问。
6. **容错性和恢复性**：
	- HDFS具有很高的容错性和恢复性。当某个节点发生故障时，NameNode可以通过复制的数据块进行数据恢复，以确保数据的完整性和可用性。

###### 主存结构

1. **NameNode（主节点）**：
	- NameNode是HDFS的核心组件之一，负责管理文件系统的命名空间、文件和目录结构，以及数据块的元数据信息。
		- 命名空间：文件系统的目录结构
	- 它维护了文件系统的所有元数据信息，包括文件和目录的**名称、权限、时间戳、数据**块的位置等。
	- NameNode不存储实际的文件数据，而是负责维护数据块的**元数据信息**，并管理数据块的**复制、移动和恢复**。
	- NameNode是HDFS的单点故障，因此它的**高可用性和容错性**对系统的稳定性至关重要。
2. **DataNode（数据节点）**：
	- DataNode是HDFS中存储实际数据的节点，负责存储和管理数据块。
	- 每个数据节点在本地存储文件系统上存储一个或多个数据块的副本，并根据NameNode的指令执行数据的读写操作。
	- DataNode周期性地向NameNode报告自身的健康状态和存储容量，并接收来自NameNode的指令，如数据块的复制、移动和删除。
3. **客户端**：
	- 客户端是HDFS的数据访问接口，负责向NameNode查询文件系统的元数据信息，并与DataNode节点进行数据的读写操作。
	- 客户端通过HDFS提供的API或命令行工具与HDFS进行交互，可以进行文件的上传、下载、删除、重命名等操作。

###### 优缺点

**优点**

1.高容错性

> 数据自动保存多个副本。它通过增加副本的形式，提高容错性。
> 某一个副本丢失以后，它可以自动恢复。

2.适合大数据处理

> 数据规模：能够处理数据规模达到 GB、TB、甚至 PB 级别的数据。
> 文件规模：能够处理百万规模以上的文件数量，数量相当之大。

3.流式数据访问，它能保证数据的一致性。
4.可构建在廉价机器上，通过多副本机制，提高可靠性。

**缺点**

1.不适合低延时数据访问，比如毫秒级的存储数据，是做不到的。
2.无法高效的对大量小文件进行存储。

> 存储大量小文件的话，它会占用 NameNode 大量的内存来存储文件、目录和块信息。这样是不可取的，因为 NameNode 的内存总是有限的。
> 小文件存储的寻道时间会超过读取时间，它违反了 HDFS 的设计目标。

3.不支持并发写入、文件随机修改

> 一个文件只能有一个写，不允许多个线程同时写。
> 仅支持数据 append（追加），不支持文件的随机修改。

###### 常用命令

| 命令           | 说明                                         | 举例                                           |
| -------------- | -------------------------------------------- | ---------------------------------------------- |
| -help          | 输出这个命令参数                             | hadoop fs -help rm-ls                          |
| -ls            | 显示目录信息                                 | hadoop fs -ls /                                |
| -mkdir         | 在 HDFS 上创建目录                           | hadoop fs -mkdir -p /usr/input                 |
| -rmdir         | 删除空目录                                   | hadoop fs -rmdir /test                         |
| -rm            | 删除文件或文件夹                             | hadoop fs -rm /usr/input/test.txt              |
| -moveFromLocal | 从本地剪切粘贴到 HDFS                        | hadoop fs -moveFromLocal a.txt /usr/input      |
| -copyFromLocal | 从本地文件系统中拷贝文件到 HDFS 路径去       | hadoop fs -copyFromLocal c.txt /               |
| -copyToLocal   | 从 HDFS 拷贝到本地                           | hadoop fs -copyToLocal /usr/input/a.txt        |
| -appendToFile  | 追加一个文件到已经存在的文件末尾             | hadoop fs -appendToFile b.txt /usr/input/a.txt |
| -cat           | 显示文件内容                                 | hadoop fs -cat /usr/input/a.txt                |
| -cp            | 从 HDFS 的一个路径拷贝到 HDFS 的另一个路径   | hadoop fs -cp /usr/input/a.txt /f.txt          |
| -mv            | 在 HDFS 目录中移动文件                       | hadoop fs -mv /f.txt /usr/input/               |
| -get           | 等同于 copyToLocal                           | hadoop fs -get /usr/input/a.txt                |
| -put           | 等同于 copyFromLocal                         | hadoop fs -put d.txt /usr/input/               |
| -getmerge      | 合并下载多个文件                             | hadoop fs -getmerge /usr/input/* ./tmp.txt     |
| -tail          | 显示一个文件的末尾                           | hadoop fs -tail /usr/input/a.txt               |
| -chmod         | Linux 文件系统中的用法一样，修改文件所属权限 | hadoop fs -chmod 666 /usr/input/a.txt          |
| -setrep        | 设置 HDFS 中文件的副本数量                   | hadoop fs -setrep 10 /usr/input/a.txt          |
| -fu            | 统计文件夹的大小信息                         | hadoop fs -du -s -h /usr/input                 |

#### Hadoop MapReduce

###### MapReduce编程模型

1. **Map阶段**：

	- 在Map阶段，输入数据被划分为多个数据块，并在集群中的多个节点上进行并行处理和转换。

	- 用户编写一个Map函数，接受输入键值对，对每个键值对进行处理，并生成中间结果。Map函数通常会执行一些计算、过滤、转换或提取操作，并产生零个或多个中间键值对。

	- Map函数的输出被分区，并根据键的哈希值分发到集群中的不同节点上的Reduce任务进行处理。

	- ```python
		# 伪代码，用于说明Map阶段的过程
		
		# Map函数：将文本中的每个单词映射为键值对（单词, 1）
		def map_function(line):
		    word_count = {}
		    words = line.split()  # 将行拆分为单词列表
		    for word in words:
		        word_count[word] = 1  # 将单词映射为计数为1的键值对
		    return word_count
		
		# 输入文本数据
		input_data = [
		    "Hello world",
		    "Hello Hadoop",
		    "Hadoop is a framework for big data processing"
		]
		
		# 对每行数据执行Map操作，生成中间键值对
		intermediate_data = []
		for line in input_data:
		    intermediate_data.append(map_function(line))
		
		# 输出中间结果
		for item in intermediate_data:
		    print(item)
		    
		# 中间结果
		{'Hello': 1, 'world': 1}
		{'Hello': 1, 'Hadoop': 1}
		{'Hadoop': 1, 'is': 1, 'a': 1, 'framework': 1, 'for': 1, 'big': 1, 'data': 1, 'processing': 1}
		```

2. **Shuffle和Sort阶段**：

	- 在Map阶段结束后，Map任务的输出被收集和分区，并根据键的哈希值进行排序。
	- 相同键的中间结果（Hello、Hadoop）会被分发到相同的Reduce任务进行处理，以便将具有相同键的中间结果进行合并和汇总。

3. **Reduce阶段**：

	- 在Reduce阶段，每个Reduce任务处理一个或多个Map任务的输出，并对其进行合并和汇总，生成最终的输出结果。

	- 用户编写一个Reduce函数，接受相同键的一组中间值，并对这些值进行聚合计算，生成最终的输出。

	- Reduce函数的输出通常会被写入到文件或其他存储系统中，并作为MapReduce作业的最终输出。

	- > Reduce任务首先接收到具有相同单词的一组中间结果。这些中间结果已经按键分组，并且具有相同单词的中间结果被分配给了同一个Reduce任务。
		>
		> 对于每个单词，Reduce任务迭代处理接收到的所有中间结果，并将它们的值相加以得到最终的计数结果。
		>
		> 例如，Reduce任务接收到了以下中间结果：
		>
		> ```tex
		> {'Hello': 1, 'world': 1}  # 来自Map任务1
		> {'Hello': 1, 'Hadoop': 1}  # 来自Map任务2
		> ```
		>
		> 对于单词'Hello'，合并两个中间结果得到计数结果为2；对于单词'world'和'Hadoop'，由于只有一个中间结果，因此计数结果分别为1。

4. **分布式计算框架**：

	- MapReduce编程模型运行在一个分布式计算框架上，其中包括一个集群管理器（如YARN）和多个计算节点。
	- 集群管理器负责任务的调度、节点的监控和资源管理，以及任务的执行和失败处理。
	- 计算节点上运行着Map任务和Reduce任务，它们通过集群管理器协调和通信，并使用Hadoop分布式文件系统（HDFS）来进行数据的存储和传输。

#### Hadoop Yarn

1. **ResourceManager**（资源管理器）：
	- ResourceManager是YARN的主要组件，负责管理集群中的资源，并为运行在集群上的应用程序分配资源。
	- ResourceManager包含两个核心组件：Scheduler（调度器）和 ApplicationManager（应用程序管理器）。
	- Scheduler负责根据应用程序的资源需求和集群的资源情况进行资源的调度和分配。
	- ApplicationManager负责接收来自客户端的应用程序提交请求，并为每个应用程序启动一个ApplicationMaster。
2. **NodeManager**（节点管理器）：
	- NodeManager是集群中每个节点上的代理，负责管理节点的资源，并监控节点上运行的容器（container）。
	- 它负责与ResourceManager通信，汇报节点的资源使用情况，并接受来自ResourceManager的资源分配请求。
	- NodeManager还负责启动、监控和停止应用程序的容器，并向ResourceManager汇报容器的状态和运行情况。
3. **ApplicationMaster**（应用程序管理器）：
	- 每个运行在集群上的应用程序都有一个对应的ApplicationMaster，负责管理应用程序的执行过程。
	- ApplicationMaster负责向ResourceManager请求资源，监控应用程序的执行情况，并与NodeManager协调容器的启动和管理。
	- 它还负责处理应用程序的失败和重启，以及与客户端进行通信和状态报告。