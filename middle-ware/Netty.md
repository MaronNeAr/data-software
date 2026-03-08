#### 核心架构：Reactor模型

###### 单 Reactor 单线程模型 (The Basic Model)

- **工作流程**：Reactor 通过 `select/epoll` 监听事件，收到事件后分发（Dispatch）给对应的 Handler 执行
- **局限性**：无法利用多核 CPU 。如果某个业务逻辑（如复杂的 SQL 翻译 ）耗时较长，整个系统的 I/O 都会被阻塞
- **适用场景**：Redis 的核心执行逻辑（非 I/O 部分）即类似此模型

###### 单 Reactor 多线程模型 (Worker Thread Pool)

- **工作流程**：

  > Reactor 只负责监听连接和读取数据 （Accept、Read）
  >
  > 读取到数据后，Reactor 将业务逻辑（如逻辑计算、数据脱敏 ）分发给 **Worker 线程池**
  >
  > Worker 线程处理完后，将结果返回给 Reactor，由 Reactor 负责将数据写回 Socket（Read）

- **优势**：释放了 Reactor 线程，使其能专注于高频的 I/O 事件
- **痛点**：在高并发场景下，单个 Reactor 线程在处理海量连接的编码/解码时仍可能成为瓶颈

###### 主从 Reactor 多线程模型 (Main-Sub Reactor)

- **MainReactor (BossGroup)**：专门负责处理 `Accept` 事件，建立新连接。一旦连接建立，将其注册到 SubReactor 
- **SubReactor (WorkerGroup)**：负责处理已连接 Socket 的所有 `Read/Write` 事件及数据编解码

- **优点**：

  - **职责分明**：父线程池（Boss）只管接客，子线程池（Worker）只管服务，互不干扰 

  - **可伸缩性**：SubReactor 的数量通常设置为 CPU 核心数的 2 倍，极大提升了吞吐量

###### HPC-ADX实战

- **Netty (SubReactor)**：负责接收来自广告主的毫秒级竞价请求 ，完成协议解析

- **无锁流水线**：解析后的请求不直接在 Worker 线程处理，而是投入 **Disruptor 环形队列** 

- **解耦计算**：利用 Disruptor 的无锁特性，在后端进行亚秒级的流式计费与风控拦截 ，从而确保了全链路 P99 时延稳定在 **100ms** 以内 

- **Interview**

  - Q：面试官可能会问“为什么不直接在 Netty 的 Worker 线程做业务？

  - A：为了防止复杂的业务逻辑占用 I/O 线程，导致新的网络请求无法及时被响应，通过这种架构实现了**计算与 I/O 的彻底分离** 

#### 内存管理：零拷贝与堆外内存

###### 零拷贝 (Zero-Copy)

- **操作系统层面的零拷贝**

  - **mmap (Memory Mapping)**：将内核缓冲区与用户缓冲区映射到同一块物理地址，省去了数据在内核态与用户态之间的拷贝，但仍需 CPU 参与将数据拷贝到 Socket 缓冲区

  - **sendfile**：Linux 提供的系统调用，直接在内核中将数据从文件描述符传输到 Socket 描述符，完全不经过用户态

- **Netty 层面的零拷贝实现**

  - **CompositeByteBuf**：将多个 `ByteBuf` 组合成一个逻辑上的 Buffer，例如将 HTTP Header 和 Body 组合时，无需创建新数组进行 `System.arraycopy()`

  - **Unpooled.wrappedBuffer**：将已有的 byte 数组包装成 `ByteBuf` 对象，而不是拷贝一份新数据

  - **FileRegion**：直接封装了 Java NIO 的 `FileChannel.transferTo()` 方法，在发送文件时直接触发操作系统的 `sendfile` 硬件级零拷贝

###### 堆外内存

- **减轻 GC 压力**：堆外内存直接受操作系统管理，不占用 JVM 堆空间，因此不会触发频繁的 Full GC。

- **提升 I/O 效率**：在进行 Socket 读写时，堆内内存（Heap）的数据必须先拷贝到堆外临时空间（Direct Memory）才能交给内核。直接使用堆外内存可以省去这一次“内存到内存”的拷贝。

- **Netty 的内存分配器：PooledByteBufAllocator**

  - **内存池化**：预先申请大块内存（Chunk），然后切分成小的 Page 和 Subpage。这种机制能极大地降低高并发下创建缓冲区对内存分配器的冲击

  - **引用计数 (Reference Counting)**：这是你必须掌握的重点

    > `ByteBuf` 实现了 `ReferenceCounted` 接口
    >
    > 初始计数为 1，调用 `retain()` 加 1，调用 `release()` 减 1
    >
    > **当计数归 0 时**，Netty 会将该内存块回收到对象池中。如果不手动 release，会导致严重的堆外内存泄漏

###### HPC-ADX实战

- **对象池化**：你封装的 `OffHeapBuffer` 结合对象池技术，确保了在海量竞价包涌入时，系统不需要频繁向操作系统申请内存 

- **避免 GC 暂停**：由于竞价请求生命周期短、数量极大，如果放在堆内，Young GC 会非常频繁。通过 **堆外内存**，你成功将 JVM 内存水位保持在平稳状态，确保了 **P99 时延稳定在 100ms 以内**

- **Interview**

  - Q：如果你的堆外内存发生了泄漏，你该如何排查？

  - A：

    > **工具层面**：利用 Netty 提供的 `-Dio.netty.leakDetection.level=PARANOID`（最高级别采样）开启内存泄漏检测
    >
    > **代码层面**：检查 `SimpleChannelInboundHandler` 是否已经自动释放，或者在自定义 Handler 中是否忘记在 `finally` 块中调用 `ReferenceCountUtil.release(msg)`
    >
    > **系统层面**：查看 `NMT` (Native Memory Tracking) 或使用 `jstat` 观察非堆区的变化

#### 数据流水线：Pipeline 与 Handler

###### 核心概念：流水线与处理器

- **ChannelPipeline**：每一个连接（Channel）都有一个专属的 Pipeline ，它是一个拦截过滤器的列表，决定了数据如何被处理
- **ChannelHandler**：流水线上的工人，每个 Handler 负责特定的任务（如：解码、业务逻辑计算、数据脱敏、编码）

###### 数据流向：入站与出站

- **入站 (Inbound) 处理**

  - **处理方向**：从链表头部（Head）流向尾部（Tail）

  - **解码 (Decoder)**：将字节流转换为 POJO 对象
  - **业务逻辑**：执行你提到的指标一致性检查或 SQL 翻译 

  - **数据填充**：利用你提到的 AOP 组件进行缓存预热 

- **出站 (Outbound) 处理**

  - **处理方向**：从链表尾部（Tail）流向头部（Head）

  - **编码 (Encoder)**：将对象转换为字节流
  - **流控**：结合你提到的 Redisson 令牌桶模式进行多租户限流 

###### 核心机制：Context 与 线程模型

- **ChannelHandlerContext**：这是 Handler 与 Pipeline 交互的桥梁，通过 Context，你可以将处理结果传递给下一个 Handler（`fireChannelRead`）或者跳过后续步骤
- **线程安全保证**：Netty 保证同一个 Channel 的 Pipeline 处理始终由同一个 **SubReactor (EventLoop)** 线程执行
- **优势**：这意味着你在编写 Handler 逻辑时（如更新指标统计），通常**无需加锁**，从而保证了系统的高性能 

###### HPC-ADX实战

- **Decoder 层**：接收来自商业化或直播团队的请求，将原始字节解析为查询 Domain 对象 
- **Auth Handler**：利用你提到的 **RBAC 鉴权体系**，在流水线早期拦截非法请求，保障安全合规 
- **Cache Handler**：检查 Redis 或泰山 KV 缓存。如果命中，直接构建响应并触发 **Outbound** 流程返回，不再请求底层引擎
- **Routing Handler**：根据查询类型（实时/离线）决定路由到 ClickHouse 还是 Iceberg 
- **Encoder 层**：将查询结果序列化，通过 **OffHeapBuffer** 异步写回，实现秒级响应
- **Interview**
  - Q：如果在某个 InboundHandler 中没有调用 `fireChannelRead` 会发生什么？
  - A：数据流会在此中断，后续的 Handler 将收不到任何消息，这在你实现 **限流（Rate Limiting）** 或 **黑名单过滤** 时非常有用——一旦触发限流，直接在当前 Handler 返回错误响应并停止下传

#### Netty Component

- **ServerBootstrap**：引导类，作用是把分散的网络组件（线程组、通道类型、处理器、配置参数）组装起来，并最终让服务器启动
- **EventLoopGroup定义**
  - **Event（事件）**：比如连接就绪、数据到达、缓冲区可写等
  - **Loop（循环）**：一个永远不停的 `while(true)` 循环，在里面不断询问：“有没有事要做？”
  - **Group（组）**：它不是一个线程，而是一组线程的集合

- **EventLoopGroup执行机制**
  - **轮询（Select）**：询问操作系统内核，我管辖的这些连接里，谁有新数据到了？
  - **处理 IO（Process IO）**：执行具体的读写操作，触发你写的 `AdxHandler`
  - **处理任务（Process Task）**：如果你想定时执行任务，或者从别的线程给它派活，它也会在循环里顺便干掉

- **SimpleChannelInboundHandler**
  - **自动释放内存**：它会在业务逻辑执行完后自动释放 `ByteBuf` 资源，防止内存泄漏，省去了手动调用的麻烦
  - **自带类型过滤**：通过泛型指定处理类型，只有匹配的消息才会进入 `channelRead0` 方法，不匹配的会自动转发给下一个 Handler
  - **仅限同步使用**：它只适合**即读即用**的场景；如果你要将消息交给异步线程处理，它会自动提前释放资源导致报错，这种情况下必须改用原生的 `ChannelInboundHandlerAdapter`
