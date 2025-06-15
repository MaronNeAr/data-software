## 第一章 Java内存区域与内存溢出异常

#### 内存区域划分

###### 程序计数器（Program Counter Register）

- 记录当前线程执行的字节码指令地址（分支、循环、跳转等逻辑控制）
- 线程私有，生命周期与线程绑定
- 唯一不会发生内存溢出的区域

###### Java虚拟机栈（Java Virtual Machine Stacks）

- 存储方法调用的栈帧（Stack Frame），每个方法调用对应一个栈帧，包含局部变量表、操作数栈、动态链接等
  - 动态链接：符号引用com.example.MyClass#myMethod转为直接引用0x7f3e8c，虚方法调用和接口方法调用时触发
  - 虚方法通过extends实现，接口方法通过implements实现
  - 虚方法表（vtable）：子类虚方法完全复制父类的vtable，再追加自己的新方法；重写父类方法时，覆盖父类方法在vtable中位置
  - 接口方法表（itable）：支持多接口，动态链接
- 线程私有，每个线程独立分配栈内存
- 通过`-Xss`参数设置栈大小（例如`-Xss1m`）
- 内存溢出场景：
  - StackOverflowError：栈深度超过限制（如无限递归调用）。（-Xss 虚拟机栈大小，通常为1MB）
  - OutOfMemoryError：线程过多导致栈内存耗尽（常见于大量线程创建）（-Xms）

###### 本地方法栈（Native Method Stack）

- 为JVM调用本地（Native）方法（如C/C++代码）服务
- 线程私有，与虚拟机栈类似
- HotSpot将虚拟机栈与本地方法栈合并实现
- 溢出异常：与虚拟机栈相同（StackOverflowError、OutOfMemoryError）

###### Java堆（Java Heap）

- 存放对象实例和数组，是垃圾回收（GC）的主要区域

- 线程共享，几乎所有对象在此分配

- 通过`-Xms`（初始堆大小）、`-Xmx`（最大堆大小）参数控制

- 进一步划分为新生代（Eden、Survivor区）和老年代

- 内存溢出场景（OutOfMemoryError: Java heap space）：对象数量超过堆容量且无法被GC回收（如内存泄漏）

  ```java
  // 堆溢出示例（无限创建对象）
  public class HeapOOMDemo {
      public static void main(String[] args) {
          List<Object> list = new ArrayList<>();
          while (true) {
              list.add(new Object()); // 无限添加对象导致堆溢出
          }
      }
  }
  ```

###### 方法区（Method Area）

- 存储类信息、常量、静态变量、即时编译器编译后的代码等，线程共享

- JDK 8之前：称为“永久代（PermGen）”，通过`-XX:PermSize`、`-XX:MaxPermSize`配置

- JDK 8及之后：改为“元空间（Metaspace）”，使用本地内存，通过`-XX:MetaspaceSize`、`-XX:MaxMetaspaceSize`配置

- 内存溢出场景：

  - OutOfMemoryError: PermGen space（JDK 8前）：加载过多类（如动态生成类、反射滥用）
  - OutOfMemoryError: Metaspace（JDK 8后）：元空间内存不足

  ```java
  // 方法区溢出示例（动态生成类）
  public class MetaspaceOOMDemo {
      static class OOMObject {}
      public static void main(String[] args) {
          int i = 0;
          try {
              while (true) {
                  // 使用CGLIB动态生成类
                  Enhancer enhancer = new Enhancer();
                  enhancer.setSuperclass(OOMObject.class);
                  enhancer.setUseCache(false);
                  enhancer.setCallback((MethodInterceptor) (obj, method, args1, proxy) -> proxy.invokeSuper(obj, args1));
                  enhancer.create(); // 无限生成代理类导致元空间溢出
                  i++;
              }
          } catch (Throwable e) {
              System.out.println("动态生成类次数: " + i);
              throw e;
          }
      }
  }
  ```

###### 运行时常量池（Runtime Constant Pool）

- 存储类文件中的常量池表（如字面量、符号引用）
  - 属于方法区的一部分（JDK 8后属于元空间）
- 内存溢出场景：与方法区溢出类似（如大量字符串常量）

###### 直接内存（Direct Memory）

- 通过`DirectByteBuffer`或NIO的`allocateDirect`分配的堆外内存，避免Java堆与Native堆间数据复制

- 不受JVM堆大小限制，但受物理内存限制

- 通过`-XX:MaxDirectMemorySize`设置最大直接内存

- 内存溢出场景（OutOfMemoryError: Direct buffer memory）：频繁分配堆外内存未释放

  ```java
  // 直接内存溢出示例
  public class DirectMemoryOOMDemo {
      public static void main(String[] args) {
          List<ByteBuffer> list = new ArrayList<>();
          while (true) {
              // 分配直接内存（默认不受-Xmx限制）
              ByteBuffer buffer = ByteBuffer.allocateDirect(1024 * 1024 * 100); // 100MB
              list.add(buffer);
          }
      }
  }
  ```

#### 对象创建与访问

###### 对象创建过程

- 类加载检查：检查类是否已被加载、解析和初始化；若未加载，则执行类加载过程（加载、验证、准备、解析、初始化）
- 内存分配：利用指针碰撞（适用于内存规整）或者空闲列表（适用于内存不规整）进行内存分配
  - 并发问题：利用TLAB进行为每个线程预先分配一小块内存，TLAB不足时使用CAS机制分配内存
- 初始化0值：int为0，boolean为false
- 设置对象头：Mark Word（哈希码、GC分代年龄、锁状态） + Klass Pointer（元数据） + 数组长度（仅限数组）
  - 元数据：JVM 在运行时用于描述类信息的数据结构，它存储了类的类型、方法、字段、继承关系、注解等详细信息

###### 对象的内存布局

- 对象头：Mark Word（8B） + Klass Pointer（4B/8B） + 数组长度（4B）
- 实例数据：对象的实例字段，父类字段在前，子类字段在后
- 对齐填充：确保对象的大小是8字节的整数倍（内存对齐），提高CPU访存效率

###### 对象的访问定位

- 句柄访问：在堆中划分一块句柄池，存储对象的实例数据指针和类型数据指针，栈中的引用指向句柄池中的句柄
- 直接指针访问：栈中的引用直接指向堆中的对象实例数据，对象头中的Klass Pointer指向方法区中的类元数据
- Hotspot的实现：直接指针访问（性能优先）

###### 对象创建与访问的优化技术

- 逃逸分析（Escape Analysis）：分析对象的作用域是否仅限于方法内部（不逃逸、方法逃逸、线程逃逸）
  - 栈上分配：若对象未逃逸，直接在栈上分配（减少GC压力）
  - 标量替换：将对象拆分为基本类型字段，分配在栈上
- 锁消除 （Lock Elision）：若对象未逃逸且未共享，消除不必要的同步锁
- TLAB（Thread Local Allocation Buffer）：为每个线程预先分配一小块内存，避免多线程竞争

#### 内存溢出实战

###### 堆内存溢出（Java Heap Space）

- 内存泄漏：对象被无意长期引用（如静态集合类缓存数据未清理）
- 大对象分配：一次性加载超大文件或数据集到内存（如大数组、缓存数据）
- 配置不足：堆内存（`-Xmx`）设置过小，无法支撑应用正常运行
- 诊断步骤：生成堆转储文件`jmap -dump:format=b,file=heapdump.hprof <pid>`
- 解决方案：代码修复、参数调优、监控告警

###### 栈溢出（StackOverflowError）

- 无限递归：递归调用未设置终止条件
- 线程过多：高并发场景下创建大量线程（每个线程占用独立栈空间）
- 诊断步骤：查看线程栈`jstack <pid> > thread_dump.txt`
- 解决方案：修复代码逻辑、调整栈大小、控制线程数量

###### 方法区/元空间溢出（Metaspace）

- 动态生成类：频繁使用反射（`Class.forName()`）、CGLIB或ASM生成代理类
- 大量加载类：应用依赖过多第三方库或未关闭的类加载器
- 诊断步骤：监控元空间的使用`jstat -gcmetacapacity <pid> 1000`
- 解决方案：增大元空间、启动类缓存（CGLIB的`setUseCache(true)`）、减少动态类的生成

###### 直接内存溢出（Direct Buffer Memory）

- NIO操作：频繁分配堆外内存（如`ByteBuffer.allocateDirect()`）未释放
- JNI调用：本地代码（如C/C++）未正确释放内存
- 诊断步骤：监控直接内存`jcmd <pid> VM.native_memory`
- 解决方案：限制直接内存大小、显式触发GC（`System.gc()`）、手动释放内存（`buffer.cleaner().clean()`）

###### 通用诊断工具与流程

| **工具** | **用途**                                   |
| :------- | :----------------------------------------- |
| `jmap`   | 生成堆转储文件（Heap Dump）                |
| `jstack` | 获取线程栈快照，分析死锁或栈溢出           |
| `jstat`  | 监控GC、类加载、编译统计信息               |
| VisualVM | 图形化监控内存、线程、CPU使用率            |
| MAT      | 分析堆转储，定位内存泄漏根源               |
| Arthas   | 在线诊断工具，支持动态查看类加载、方法调用 |

## 第二章 垃圾收集器与内存分配策略

#### 对象存活判定

###### 引用计数法（Reference Counting）

- 每个对象维护一个引用计数器，记录有多少引用指向该对象。
- 当引用计数器为0时，表示对象不再被使用，可以被回收。
- 无法解决循环引用问题（如上述示例），需要额外的空间存储引用计数

###### 可达性分析算法（Reachability Analysis）

- 通过一系列称为**GC Roots**的根对象作为起点，从这些根对象开始向下搜索，形成引用链
- 如果一个对象不在任何引用链上（即不可达），则判定为可回收对象
- GC Roots的类型：
  - 虚拟机栈中的局部变量：当前正在执行的方法中的局部变量引用的对象
  - 方法区中的静态变量：类的静态字段引用的对象
  - 方法区中的常量：字符串常量池中的引用
  - 本地方法栈中的JNI引用：Native方法引用的对象
  - Java虚拟机内部的引用：如基本数据类型对应的Class对象、系统类加载器
  - 同步锁持有的对象：被`synchronized`锁持有的对象
- 解决了循环引用问题，适用于复杂的对象引用关系，但需要遍历所有对象，性能开销较大

###### 引用类型与对象回收

- 强引用（Strong Reference）：最常见的引用类型，只要强引用存在，对象就不会被回收
  - `Object obj = new Object();`
- 软引用（Soft Reference）：在内存不足时，软引用对象会被回收（适合缓存场景）
  - `SoftReference<Object> softRef = new SoftReference<>(new Object());`
- 弱引用（Weak Reference）：无论内存是否充足，弱引用对象都会被回收（适合临时缓存）
  - `WeakReference<Object> weakRef = new WeakReference<>(new Object());`
- 虚引用（Phantom Reference）：虚引用对象无法通过`get()`获取，主要用于跟踪对象被回收的状态
  - `PhantomReference<Object> phantomRef = new PhantomReference<>(new Object(), new ReferenceQueue<>());`

###### 对象存活的特殊情况

- `finalize()`方法：对象在被回收前，可以重写`finalize()`方法进行资源释放
- `finalize()`执行时机不确定，可能导致对象复活（重新被引用），性能开销大，不建议使用

#### 垃圾收集算法

###### 标记-清除算法（Mark-Sweep）

- 标记阶段：从GC Roots开始遍历，标记所有可达对象
- 清除阶段：遍历整个堆，回收未被标记的对象
- 优点：实现简单，适用于老年代（对象存活率高）
- 缺点：内存碎片化严重，可能导致大对象无法分配
- 适用场景：CMS（Concurrent Mark-Sweep）收集器的老年代回收

###### 复制算法（Copying）

- 将堆内存（Survivor）分为两块（From空间和To空间）
- 复制阶段：将From空间中的存活对象复制到To空间
- 清理阶段：清空From空间，交换From和To空间的角色
- 优点：无内存碎片化问题，效率高（只需复制存活对象）
- 缺点：内存利用率低，不适合对象存活率高的场景（如老年代）
- 适用场景：新生代的垃圾收集（如Serial、ParNew收集器）

###### 标记-整理算法（Mark-Compact）

- 标记阶段：与标记-清除算法相同，标记所有可达对象
- 整理阶段：将存活对象向一端移动，清理边界外的内存
- 优点：无内存碎片化问题，适合对象存活率高的场景（如老年代）
- 缺点：效率较低（需要移动对象）
- 适用场景：Serial Old、Parallel Old收集器的老年代回收

###### 分代收集算法（Generational Collection）

- 根据对象生命周期将堆内存划分为**新生代**和**老年代**
- 新生代：对象生命周期短，采用复制算法
- 老年代：对象生命周期长，采用标记-清除或标记-整理算法
- Eden区：新对象首先分配在Eden区
- Survivor区：分为From和To空间，用于存放经过一次GC后存活的对象
- 晋升机制：对象在Survivor区经历多次GC后，晋升到老年代
- 优点：针对不同代采用不同算法，效率高，减少Full GC的频率
- 适用场景：现代JVM的主流垃圾收集器（如G1、CMS、ZGC）

###### 增量收集算法（Incremental Collecting）

- 将GC过程分为多个小步骤，与用户线程交替执行

###### 分区收集算法（Region-Based Collecting）

- 将堆内存划分为多个独立区域（Region），每个区域独立回收
- 适用场景：G1收集器

| **算法**  | **优点**               | **缺点**           | **适用场景**             |
| :-------- | :--------------------- | :----------------- | :----------------------- |
| 标记-清除 | 实现简单               | 内存碎片化，效率低 | 老年代（CMS）            |
| 复制      | 无碎片化，效率高       | 内存利用率低       | 新生代（Serial、ParNew） |
| 标记-整理 | 无碎片化，适合高存活率 | 效率较低           | 老年代（Serial Old）     |
| 分代收集  | 针对不同代优化，效率高 | 实现复杂           | 现代JVM（G1、CMS）       |

#### 经典垃圾收集器

###### 垃圾收集器的核心目标

- 低延迟（Low Pause Time）：减少用户线程的停顿时间（STW）
- 高吞吐量（High Throughput）：最大化应用运行时间占比
- 内存效率（Memory Efficiency）：减少内存碎片，提升内存利用率

###### 经典垃圾收集器分类

| **收集器**            | **工作模式** | **适用区域**  | **算法**          | **特点**                      |
| :-------------------- | :----------- | :------------ | :---------------- | :---------------------------- |
| **Serial**            | 串行         | 新生代/老年代 | 复制/标记-整理    | 单线程、简单高效、STW长       |
| **ParNew**            | 并行         | 新生代        | 复制              | 多线程版Serial，配合CMS使用   |
| **Parallel Scavenge** | 并行         | 新生代        | 复制              | 吞吐量优先                    |
| **Serial Old**        | 串行         | 老年代        | 标记-整理         | Serial的老年代搭档            |
| **Parallel Old**      | 并行         | 老年代        | 标记-整理         | Parallel Scavenge的老年代搭档 |
| **CMS**               | 并发         | 老年代        | 标记-清除         | 低延迟、内存碎片化            |
| **G1**                | 并发+分区    | 全堆          | 标记-整理+复制    | 平衡吞吐量与延迟              |
| **ZGC**               | 并发         | 全堆          | 染色指针+读屏障   | 超低延迟（<10ms）             |
| **Shenandoah**        | 并发         | 全堆          | Brooks指针+读屏障 | 低延迟、与堆大小无关          |

###### STW（Stop-The-World）

- 暂停所有线程：在 STW 期间，除了垃圾回收线程，所有应用线程都会被暂停
- 全局一致性：确保垃圾回收器在回收内存时，应用线程不会修改对象引用或创建新对象
- 持续时间：STW 的时间长短取决于垃圾回收器的类型和堆内存的大小
- 触发场景：Yong GC有概率触发STW，Full GC一定会触发STW

###### Serial收集器

- 新生代：采用**复制算法**，单线程处理垃圾回收，触发Minor GC时暂停所有用户线程（STW）
- 老年代：Serial Old收集器使用**标记-整理算法**
- 适用场景：单核CPU环境（如客户端应用、嵌入式系统），内存较小（几百MB）且对停顿时间不敏感的场景
- 参数配置：`-XX:+UseSerialGC  # 显式启用Serial收集器`

###### ParNew收集器

- 新生代并行版Serial，多线程执行复制算法，需配合CMS使用
- 默认线程数与CPU核数相同，可通过`-XX:ParallelGCThreads`调整
- 适用场景：多核CPU且需低延迟的老年代回收（与CMS搭配）
- 参数配置：`-XX:+UseParNewGC  # 启用ParNew收集器（需与CMS配合）`

###### Parallel Scavenge/Old收集器

- 新生代（Parallel Scavenge）：多线程复制算法，吞吐量优先（`-XX:GCTimeRatio`控制GC时间占比）

- 老年代（Parallel Old）：多线程标记-整理算法。

- 适用场景：后台计算任务（如批处理），允许较长STW以换取高吞吐量

- 参数配置

  ```bash
  -XX:+UseParallelGC      # 启用Parallel Scavenge
  -XX:+UseParallelOldGC   # 启用Parallel Old
  -XX:MaxGCPauseMillis=200  # 目标最大停顿时间（不保证）
  ```

###### CMS（Concurrent Mark-Sweep）收集器

- 初始标记（Initial Mark）：标记GC Roots直接关联对象（STW）

- 并发标记（Concurrent Mark）：遍历对象图，标记可达对象（与用户线程并发）

- 重新标记（Remark）：修正并发标记期间的变化（STW）

- 并发清除（Concurrent Sweep）：回收垃圾对象（与用户线程并发）

- 内存碎片问题：标记-清除算法可能导致碎片，需定期Full GC（或开启`-XX:+UseCMSCompactAtFullCollection`压缩内存）

- 适用场景：对延迟敏感的应用（如Web服务），老年代回收时允许部分并发

- 参数配置

  ```bash
  -XX:+UseConcMarkSweepGC       # 启用CMS
  -XX:CMSInitiatingOccupancyFraction=70  # 老年代使用率70%时触发CMS
  -XX:+CMSParallelRemarkEnabled # 并行重新标记以缩短STW
  ```

###### G1（Garbage-First）收集器

- 内存分区：将堆划分为多个等大的Region（1MB~32MB），每个Region可属于Eden、Survivor、Old或Humongous（大对象区）

- Young GC：回收Eden和Survivor区

- Mixed GC：回收部分Old Region（基于回收价值预测）

- SATB（Snapshot-At-The-Beginning）：通过快照保证并发标记的正确性

- 适用场景：大内存（6GB以上）、多核CPU，需平衡吞吐量与延迟（如大数据平台）

- 参数配置

  ```bash
  -XX:+UseG1GC                 # 启用G1
  -XX:MaxGCPauseMillis=200     # 目标最大停顿时间
  -XX:G1HeapRegionSize=16m     # 设置Region大小
  ```

###### ZGC（Z Garbage Collector）

- 染色指针（Colored Pointers）：在指针中嵌入元数据（如标记、重映射状态），减少内存开销

- 读屏障（Load Barrier）：在访问对象时触发屏障，处理并发标记和转移

- 并发转移：将存活对象移动到新Region，无需长时间STW

- 适用场景：超大堆（TB级）、要求亚毫秒级停顿（如实时交易系统）

- 参数配置

  ```bash
  -XX:+UseZGC                  # 启用ZGC（JDK 15+默认支持）
  -XX:+ZGenerational           # 启用分代ZGC（JDK 21+）
  ```

###### Shenandoah收集器

- Brooks指针：每个对象额外存储一个转发指针（Forwarding Pointer），用于并发转移

- 并发回收：标记、转移、重映射阶段均与用户线程并发

- 适用场景：与ZGC类似，但实现更简单，适用于OpenJDK社区版本

- 参数配置

  ```bash
  -XX:+UseShenandoahGC         # 启用Shenandoah
  -XX:ShenandoahGCMode=iu      # 设置并发模式（iu：增量更新）
  ```

###### ZGC 减少 STW 停顿的关键技术

1. **并发标记与清理**：ZGC 的标记和清理过程大部分是并发执行的，不会阻塞应用程序的执行。
2. **染色指针**：使得垃圾回收线程与应用程序线程可以在不进行同步的情况下访问对象，避免了锁的争用。
3. **区域化堆**：堆被分为多个区域，能够在多个区域上并行回收，避免了全局回收带来的长时间停顿。
4. **分阶段回收**：将 GC 过程拆分为多个小阶段，并将大部分阶段并发执行，只有少数几个阶段会触发短暂的 STW。
5. **延迟回收**：将某些清理工作推迟到后台执行，进一步降低了应用程序停顿的时间。

###### 垃圾收集器的选择策略

| **应用场景**                | **推荐收集器**    | **核心优势**             |
| :-------------------------- | :---------------- | :----------------------- |
| 单核/小内存客户端应用       | Serial            | 简单、无多线程开销       |
| 高吞吐量后台计算            | Parallel Scavenge | 最大化CPU利用率          |
| Web服务（低延迟老年代回收） | CMS               | 并发标记清除，减少STW    |
| 大内存多核应用（平衡型）    | G1                | 分区回收，可控停顿时间   |
| 超大内存、超低延迟          | ZGC/Shenandoah    | 亚毫秒级停顿，TB级堆支持 |

###### 调优实战建议

- 使用`jstat -gcutil <pid>`观察各代内存使用和GC时间
- 通过`-Xlog:gc*`输出详细GC日志，用工具（如GCViewer）分析
- 新生代大小：避免过小导致频繁Minor GC，或过大延长单次GC时间
- 晋升阈值：调整`-XX:MaxTenuringThreshold`控制对象晋升老年代的速度
- 使用`jmap -histo:live <pid>`查看对象分布，定位未释放的大对象

#### 内存分配策略

###### 内存分配的核心策略

- JVM的内存分配策略基于**分代收集理论**，将堆内存划分为新生代和老年代，不同区域采用不同的分配规则
- 对象优先在Eden区分配：大多数新对象在新生代的Eden区分配，Eden区内存不足时，触发**Minor GC**，回收新生代垃圾
- 大对象直接在老年代分配：大对象（如长字符串、大数组）直接在**老年代**分配，避免在Eden区和Survivor区之间反复复制
- 长期存活的对象进入老年代
  - 对象在新生代经历多次GC后仍存活，会被晋升到老年代
  - 年龄计数器：对象每存活一次Minor GC，年龄加1
  - 年龄阈值：通过`-XX:MaxTenuringThreshold`设置晋升阈值（默认值15）
- 动态对象年龄判断：若Survivor区中相同年龄对象的总大小超过Survivor区容量的一半，则年龄≥该值的对象直接晋升老年代
- 空间分配担保
  - 在Minor GC前，JVM检查老年代剩余空间是否大于新生代对象总大小
  - 若不足，则检查是否允许担保失败（`-XX:-HandlePromotionFailure`），若允许则继续Minor GC，否则触发Full GC
- 调优参数
  - `-XX:NewRatio=2`：老年代与新生代的内存比例为2:1。
  - `-XX:SurvivorRatio=8`：Eden区与Survivor区的内存比例为8:1:1（Eden:From:To ）
  - `-XX:PretenureSizeThreshold=4M` ：对象超过4MB直接进入老年代

###### 内存分配策略的调优实战

| **问题**           | **原因**                       | **解决方案**                            |
| :----------------- | :----------------------------- | :-------------------------------------- |
| **频繁Full GC**    | 老年代空间不足，对象晋升过快   | 增大老年代（`-Xmx`），调整晋升阈值      |
| **Eden区GC频繁**   | 短生命周期对象过多，Eden区过小 | 增大新生代（`-XX:NewRatio`）            |
| **大对象分配失败** | 老年代碎片化，无法容纳大对象   | 使用G1收集器整理内存，或增大堆空间      |
| **直接内存溢出**   | NIO操作未释放堆外内存          | 显式调用`System.gc()`或限制直接内存大小 |

```bash
# 堆内存与新生代配置
-Xms4g -Xmx4g                # 初始堆和最大堆设为4GB
-XX:NewRatio=3               # 老年代:新生代=3:1（新生代1GB）
-XX:SurvivorRatio=8          # Eden:Survivor=8:1:1（Eden=800MB）

# 晋升与分配优化
-XX:MaxTenuringThreshold=10  # 对象年龄阈值设为10
-XX:PretenureSizeThreshold=4M # 大对象阈值4MB
-XX:+UseTLAB                 # 启用TLAB（默认开启）
```

## 第三章 虚拟机性能监控与故障处理工具

#### 命令行工具

###### jps：Java进程状态工具

- 列出当前系统中所有的Java进程及其主类名、JVM参数等信息

- 常用命令

  ```bash
  jps -l  # 显示主类的完整包名
  jps -m  # 显示传递给main方法的参数
  jps -v  # 显示JVM启动参数
  ```

###### jstat：JVM统计监控工具

- 监控JVM的类加载、内存、GC等统计信息

- 常用命令

  ```bash
  jstat -gc <pid> 1000 10  # 每1秒输出一次GC信息，共10次
  jstat -class <pid>       # 显示类加载信息
  jstat -complier <pid>		 # 显示类编译信息
  jstat -gcutil <pid>      # 显示GC统计信息的百分比
  ```

- 输出字段解析（以`-gc`为例）

  | **字段** | **含义**                   | 字段 | 含义                         |
  | :------- | :------------------------- | ---- | ---------------------------- |
  | S0C      | Survivor 0区当前容量（KB） | S0U  | Survivor 0区已使用容量（KB） |
  | S1C      | Survivor 1区当前容量（KB） | S1U  | Survivor 1区已使用容量（KB） |
  | EC       | Eden区当前容量（KB）       | EU   | Eden区已使用容量（KB）       |
  | OC       | 老年代当前容量（KB）       | OU   | 老年代已使用容量（KB）       |
  | MC       | 元空间当前容量（KB）       | MU   | 元空间已使用容量（KB）       |
  | YGC      | Young GC次数               | YGCT | Young GC总时间（秒）         |
  | FGC      | Full GC次数                | FGCT | Full GC总时间（秒）          |

###### jinfo：JVM配置信息工具

- 查看和修改JVM的运行参数

- 常用命令

  ```bash
  jinfo <pid>                  # 查看JVM参数和系统属性
  jinfo -flag <name> <pid>     # 查看某个JVM参数的值
  jinfo -flag [+|-]<name> <pid> # 启用或禁用某个JVM参数
  ```

###### jmap：内存分析工具

- 生成堆转储文件（Heap Dump），分析内存使用情况

- 常用命令

  ```bash
  jmap -heap <pid>            # 显示堆内存使用情况
  jmap -histo <pid>           # 显示堆中对象的直方图
  jmap -dump:format=b,file=heapdump.hprof <pid>  # 生成堆转储文件

###### jstack：线程分析工具

- 生成线程快照（Thread Dump），分析线程状态

- 常用命令

  ```bash
  jstack <pid>                # 生成线程快照
  jstack -l <pid>             # 生成线程快照并显示锁信息
  ```

###### jcmd：多功能命令行工具

- 集成多种功能，包括生成堆转储、线程快照、查看JVM参数等

- 常用命令

  ```bash
  jcmd <pid> help              # 查看支持的命令
  jcmd <pid> GC.class_stats    # 显示类统计信息
  jcmd <pid> GC.heap_dump filename=heapdump.hprof  # 生成堆转储文件
  jcmd <pid> Thread.print      # 生成线程快照
  ```

###### 命令行工具使用场景

| **工具** | **主要功能**      | **适用场景**                     |
| :------- | :---------------- | :------------------------------- |
| jps      | 查看Java进程      | 快速定位Java进程ID               |
| jstat    | 监控JVM统计信息   | 实时监控GC、类加载、内存使用情况 |
| jinfo    | 查看和修改JVM参数 | 动态调整JVM配置                  |
| jmap     | 生成堆转储文件    | 分析内存泄漏、对象分布           |
| jstack   | 生成线程快照      | 定位死锁、线程阻塞问题           |
| jcmd     | 多功能工具        | 集成多种功能，简化操作           |

#### JVM调优参数

###### JVM内存管理参数

| **参数**                  | **作用**                                 | **示例值**                   |
| :------------------------ | :--------------------------------------- | :--------------------------- |
| `-Xms`                    | 初始堆大小（Heap 初始值）                | `-Xms512m`                   |
| `-Xmx`                    | 最大堆大小（Heap 最大值）                | `-Xmx4g`                     |
| `-Xmn`                    | 年轻代（Young Generation）大小           | `-Xmn1g`                     |
| `-XX:MetaspaceSize`       | 元空间（Metaspace）初始大小              | `-XX:MetaspaceSize=256m`     |
| `-XX:MaxMetaspaceSize`    | 元空间最大大小                           | `-XX:MaxMetaspaceSize=512m`  |
| `-Xss`                    | 每个线程的栈大小（影响线程数和递归深度） | `-Xss1m`                     |
| `-XX:MaxDirectMemorySize` | 直接内存（NIO Direct Buffer）的最大大小  | `-XX:MaxDirectMemorySize=1g` |

###### JVM垃圾回收参数

| **参数**                  | **作用**                                     |
| :------------------------ | :------------------------------------------- |
| `-XX:+UseG1GC`            | 启用 G1 垃圾回收器（默认 JDK 9+）            |
| `-XX:+UseParallelGC`      | 启用并行垃圾回收器（吞吐量优先）             |
| `-XX:+UseConcMarkSweepGC` | 启用 CMS 垃圾回收器（低延迟，JDK 9+ 已废弃） |
| `-XX:+UseZGC`             | 启用 ZGC（低延迟，JDK 15+ 生产可用）         |
| `-XX:+PrintGCDetails`     | 打印详细的 GC 日志                           |
| `-XX:+PrintGCDateStamps`  | 在 GC 日志中添加时间戳                       |

###### G1 专用参数

| **参数**               | **作用**                                              |
| :--------------------- | :---------------------------------------------------- |
| `-XX:MaxGCPauseMillis` | 目标最大 GC 停顿时间（如 `-XX:MaxGCPauseMillis=200`） |
| `-XX:G1NewSizePercent` | 年轻代占堆的最小比例（默认 `5%`）                     |
| `-XX:G1HeapRegionSize` | G1 堆区域大小（如 `-XX:G1HeapRegionSize=16m`）        |

###### ZGC 专用参数

| **参数**                        | **作用**                      |
| :------------------------------ | :---------------------------- |
| `-XX:+ZGenerational`            | 启用分代式 ZGC（JDK 21+）     |
| `-XX:ZAllocationSpikeTolerance` | 控制 ZGC 的内存分配尖峰容忍度 |

###### 调试与监控参数

| **参数**                           | **作用**                                                     |
| :--------------------------------- | :----------------------------------------------------------- |
| -`XX:+HeapDumpOnOutOfMemoryError`  | 在内存溢出时生成堆转储文件（`-XX:HeapDumpPath=/path/to/dump.hprof`） |
| `-XX:ErrorFile=/path/to/error.log` | 指定 JVM 崩溃日志文件路径                                    |
| `-XX:+PrintFlagsFinal`             | 打印所有 JVM 参数的最终值（验证参数是否生效）                |
| `-agentlib:jdwp=...`               | 启用远程调试                                                 |

## 第四章 类文件结构

#### Class文件格式

###### Class文件结构

- 魔数与版本号：标识文件类型和版本
- 常量池：存储字面量和符号引用
- 访问标志：类的访问权限和属性（如`public`、`final`）
- 类索引、父类索引与接口索引：类的继承关系
- 字段表：类的字段信息
- 方法表：类的方法信息
- 属性表：附加信息（如代码、行号、注解）

###### 魔数与版本号

- 魔数：固定值`0xCAFEBABE`，标识这是一个Class文件
- 版本号：次版本号（低16位）+ 主版本号（高16位）

###### 常量池（Constant Pool）

- 作用：存储字面量（如字符串、数值）和符号引用（如类名、方法名）

- 结构：常量池是一个表，每一项是一个`cp_info`结构（如下）；常量池索引从1开始（0表示无效索引）

- 常见常量类型

  | **常量类型**         | **`tag` 值** | **`info` 结构**                           | **用途**                         |
  | :------------------- | :----------- | :---------------------------------------- | :------------------------------- |
  | `CONSTANT_Class`     | 7            | `u2 name_index`                           | 表示类或接口的符号引用           |
  | `CONSTANT_Fieldref`  | 9            | `u2 class_index + u2 name_and_type_index` | 表示字段的符号引用               |
  | `CONSTANT_Methodref` | 10           | `u2 class_index + u2 name_and_type_index` | 表示方法的符号引用（非接口方法） |
  | `CONSTANT_String`    | 8            | `u2 string_index`                         | 表示字符串字面量                 |
  | `CONSTANT_Integer`   | 3            | `u4 bytes`                                | 存储 4 字节的整数值（如 `int`）  |
  | `CONSTANT_Utf8`      | 1            | `u2 length + u1 bytes[length]`            | 存储 UTF-8 编码的字符串          |

###### 访问标志（Access Flags）

- 描述类或接口的访问权限和属性

- 常见访问标志

  | **标志**      | **值** | **描述**                        |
  | :------------ | :----- | :------------------------------ |
  | ACC_PUBLIC    | 0x0001 | `public`类                      |
  | ACC_FINAL     | 0x0010 | `final`类                       |
  | ACC_SUPER     | 0x0020 | 使用`invokespecial`指令的新语义 |
  | ACC_INTERFACE | 0x0200 | 接口                            |
  | ACC_ABSTRACT  | 0x0400 | 抽象类                          |
  | ACC_SYNTHETIC | 0x1000 | 编译器生成的类                  |

###### 类索引、父类索引与接口索引

- 类索引（this_class）：指向常量池中该类名的符号引用
- 父类索引（super_class）：指向常量池中父类名的符号引用（`java.lang.Object`的父类索引为0）
- 接口索引（interfaces）：指向常量池中接口名的符号引用数组

###### 字段表（Fields）

- 描述类的字段（成员变量）

- 每个字段是一个`field_info`结构，包含字段名、描述符、访问标志等

- 字段描述符

  | **类型** | **描述符**           |
  | :------- | :------------------- |
  | `int`    | `I`                  |
  | `long`   | `J`                  |
  | `String` | `Ljava/lang/String;` |
  | `int[]`  | `[I`                 |

###### 方法表（Methods）

- 每个方法是一个`method_info`结构，包含方法名、描述符、访问标志、代码属性等

- 方法描述符

  | **字段名**         | **数据类型**       | **说明**                                                     |
  | :----------------- | :----------------- | :----------------------------------------------------------- |
  | `access_flags`     | `u2`（2字节）      | 方法的访问修饰符和属性（如 `public`、`static`、`synchronized` 等）。 |
  | `name_index`       | `u2`               | 指向常量池（Constant Pool）中 `CONSTANT_Utf8` 类型的索引，表示方法名。 |
  | `descriptor_index` | `u2`               | 指向常量池中 `CONSTANT_Utf8` 类型的索引，表示方法描述符（参数和返回类型）。 |
  | `attributes_count` | `u2`               | 方法属性的数量。                                             |
  | `attributes`       | `attribute_info[]` | 方法的属性表，每个属性描述方法的额外信息（如字节码、异常表等）。 |

###### 属性表（Attributes）

- 存储类、字段、方法的附加信息

- 常见属性

  | **属性**           | **描述**                 |
  | :----------------- | :----------------------- |
  | Code               | 方法的字节码指令         |
  | LineNumberTable    | 源代码行号与字节码的映射 |
  | LocalVariableTable | 局部变量表               |
  | SourceFile         | 源文件名                 |
  | Exceptions         | 方法抛出的异常           |

###### Class文件解析工具

- 反编译Class文件，查看字节码和常量池

  ```bash
  javap -v HelloWorld.class // 反编译Class文件，查看字节码和常量池
  ```

#### 字节码指令集

###### 字节码指令的分类

| **类别**               | **功能**                                     | **典型指令**                       |
| :--------------------- | :------------------------------------------- | :--------------------------------- |
| **加载与存储指令**     | 将数据从局部变量表加载到操作数栈，或反向存储 | `iload`, `istore`, `aload`         |
| **算术指令**           | 执行基本数学运算（加减乘除等）               | `iadd`, `isub`, `imul`             |
| **类型转换指令**       | 在不同数据类型之间转换                       | `i2l`, `d2i`, `f2d`                |
| **对象操作指令**       | 创建对象、访问字段、调用方法                 | `new`, `getfield`, `invokevirtual` |
| **操作数栈管理指令**   | 直接操作操作数栈                             | `pop`, `dup`, `swap`               |
| **控制转移指令**       | 实现条件分支、循环和跳转                     | `ifeq`, `goto`, `tableswitch`      |
| **方法调用与返回指令** | 调用方法及返回结果                           | `invokestatic`, `ireturn`          |
| **异常处理指令**       | 抛出异常和处理异常                           | `athrow`, `jsr`（已废弃）          |
| **同步指令**           | 实现`synchronized`同步块                     | `monitorenter`, `monitorexit`      |

###### 加载与存储指令

- 在局部变量表（Local Variable Table）和操作数栈（Operand Stack）之间传输数据

- 常见指令

  > `iload_0`：将局部变量表索引0的`int`值压入栈
  >
  > `aload_1`：将局部变量表索引1的对象引用压入栈
  >
  > `istore_2`：将栈顶`int`值存储到局部变量表索引2

###### 算术指令

- 执行基本数学运算，不同数据类型有对应指令

- 常见指令

  > `iadd`：栈顶两个`int`值相加，结果压栈
  >
  > `dsub`：栈顶两个`double`值相减，结果压栈
  >
  > `imul`：栈顶两个`int`值相乘

###### 类型转换指令

- 将一种数据类型转换为另一种（如`int`转`long`）

- 常见指令

  > `i2l`：将`int`转换为`long`
  >
  > `d2i`：将`double`转换为`int`（可能丢失精度）

###### 对象操作指令

- 对象的创建、字段访问和方法调用

- 常见指令

  > `new`：创建对象（未初始化，需调用构造方法）
  >
  > `getfield`：获取对象字段值
  >
  > `invokevirtual`：调用实例方法（动态分派）

###### 控制转移指令

- 实现条件分支（`if`）、循环（`for`）和跳转

- 常见指令

  > `ifeq`：若栈顶`int`值为0，则跳转
  >
  > `goto`：无条件跳转
  >
  > `tableswitch`：用于`switch`语句（连续case值）

###### 性能监控与故障排查

- 性能瓶颈定位：某方法执行缓慢
  - 使用`javap`查看方法字节码，分析指令数量（如循环内的冗余操作）
  - 结合`jstat`监控GC频率，判断是否因对象频繁创建导致GC压力
- 内存泄漏分析：堆内存持续增长
  - 通过`jmap`生成堆转储，用MAT分析大对象
  - 查看相关类的字节码，检查是否有未释放的静态集合操作（如`putstatic`指令）
- 线程死锁诊断：应用无响应
  - 使用`jstack`获取线程快照，查找`BLOCKED`状态的线程
  - 分析线程持有的锁（`monitorenter`和`monitorexit`指令是否匹配）

## 第五章 类加载机制

#### 类加载过程

###### 加载（Loading）

- 将类的字节码文件（`.class`文件）加载到内存，并生成对应的`Class`对象（存储在方法区）

- 查找字节码：通过类的全限定名（如`java.lang.String`）定位字节码文件
- 读取字节码：将字节码转换为二进制流
- 生成Class对象：在方法区中创建类的`Class`对象（后续反射操作的基础）
  - `Class<?> clazz = Class.forName("com.example.MyClass");  // 触发加载阶段`

###### 验证（Verification）

- 确保字节码符合JVM规范，防止恶意代码或错误字节码危害JVM安全
- 文件格式验证：检查魔数（`0xCAFEBABE`）和版本号是否合法，验证常量池中的常量类型是否有效
- 元数据验证：检查类是否有父类（除`Object`外）、是否继承final类、字段/方法是否与父类冲突
  - 若一个类尝试继承`final`类（如`String`），验证阶段会抛出`java.lang.VerifyError`
- 字节码验证：确保方法体的字节码不会导致JVM崩溃（如操作数栈溢出、跳转到不存在的指令）
- 符号引用验证：检查符号引用（如类名、方法名）是否合法，确保解析阶段能正确绑定

###### 准备（Preparation）

- 为类的静态变量（类变量）分配内存，并设置初始值（零值）

###### 解析（Resolution）

- 将常量池中的**符号引用**转换为**直接引用**（内存地址或偏移量）
- 符号引用：一组符号描述目标（如`java/lang/Object`）
- 直接引用：指向目标的指针、偏移量或句柄
- 类/接口解析：将类名符号引用转换为对应的`Class`对象
- 字段解析：将字段名转换为内存中的偏移量
- 方法解析：将方法名转换为方法入口地址
- 解析阶段可能在初始化之后触发（如动态绑定）

###### 初始化（Initialization）

- 执行类的初始化代码（`<clinit>()`方法），为静态变量赋真实值，执行静态代码块
- 触发条件
  - 创建类的实例（`new`）
  - 访问类的静态变量（非常量）或静态方法
  - 反射调用（如`Class.forName()`）
  - 初始化子类时，父类需先初始化
  - JVM启动时指定的主类（包含`main()`的类）
- 初始化顺序：父类的`<clinit>()`先于子类执行，静态变量赋值和静态代码块按代码顺序执行
- 若多个线程同时初始化一个类，JVM会保证同步（仅一个线程执行`<clinit>()`）

###### 类加载总结

| **阶段** | **输入**         | **输出**                 | **核心操作**                               |
| :------- | :--------------- | :----------------------- | :----------------------------------------- |
| 加载     | `.class`文件     | `Class`对象              | 查找字节码，生成内存结构                   |
| 验证     | 字节码           | 合法字节码               | 检查格式、元数据、字节码逻辑               |
| 准备     | 静态变量符号引用 | 静态变量内存分配（零值） | 分配内存，设置默认值                       |
| 解析     | 符号引用         | 直接引用                 | 绑定类、字段、方法的内存地址               |
| 初始化   | 静态变量和代码块 | 类完全可用               | 执行`<clinit>()`，赋真实值，执行静态代码块 |

###### 常见问题与解决方案

- 类加载失败：
  - 原因：类路径错误、字节码损坏、版本不兼容
  - 解决：检查`-classpath`配置，确认类文件完整性

- 静态代码块死锁：

  - 原因：多线程初始化时，静态代码块内同步操作导致死锁

  - 解决：避免在静态代码块中使用复杂同步逻辑

- 类重复加载：
  - 原因：不同类加载器加载同一类
  - 解决：遵循双亲委派模型，避免自定义类加载器破坏机制

#### 类加载器

###### 类加载器的核心作用

- 加载字节码：从文件系统、网络、JAR包等来源读取字节码

- 类隔离：通过不同类加载器实现类的命名空间隔离（如Tomcat中的Web应用）

- 动态加载：支持运行时加载类（如插件化、热部署）

- 安全性控制：防止恶意代码替换核心类（如`java.lang.String`）

###### 启动类加载（Bootstrap ClassLoader）

- 职责：加载JVM核心类库（`jre/lib`目录下的`rt.jar`、`resources.jar`等）
  - Java 9+ 加载`java.base(java.lang/java.util/java.io)`、`java.datatransfer`、`java.instrument`
- 实现：由C/C++编写，是JVM的一部分，无Java类实例
- 访问限制：无法在Java代码中直接引用（`getClassLoader()`返回`null`）

###### 扩展类加载器（Extension ClassLoader）/平台类加载器（Platform ClassLoader, Java 9+）

- 职责：加载扩展类库（`jre/lib/ext`目录下的JAR包）
  - Java 9+ 加载`java.sql`、`java.xml`、`java.logging`、`java.management`
- 实现：Java类`sun.misc.Launcher$ExtClassLoader`
- 父加载器：启动类加载器

######  应用程序类加载器（Application ClassLoader）

- 职责：加载用户类路径（`-classpath`或`CLASSPATH`环境变量）下的类
- 实现：Java类`sun.misc.Launcher$AppClassLoader`
- 父加载器：扩展类加载器
- 默认类加载器：`ClassLoader.getSystemClassLoader()`返回此加载器

###### 双亲委派模型（Parent Delegation Model）

- 委派父加载器：优先让父加载器尝试加载
- 父加载器失败：若父加载器无法加载，自己尝试加载。
- 最终失败：若所有加载器无法加载，抛出`ClassNotFoundException`
- 双亲委派的优势
  - 避免类重复加载：父加载器加载的类，子加载器不会重复加载
  - 保护核心类库：防止用户自定义类覆盖核心类（如自定义`java.lang.Object`

###### 自定义类加载器

- 继承`ClassLoader`：重写`findClass()`方法
- 加载字节码：从自定义路径（如网络、加密文件）读取字节码
- 定义类：调用`defineClass()`生成`Class`对象
- 使用场景
  - 热替换：动态加载修改后的类（如调试环境）
  - 加密类加载：加载加密的字节码文件
  - 模块隔离：不同模块使用独立类加载器（如Tomcat的Web应用）

###### Tomcat的类加载器

- 层级结构：
  - Common ClassLoader：加载Tomcat和Web应用共享的类
  - WebApp ClassLoader：每个Web应用独立，加载`WEB-INF/classes`和`WEB-INF/lib`
  - JSP ClassLoader：动态加载JSP编译后的类，支持热替换
- 隔离机制：不同Web应用的类加载器相互隔离，避免类冲突

###### Spring的动态代理

- 场景：为接口生成代理类。
- 类加载器：使用`AppClassLoader`加载代理类，或通过`Thread.currentThread().getContextClassLoader()`获取

###### OSGi模块化

- 机制：每个Bundle（模块）有自己的类加载器，按需动态加载依赖
- 优势：支持模块热插拔和版本共存

#### 打破双亲委派

###### 线程上下文类加载器（Thread Context ClassLoader）

- 背景：在SPI机制中，核心接口由启动类加载器加载，但实现类需由应用类加载器加载
- 通过`Thread.currentThread().setContextClassLoader()`设置线程上下文类加载器
- 在SPI代码中，使用`Thread.currentThread().getContextClassLoader()`加载实现类

###### 自定义类加载器

- 背景：需要动态加载类或实现模块化隔离
- 继承`ClassLoader`，重写`loadClass()`或`findClass()`方法，直接加载类而不委派父加载器

######  OSGi模块化

- 背景：每个模块（Bundle）需要独立的类加载器，支持模块热插拔和版本共存
- 每个Bundle有自己的类加载器，按需加载依赖模块
- 通过`Import-Package`和`Export-Package`声明模块间的依赖关系

###### 打破双亲委派的应用场景

- SPI机制：JDBC、JNDI、JAXP等服务的实现类需由应用类加载器加载，使用线程上下文类加载器加载实现类
- 热部署：在开发或调试环境中，动态替换已加载的类，自定义类加载器直接加载新版本的类，旧版本类由GC回收
- OSGI模块化与插件化：不同模块或插件需要独立的类加载器，避免类冲突，每个模块或插件使用独立的类加载器

## 第六章 虚拟机字节码执行引擎

#### 运行时栈帧结构

###### 栈帧的组成

| **组件**         | **作用**                                                 |
| :--------------- | :------------------------------------------------------- |
| **局部变量表**   | 存储方法参数和局部变量                                   |
| **操作数栈**     | 执行字节码指令时的工作区，用于存储中间计算结果           |
| **动态链接**     | 指向运行时常量池中该栈帧所属方法的符号引用               |
| **方法返回地址** | 记录方法执行完成后需要返回的位置（调用者的程序计数器值） |
| **附加信息**     | 可选信息（如调试信息、异常处理表）                       |

###### 局部变量表（Local Variable Table）

- 存储**方法参数**和**方法内部定义的局部变量**
- 在编译期间确定容量（不可运行时修改），容量存储在方法的`Code`属性中
- 以**变量槽（Variable Slot）**为最小单位，每个Slot占用32位（`int`、`float`、`引用类型`）
- 64位类型（`long`、`double`）占用两个连续Slot
- 实例方法：第0位Slot存储`this`引用，后续按参数和局部变量顺序分配
- 静态方法：无`this`引用，从第0位开始存储参数

###### 操作数栈（Operand Stack）

- 作为字节码指令的工作区，用于存储计算过程中的中间结果
- 操作数栈的最大深度在编译时确定（存储在方法的`Code`属性中）
- 操作指令实例
  - **`iload_1`**：将局部变量表索引1的`int`值压入栈顶
  - **`iadd`**：弹出栈顶两个`int`值相加，结果压栈
  - **`istore_3`**：将栈顶`int`值存入局部变量表索引3

###### 动态链接（Dynamic Linking）

- 指向运行时常量池中该栈帧所属方法的符号引用
- 在类加载的**解析阶段**，符号引用被转换为直接引用（方法实际入口地址）
- 支持**多态性**：在运行时确定实际调用的方法（如虚方法分派）
- 实现动态绑定：如接口方法调用（`invokeinterface`）和虚方法调用（`invokevirtual`）

###### 方法返回地址（Return Address）

- 存储方法执行完成后需要返回的位置（即调用者的程序计数器值）
- 正常返回：执行`return`指令，恢复调用者的程序计数器
- 异常返回：通过异常处理表确定跳转地址
- 返回方式：
  - **`ireturn`**：返回`int`
  - **`areturn`**：返回对象引用
  - **`return`**：返回`void`

###### 附加信息

- 调试信息：如行号表（`LineNumberTable`），用于定位字节码对应的源码行号
- 异常处理表：记录`try-catch`块的字节码范围和处理逻辑
- 栈帧数据：部分JVM实现可能扩展附加信息，用于支持本地方法或特殊优化

###### 栈帧的分配与回收

- 分配方式：
  - 静态分配：编译时确定栈帧大小（局部变量表和操作数栈的容量）
  - 动态扩展：某些JVM允许栈帧动态扩展（但可能抛出`StackOverflowError`）
- 内存分配：
  - 栈帧内存分配在**虚拟机栈**（Java方法）或**本地方法栈**（Native方法）
  - 每个线程有独立的栈空间，栈帧随方法调用入栈，方法结束出栈
- 栈溢出：
  - 原因：栈深度超过虚拟机栈的最大容量（如无限递归）
  - 错误类型：`StackOverflowError`（线程请求的栈深度超过限制）

#### 方法调用

###### 方法调用的字节码指令

| **指令**          | **作用**                               | **适用场景**                         |
| :---------------- | :------------------------------------- | :----------------------------------- |
| `invokestatic`    | 调用静态方法                           | 类静态方法（无`this`引用）           |
| `invokespecial`   | 调用构造方法、私有方法、父类方法       | `<init>`、`private`方法、`super`调用 |
| `invokevirtual`   | 调用虚方法（根据对象实际类型动态分派） | 实例方法（非`final`、非`private`）   |
| `invokeinterface` | 调用接口方法                           | 接口方法实现                         |
| `invokedynamic`   | 动态解析调用点（支持动态类型语言）     | Lambda表达式、Groovy等动态语言       |

###### 解析（Resolution）

- 目标：将符号引用（常量池中的方法引用）转换为直接引用（方法入口地址）
- 触发条件：在类加载的解析阶段完成（适用于静态方法、私有方法等**非虚方法**）

###### 分派（Dispatch）

- 目标：根据对象的实际类型确定调用的方法版本（适用于虚方法和接口方法）
- 静态分派（Static Dispatch）：根据静态类型选择方法（方法重载）
- 动态分派（Dynamic Dispatch）：根据实际类型选择方法（方法重写）

###### 静态分派（方法重载）

- 依据：方法的**静态类型**（声明类型）决定调用的方法版本
- 编译时确定：在编译阶段即可确定目标方法

###### 动态分派（方法重写）

- 依据：对象的**实际类型**（运行时类型）决定调用的方法版本

- 运行时确定：通过虚方法表（vtable）实现动态分派

- 虚方法表（Virtual Method Table, vtable）

  - 结构：每个类维护一个虚方法表，存储该类所有虚方法的实际入口地址
  - 继承关系：子类的虚方法表复制父类的表项，重写的方法替换为子类方法地址

- 接口方法表（Interface Method Table, itable）

  - 结构：接口方法的动态分派通过itable实现，存储接口方法的具体实现地址

  - 复杂度：由于类可实现多个接口，itable的查找比vtable更复杂

###### 动态类型语言支持（invokedynamic）

- 动态类型语言：类型检查在运行时进行（如JavaScript、Python）
- Java的限制：传统字节码指令（如`invokevirtual`）无法高效支持动态类型
- invokedynamic指令
  - 引导方法（Bootstrap Method）：在首次调用时动态解析方法
  - CallSite：绑定具体的调用目标（如方法句柄MethodHandle）

###### 方法调用的性能优化

- 内联缓存（Inline Caching）：缓存最近调用的方法地址，减少查表开销
  - 单态内联（Monomorphic）：仅缓存一个目标方法
  - 多态内联（Polymorphic）：缓存有限数量的目标方法
  - 超多态（Megamorphic）：退化为查表
- JIT编译器优化
  - 方法内联：将热点方法直接内联到调用处，避免方法调用开销
  - 去虚化（Devirtualization）：在可确定实际类型时，将虚方法调用转为静态调用

## 第七章 程序编译与代码优化

#### 即时编译器（JIT）

###### JIT基本作用

- **目标**：弥补 Java 解释执行的性能短板，通过动态编译将高频执行的字节码转换为高效的本地代码
- **核心思想**：**“二进制翻译” + “运行时优化”**，即在程序运行的过程中，根据实际执行情况针对性地优化代码

###### JIT的工作流程

- 代码加载与解析
  - **字节码生成**：JVM 将 `.class` 文件加载到内存并解析为字节码流
  - **解释执行**：初始阶段由解释器逐条执行字节码（避免编译开销）
- 热点代码探测
  - **热点代码（Hot Code）**：程序中频繁执行的方法或循环（如循环体、高频调用的函数）
  - **计数器**：统计方法调用的次数（如 HotSpot 中的 `MethodCounter`）
  - **回边计数**：统计分支跳转的命中率（如循环的预测是否准确）
  - **触发条件**：当某段代码达到预设的阈值（如执行次数超过 1000 次），会被标记为“热点”
- 代码编译与优化
  - **即时编译**：将热点字节码转换为本地机器码（如 x86、ARM 指令）
  - **优化技术**：在编译阶段进行多种性能优化（见下文）

###### JIT编译器的类型

###### C1 编译器（Client Compiler）

- **适用场景**：启动阶段的短时间编译，快速执行。
- 注重编译速度，生成代码质量稍低。
- 适合交互式应用（如 IDE、调试环境）。

###### C2 编译器（Server Compiler）

- **适用场景**：程序稳定运行后的长期优化。
- 注重编译质量，生成高效的本地代码。
- 支持复杂的优化技术（如逃逸分析、内联）。
- 适合服务器端长生命周期的应用。

#### 代码优化技术

###### 方法内联（Method Inlining）

- **目的**：减少方法调用的开销。
- **实现**：将频繁调用的短方法的字节码直接嵌入到调用者的代码中

###### 逃逸分析（Escape Analysis）

- **目的**：确定对象的作用域是否被限制在当前方法内
- **栈上分配**：如果对象不会逃逸到方法外，可直接在栈上分配内存，避免堆分配和垃圾回收（GC）
- **标量替换**：将对象拆解为基本数据类型，消除不必要的装箱/拆箱操作

###### 分支预测与消除（Branch Prediction & Elimination）

- **分支预测**：通过历史执行数据预测条件分支的走向（如 CPU 的硬件预测机制）
- **分支消除**：对于无法预测的分支（如 `if-else` 中无规律的条件），通过编译技术减少跳转开销

###### 循环优化（Loop Optimization）

- **循环展开**：将循环体重复多次执行，减少循环控制的开销
- **尾递归优化**：将递归调用转换为迭代（需语言和编译器支持，Java 不原生支持尾递归优化）

###### 类型推测（Type Inference）

- **目的**：在泛型代码中，根据实际传入类型推测泛型参数的具体类型

###### 死代码消除（Dead Code Elimination）

- 移除无实际用处的代码（如未使用的变量、永远不会执行的分支）

## 第八章 Java内存模型与线程

#### Java内存模型（JMM）

###### JMM 的核心概念

**主内存与工作内存**：

- **主内存**（Main Memory）是所有线程共享的内存区域，存放着所有变量的值
- 每个线程都有自己的 **工作内存**（Working Memory），它是该线程的私有内存区域。线程操作共享变量时，先从主内存将变量拷贝到工作内存中，然后对工作内存中的变量进行修改，最后再将修改结果写回主内存

**共享变量**：

- JMM共享变量是多个线程可以访问的变量。通常是 `static` 变量或者实例变量。局部变量是线程私有的，不受 JMM 的影响

**内存屏障（Memory Barriers）**：

- 内存屏障是指 CPU 或者编译器用来保证操作顺序的一种机制。它通过禁止指令重排，确保某些操作在执行时的顺序

**变量的可见性、原子性与有序性**：

- **可见性**：当一个线程修改了共享变量的值，其他线程能够看到这个修改
- **原子性**：对共享变量的操作要么完全成功，要么完全失败，不会中断。对于一些基本的操作（如 `i++`）来说，JMM 并不保证其原子性，需要通过同步手段来确保
- **有序性**：JMM 保证每个线程内的代码执行顺序，但不一定保证所有线程之间的执行顺序。为了确保线程之间操作的顺序，JMM 提供了**同步机制**来控制

###### JMM 中的关键规则

**线程间的可见性保证**：

- **可见性问题**的核心是，当一个线程修改了共享变量，其他线程如何及时看到这个修改。JMM 的设计通过**内存同步**（比如锁机制、`volatile` 关键字、`synchronized` 关键字等）来确保可见性
- **`volatile` 关键字**：声明为 `volatile` 的变量会直接从主内存中读取，而不是从线程的工作内存中读取。写入 `volatile` 变量时，JMM 会保证该写操作对其他线程可见。`volatile` 确保了**可见性**，但不能保证**原子性**和**有序性**

**原子性保障**：

- 在 JMM 中，只有一些基本的操作（如读取和写入一个 `long` 或 `double` 类型的变量）是原子的。对于复合操作（如 `i++`），如果不加同步，可能会出现原子性问题
- **原子性问题的解决方法**：使用 `synchronized`、`ReentrantLock` 等同步机制来确保原子性

**有序性保障**：

- JMM 规定了每个线程内的指令执行顺序，但在不同线程之间，JMM 不保证执行顺序。为了控制执行顺序，可以使用 **`synchronized`**、**`volatile`** 或 **`Lock`** 等手段
- **`synchronized` 关键字**：`synchronized` 用于确保代码块的互斥执行，并在释放锁时会**刷新工作内存中的值到主内存**，从而保证线程间的可见性和顺序性
- **`volatile` 关键字**：保证了变量的写操作立即刷新到主内存，且对该变量的读操作总是直接从主内存读取，避免了线程之间的数据不一致性

###### JMM 的实现和底层原理

- **MESI 协议**（Modified, Exclusive, Shared, Invalid），用于多核 CPU 之间缓存数据的一致性
- **内存屏障（Memory Barrier）**：用于禁止指令重排，确保特定操作的顺序执行

#### volatile语义

###### 可见性（Visibility）

- **保证**：当一个线程修改了 `volatile` 变量的值，新值会立即被刷新到主内存；其他线程在读取该变量时，会从主内存中重新加载最新值
- **实现机制**：通过插入 `Memory Barrier`（内存屏障）或缓存一致性协议（如 MESI 协议）强制同步主内存和工作内存的数据

###### 原子性（Atomicity）

- **单变量操作**：对 `volatile` 变量的读写操作是原子性的（例如 `count++` 不会被拆分为 `read+increment+write`）
- **复合操作**：`volatile` 不能保证复合操作的原子性（例如 `i++` 或 `a = b + c`），仍需借助 `synchronized` 或 `AtomicInteger` 等类

###### 禁止指令重排序（Ordering）

- **编译器优化**：编译器和处理器可能会对指令进行重排序以提高性能
- **读操作**：在读取 `volatile` 变量前插入 `Load Barrier`，禁止之前的读/写操作被重排到其后
- **写操作**：在写入 `volatile` 变量后插入 `Store Barrier`，禁止之后的读/写操作被重排到其前
- **效果**：保证 `volatile` 变量的读写顺序符合程序逻辑

#### happens-before规则

###### 程序顺序规则

- **同一线程内**，代码执行顺序与书写顺序一致（编译器和处理器可能重排指令，但需保证单线程结果不变）

###### 监视器锁规则

- **Lock → Unlock**：对同一锁的 `synchronized` 块，`Lock` 操作必在 `Unlock` 前发生
- **Unlock → Lock**：`Unlock` 后，其他线程的 `Lock` 操作才能获取该锁

###### volatile 变量规则

- **写 → 读**：对 `volatile` 变量的写操作，必在后续读操作之前完成
- **读 → 写**：对 `volatile` 变量的读操作，必在后续写操作之前完成

###### 线程启动规则

- `Thread.start()` 必须在新建线程的任何操作之前发生

###### 线程终止规则

- 线程的 `run()` 方法结束（正常或异常退出）必在 `join()` 返回之前发生

###### 中断规则

- 对线程的 `interrupt()` 调用必在该线程检测到中断状态（如 `isInterrupted()`）之前发生

###### 对象终结规则

- 对象的 `finalize()` 方法执行完必在字段被垃圾回收之前发生（注：`finalize()` 已废弃）

###### 传递性规则

- 若 A → B 且 B → C，则 A → C（可通过多条规则推导复杂顺序约束）

## 第九章 线程安全与锁优化

#### 线程安全级别

|        **级别**         |                 **描述**                 |                   **示例**                   |
| :---------------------: | :--------------------------------------: | :------------------------------------------: |
| **不可变（Immutable）** |      对象状态不可变，天然线程安全。      |             `String`、`Integer`              |
|    **绝对线程安全**     |  所有操作都线程安全（Java 中极少见）。   | `Vector`（通过同步实现，但复合操作仍不安全） |
|    **相对线程安全**     |    单次操作线程安全，复合操作需同步。    |       `Collections.synchronizedList()`       |
|      **线程兼容**       |        需调用方通过同步保证安全。        |                 `ArrayList`                  |
|      **线程对立**       | 无论是否同步都无法保证安全（设计错误）。 |             未正确同步的共享变量             |

#### 线程安全的实现方式

###### 互斥同步（Mutual Exclusion）

- **核心机制**：通过锁（如 `synchronized`、`ReentrantLock`）保证临界区代码的原子性

- **底层原理**：

  - **synchronized**：基于对象头中的 **Mark Word** 和 **Monitor 锁**实现

    ```c
    // HotSpot 虚拟机对象头（64位）
    Mark Word (64 bits): [锁状态标志 | 线程ID | 分代年龄 | ...]
    ```

  - **锁状态**：无锁 → 偏向锁 → 轻量级锁 → 重量级锁

  - **ReentrantLock**：基于 **AQS（AbstractQueuedSynchronizer）** 实现，支持公平/非公平锁

###### 非阻塞同步（Non-Blocking）

- **核心机制**：通过 **CAS（Compare-And-Swap）** 实现无锁编程
- **CAS 指令**：`AtomicInteger`、`AtomicReference` 等类的底层实现
- **ABA 问题**：通过 `AtomicStampedReference` 解决
- **硬件支持**：x86 架构的 `cmpxchg` 指令

#### 锁优化策略

###### 偏向锁（Biased Locking）

- **目的**：减少无竞争场景下的同步开销
- **原理**：记录首个获取锁的线程 ID，后续该线程无需 CAS 操作即可直接进入临界区
- **适用场景**：单线程重复访问锁
- **参数**：`-XX:+UseBiasedLocking`（默认开启）

###### 轻量级锁（Lightweight Locking）

- **目的**：减少多线程轻度竞争时的锁开销
- **原理**：通过 CAS 操作将对象头替换为线程栈指针，避免操作系统级阻塞
- **适用场景**：低并发竞争（如两个线程交替访问）

###### 自旋锁（Spin Lock）

- **目的**：减少线程阻塞和唤醒的上下文切换开销
- **原理**：线程在竞争锁时循环等待（自旋），而非立即挂起
- **参数**：`-XX:PreBlockSpin=10`（默认自旋 10 次后升级为阻塞）

###### 锁消除（Lock Elimination）

- **目的**：去除无实际竞争场景下的冗余锁
- **原理**：JIT 编译器通过 **逃逸分析**，若发现锁对象仅被当前线程使用，直接删除锁操作

###### 锁粗化（Lock Coarsening）

- **目的**：减少频繁加锁/解锁的开销
- **原理**：合并多个连续的锁操作为一个更大的锁范围

###### 适应性自旋（Adaptive Spinning）

- **目的**：动态调整自旋策略，平衡 CPU 资源消耗
- **原理**：根据历史自旋成功率和锁持有时间，动态调整自旋次数或直接阻塞

###### 重量级锁（Heavyweight Locking）

- **目的**：解决高并发竞争下的线程安全
- **原理**：依赖操作系统互斥量（mutex）和条件变量（condition variable），直接阻塞竞争线程
- **适用场景**：多线程高竞争（如秒杀场景）

