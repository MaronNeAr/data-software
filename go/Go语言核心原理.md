## 第一部分 数据结构与类型系统的底层实现
#### 基本类型与内存对齐

###### 基本类型

| **类型**                  | **大小 (字节)** | **对齐要求 (字节)** | **备注**                                  |
| ------------------------- | --------------- | ------------------- | ----------------------------------------- |
| `int8` / `uint8` (byte)   | 1               | 1                   | 最小单位                                  |
| `int16` / `uint16`        | 2               | 2                   |                                           |
| `int32` / `uint32` (rune) | 4               | 4                   |                                           |
| `int64` / `uint64`        | 8               | 8                   |                                           |
| `int` / `uint`            | **8**           | **8**               | **取决于 CPU 字长**，64位系统下同 `int64` |
| `uintptr`                 | 8               | 8                   | 能够存储指针地址的整数                    |
| `float32`                 | 4               | 4                   |                                           |
| `float64`                 | 8               | 8                   |                                           |

###### 内存对齐

- **结构体整体对齐：** 整个结构体的长度必须是其**最大成员对齐值**的倍数

#### Slice（切片）

###### Slice 底层结构

```go
type SliceHeader struct {
    Data uintptr // 指向底层数组的指针
    Len  int     // 当前切片的长度
    Cap  int     // 切片的容量（底层数组从 Data 指针开始算起的最大长度）
}
```

- **Data：** 这是一个指针，指向内存中真正存储数据的连续数组。
- **Len：** 你当前能访问到的元素个数。
- **Cap：** 底层数组的大小。它决定了在不重新分配内存的情况下，切片最大能增长到多少。

###### Slice 内存共享

```go
arr := [5]int{1, 2, 3, 4, 5}
s1 := arr[1:3] // s1 = [2, 3], len=2, cap=4
s2 := s1[0:1]  // s2 = [2], len=1, cap=4
```

- **副作用：** 修改 `s2[0]`，`s1[0]` 和 `arr[1]` 也会随之改变。
- **安全性：** 在大数据量处理（如 `clickhouse-go` 解析数据行）时，通过共享内存可以避免昂贵的内存拷贝，但也需要小心数据污染。

###### Slice 扩容机制

- 如果期望容量（double cap）比旧容量的两倍还大，直接使用期望容量
- 否则，如果旧容量小于 **256**，则新容量直接翻倍（$new = old \times 2$）
- 如果旧容量大于等于 **256**，则使用公式：$new = old + (old + 3 \times 256) / 4$
  - 随着 `old` 增大，扩容比例会从 $2.0$ 逐渐平滑下降到 $1.25$

###### 拷贝与传递

- **切片作为函数参考**：函数内部修改切片元素，外部可见；函数内部对切片进行 `append` 导致扩容，外部**不可见**
- **Copy函数**：`copy(dst, src)` 会将数据从源切片拷贝到目标切片；对较小的 Len 进行拷贝，是深拷贝

#### Map（映射）

###### hmap

- **count**: 当前 map 中的元素个数

- **B**: 桶的个数的对数（桶的个数 = $2^B$）

- **buckets**: 指向桶数组的指针

- **oldbuckets**: 扩容时指向旧桶数组的指针（搬迁完后为 nil）

- **nevacuate**: 搬迁进度计数器

- **extra**: 溢出桶相关信息

###### bmap

- **tophash**: 一个长度为 8 的数组，存储每个 Key 哈希值的高 8 位，用于快速定位
- **keys**: 连续存储 8 个 Key（内存对齐）
- **elems (values)**: 连续存储 8 个 Value
- **overflow**: 指向下一个溢出桶的指针（当 8 个坑位满时使用）

###### 查找过程

- **计算哈希**：通过 Key 计算出一个 64 位的哈希值
- **定位桶**：用哈希值的 **低 B 位** 决定该 Key 落在哪个桶
- **快速筛选**：取哈希值的 **高 8 位**（tophash），在桶内的 `tophash` 数组中线性循环对比
- **精确对比**：如果 `tophash` 匹配，再根据偏移量找到 `key` 的实际位置，进行最终的 `==` 对比
- **返回结果**：匹配成功则返回 Value 的指针

###### 扩容机制：平滑迁移

- **触发条件**

  - **装载因子（Load Factor）过大**：Go 的阈值是 **6.5**（即平均每个桶存了超过 6.5 个元素）。此时触发 **翻倍扩容**

  - **溢出桶过多**：即便数据不多，但频繁删除插入导致溢出桶堆积。此时触发 **等量扩容**（整理碎片）

- **搬迁逻辑**
  - **分配新空间**：`oldbuckets` 指向旧内存，`buckets` 指向新内存（翻倍）
  - **触发搬迁**：每次对 Map 进行 **写入** 或 **删除** 操作时，都会顺便搬迁 1~2 个桶
  - **查询处理**：搬迁期间，查询会先看旧桶，如果旧桶还没搬完且存在，则去旧桶里找

###### 并发不安全

- 如一个 Goroutine 在写，另一个在读/写，程序会直接抛出 runtime panic：`fatal error: concurrent map read and write`
- **底层原理**：`hmap` 中有一个 `flags` 字段，写操作开始前会标记为 `hashWriting`，结束后清除。任何操作前都会检查这个位
- **解决方法**：使用 `sync.Mutex` 加锁，或者使用 `sync.Map`

#### Interface（接口）

###### 接口的两种底层形态

- **eface (Empty Interface)**

  ```go
  type eface struct {
      _type *_type         // 指向数据的类型信息
      data unsafe.Pointer  // 指向具体的数据实例
  }
  ```

- **iface (Non-empty Interface)**

  ```go
  type iface struct {
      tab  *itab           // 包含类型信息和方法表
      data unsafe.Pointer  // 指向具体的数据实例
  }
  ```

###### 核心组件：itab 与 _type

- **_type (元数据)**：它记录了类型的名称、大小、对齐方式、哈希值等
- **itab (接口转换表)**
  - **inter**: 指向接口本身的定义
  - **_type**: 指向具体实现的类型
  - **fun**: 函数指针数组。这里存储了具体类型实现的方法地址

#### Channel（通道）

**核心数据结构：hchan**

```go
type hchan struct {
    qcount   uint           // 当前环形队列中的元素个数
    dataqsiz uint           // 环形队列的大小（缓冲大小）
    buf      unsafe.Pointer // 指向环形队列的指针（仅对有缓冲 channel）
    elemsize uint16         // 元素大小
    closed   uint32         // 标记是否关闭
    elemtype *_type         // 元素类型
    sendx    uint           // 发送索引（环形队列写指针）
    recvx    uint           // 接收索引（环形队列读指针）
    recvq    waitq          // 等待接收的 Goroutine 队列
    sendq    waitq          // 等待发送的 Goroutine 队列
    lock     mutex          // 互斥锁，保证 Channel 操作的原子性
}
```

- **环形队列 (buf)**：有缓冲 Channel 的本质。通过 `sendx` 和 `recvx` 实现 FIFO（先进先出）
- **等待队列 (waitq)**：这是一个双向链表，存储了因为该 Channel 阻塞的 Goroutine（封装在 `sudog` 结构中）
- **互斥锁 (lock)**：Channel 的发送和接收是**线程安全**的。注意，这意味着 Channel 的操作是有锁开销的

###### 发送与接收的底层流程

- **发送过程 (`ch <- v`)**
  - **直接发送**：如果 `recvq`（等待接收队列）里有 Goroutine 在等着，直接把数据拷贝给对方的栈，并唤醒该 Goroutine
  - **存入缓冲区**：如果 `recvq` 为空，但 `buf`（缓冲区）还没满，就把数据拷贝到 `buf` 的 `sendx` 位置
  - **阻塞等待**：如果 `buf` 满了，当前的 Goroutine 就会把自己打包成 `sudog`，放入 `sendq`，然后调用 `gopark` 挂起，等待被唤醒

- **接收过程 (`v <- ch`)**
  - **直接接收**：如果 `sendq`（等待发送队列）里有 Goroutine，且没有缓冲区，直接从对方那里拿走数据
  - **从缓冲区拿**：如果有缓冲区且不为空，从 `recvx` 位置拿走数据
  - **阻塞等待**：如果缓冲区为空且没人在发送，当前的 Goroutine 放入 `recvq`，调用 `gopark` 挂起

###### Channel 的特殊状态表

| **操作**           | **未初始化 (nil)**  | **已关闭 (closed)**      | **正常 (open)**      |
| ------------------ | ------------------- | ------------------------ | -------------------- |
| **关闭 (close)**   | **Panic**           | **Panic**                | 成功，唤醒所有等待者 |
| **发送 (ch <- v)** | 永久阻塞 (导致死锁) | **Panic**                | 阻塞或成功进入 buf   |
| **接收 (<- ch)**   | 永久阻塞 (导致死锁) | **返回零值**（立即返回） | 阻塞或成功读取       |

###### 无缓冲 vs 有缓冲

- **无缓冲 (`make(chan int)`)**：同步模式。发送者和接收者必须“手递手”交接数据，否则都会阻塞
- **有缓冲 (`make(chan int, 10)`)**：异步模式。只要缓冲区没满，aa发送者就不会阻塞，类似于一个线程安全的阻塞队列

## 第二部分 并发调度模型（GMP 模型）

#### Goroutine 调度

###### GMP 三要素

| **组件**          | **说明**                                                     |
| ----------------- | ------------------------------------------------------------ |
| **G (Goroutine)** | 用户态轻量级线程，初始栈 2KB，可动态扩缩容至 1GB             |
| **M (Machine)**   | 操作系统线程，真正执行代码的实体，默认最多 10000 个          |
| **P (Processor)** | 逻辑处理器，持有本地 goroutine 队列，数量由 `GOMAXPROCS` 决定 |

###### 核心数据结构

```go
// G - Goroutine 描述符
type g struct {
    stack       stack     // 当前栈的范围 [lo, hi)
    stackguard0 uintptr   // 用于栈增长检测（抢占时设为 stackPreempt）
    m           *m        // 当前绑定的 M
    sched       gobuf     // 调度上下文（保存 SP、PC 等寄存器）
    atomicstatus atomic.Uint32 // Goroutine 状态
    goid        uint64    // Goroutine ID
    waitsince   int64     // 阻塞开始时间
    waitreason  waitReason // 阻塞原因
}

// M - 操作系统线程
type m struct {
    g0        *g       // 调度栈（执行调度逻辑的特殊 goroutine）
    curg      *g       // 当前正在运行的 G
    p         puintptr // 绑定的 P
    nextp     puintptr // 预绑定的 P（唤醒时使用）
    spinning  bool     // 是否处于自旋状态（找工作）
    park      note     // 休眠/唤醒的信号量
}

// P - 逻辑处理器
type p struct {
    status    uint32     // P 的状态
    runqhead  uint32     // 本地队列头
    runqtail  uint32     // 本地队列尾
    runq      [256]guintptr // 本地循环队列（最多 256 个 G）
    runnext   guintptr   // 优先运行的下一个 G
    mcache    *mcache    // 内存分配缓存（P 独享，无锁）
}
```

###### Goroutine 的状态流转

```go
_Gidle（刚分配）
    ↓ newproc
_Grunnable（就绪，在队列中等待）
    ↓ schedule()
_Grunning（正在 M 上执行）
    ↓ 主动让出 / 系统调用 / 阻塞
_Gwaiting（等待某个事件，不占 M）
    ↓ 事件就绪（如 channel 有数据）
_Grunnable（重新入队）

_Gsyscall（正在执行系统调用，M 与 P 解绑）
_Gdead（执行完毕，等待复用）
```

###### 全局队列 vs 本地队列

- **本地队列（LRQ）**：每个 P 持有，容量 256，无锁操作，优先从这里取 G 执行
- **全局队列（GRQ）**：全局共享，有锁，当本地队列满时 G 会进入全局队列
- **取 G 优先级**：`runnext` → 本地队列 → 全局队列（每 61 次调度必须取一次全局队列，防止饥饿）→ Work Stealing

#### 调度策略

###### Work Stealing（工作窃取）

当一个 P 的本地队列为空时，不会直接让 M 休眠，而是尝试"偷"其他 P 的工作：

```
本地队列空
    ↓
从全局队列取一批 G
    ↓ 还是空
随机选一个 P，偷取其本地队列后半部分（一半）
    ↓ 还是空
检查 NetPoller（网络就绪事件）
    ↓ 还是空
M 进入自旋（spinning），短暂循环后休眠
```

- **偷取数量**：`n = len(victim.runq) / 2`，最多偷一半，保证被偷方不会饿死
- **自旋 M 数量限制**：最多允许 `GOMAXPROCS/2` 个 M 同时自旋，避免 CPU 空转浪费

###### 调度时机（何时触发调度）

| **触发场景**        | **说明**                                     |
| ------------------- | -------------------------------------------- |
| `runtime.Gosched()` | 主动让出 CPU，G 重新入队                     |
| Channel 阻塞        | G 进入 `_Gwaiting`，M 调度其他 G             |
| 系统调用            | M 与 P 解绑，P 被其他 M 接管                 |
| `time.Sleep`        | G 进入 timer 堆，到期后重新入队              |
| 函数调用时栈检测    | 每次函数调用会检查 `stackguard0`，可触发抢占 |
| GC STW              | 所有 G 被强制停止                            |

###### 系统调用处理

```
G 发起系统调用
    ↓ entersyscall()
M 与 P 解绑（P.status = _Psyscall）
    ↓
P 被其他空闲 M 抢走继续执行其他 G
    ↓
系统调用返回 exitsyscall()
    ↓
尝试重新绑定原来的 P（如果 P 已被占用）
    ↓ 失败
从空闲 P 列表找一个绑定
    ↓ 没有空闲 P
G 进入全局队列，M 进入休眠
```

#### 抢占式调度

###### 两种抢占机制

Go 经历了两代抢占实现：

**① 协作式抢占（Go 1.13 之前）**

- 在每次**函数调用**时，编译器插入栈增长检测代码
- sysmon 发现 G 运行超过 **10ms**，将 `g.stackguard0` 设为 `stackPreempt`
- G 下次发生函数调用时，检测到该标记，主动让出 CPU
- **缺陷**：纯计算循环（无函数调用）无法被抢占，会导致其他 G 饿死

go

```go
// 这段代码在 Go 1.13 之前无法被抢占
for {
    i++ // 纯计算，没有函数调用，永远不会让出
}
```

**② 信号式抢占（Go 1.14+，异步抢占）**

- sysmon 线程检测到 G 运行超过 10ms
- 向该 G 所在的 M 发送 **SIGURG 信号**
- M 的信号处理函数 `doSigPreempt` 中断当前执行
- 在信号处理中保存寄存器上下文，将 G 标记为可抢占并重新调度
- **解决了**纯计算 goroutine 的饿死问题

###### sysmon 后台监控线程

`sysmon` 是一个不需要绑定 P 的特殊线程，每隔 **10~160ms** 唤醒一次，负责：

- 抢占运行时间过长的 G（发送 SIGURG）
- 将长时间阻塞在系统调用的 P 抢走，交给其他 M
- 触发定时的 GC 和内存归还
- 处理网络轮询事件（调用 `netpoll`）

#### 网络轮询器 (NetPoller)

###### 核心设计思路

Go 的网络 I/O 不阻塞操作系统线程，而是通过 NetPoller 将阻塞转化为 Goroutine 级别的等待：

```
G 发起 net.Read()（数据未就绪）
    ↓
将底层 fd 注册到 epoll/kqueue
G 进入 _Gwaiting（不占用 M）
    ↓
M 去执行其他 G
    ↓
epoll 检测到 fd 就绪
sysmon / 调度器 调用 netpoll()
    ↓
将对应的 G 重新置为 _Grunnable，加入运行队列
    ↓
G 被调度执行，Read() 返回数据
```

###### 底层实现（平台相关）

| **操作系统** | **底层机制** |
| ------------ | ------------ |
| Linux        | `epoll`      |
| macOS / BSD  | `kqueue`     |
| Windows      | `IOCP`       |

Go 通过 `internal/poll` 包对上述机制做了统一抽象。

###### pollDesc 结构

每个网络连接对应一个 `pollDesc`，记录：

- 等待读就绪的 G（`rg`）
- 等待写就绪的 G（`wg`）
- 关联的文件描述符
- 读写截止时间（对应 `SetDeadline`）

###### netpoll 调用时机

- `sysmon` 每次循环都会调用 `netpoll(0)`（非阻塞模式）
- 调度器在找不到可运行的 G 时，调用 `netpoll` 阻塞等待 I/O 事件
- GC 的 STW 结束后也会调用，避免长时间 GC 导致网络 G 饿死

## 第三部分 内存管理与垃圾回收（GC）
#### 内存分配系统

###### 核心设计：TCMalloc 思想

Go 的内存分配器基于 TCMalloc（Thread-Caching Malloc）改造，核心思路是**分级缓存**，减少锁竞争：

```go
G 申请内存
    ↓
mcache（P 级别，无锁）
    ↓ 没有合适 span
mcentral（全局，有锁）
    ↓ 没有合适 span
mheap（全局堆，有锁）
    ↓ 堆不够
OS（mmap 申请新内存）
```

###### 核心组件

```go
// mcache - 每个 P 独享的本地缓存（无锁）
type mcache struct {
    alloc [numSpanClasses]*mspan // 按 size class 索引的 span 列表
    tiny        uintptr          // 微对象分配器指针
    tinyoffset  uintptr          // 微对象当前偏移
    tinyAllocs  uintptr          // 微对象分配次数
}

// mspan - 内存管理的基本单元
type mspan struct {
    next     *mspan     // 双向链表
    prev     *mspan
    startAddr uintptr   // 起始地址
    npages   uintptr    // 包含的页数（1 page = 8KB）
    nelems   uintptr    // 该 span 中对象的总数
    freeindex uintptr   // 下一个空闲对象的索引
    allocBits *gcBits   // 位图：哪些对象已分配
    gcmarkBits *gcBits  // 位图：GC 标记
    spanclass spanClass // size class 索引
}
```

###### 对象大小分类

| **类型**       | **大小范围**        | **分配策略**                            |
| -------------- | ------------------- | --------------------------------------- |
| 微对象 (tiny)  | < 16B（且不含指针） | 合并到一个 16B 的 tiny block 中分配     |
| 小对象 (small) | 16B ~ 32KB          | 从对应 size class 的 mspan 分配         |
| 大对象 (large) | > 32KB              | 直接从 mheap 分配，绕过 mcache/mcentral |

###### Size Class

Go 预定义了 **68 个 size class**（从 8B 到 32KB），每个对应一种 span 规格。分配时将请求大小**向上取整**到最近的 size class，用空间换速度：

```
申请 18B → 找到 size class 对应 24B → 从 alloc[sizeclass] 的 mspan 中取一个 slot
```

###### 栈内存管理

- **初始栈大小**：2KB（远小于线程的 8MB，是 Goroutine 轻量的关键）
- **栈增长（Stack Growth）**：每次函数调用检测 `stackguard0`，不够时触发 `morestack`，分配新栈并**拷贝旧栈**（copystack）
- **栈收缩（Stack Shrink）**：GC 扫描时，如果栈使用量不到容量 1/4，则缩容为一半

#### 垃圾回收机制

###### 三色标记法

Go GC 的核心算法，将所有对象分为三种颜色：

| **颜色** | **含义**                                       |
| -------- | ---------------------------------------------- |
| **白色** | 未被扫描，GC 结束后仍为白色的对象将被回收      |
| **灰色** | 已被发现（可达），但其引用的对象还未全部扫描   |
| **黑色** | 已被扫描，且其所有引用也已扫描完毕，不会被回收 |

```
初始：所有对象白色
    ↓
将 GC Root（全局变量、栈变量）标记为灰色
    ↓
循环处理灰色对象：
  扫描灰色对象的所有引用 → 引用的白色对象变灰色
  灰色对象本身变黑色
    ↓
灰色队列为空：回收所有白色对象
```

###### 写屏障（Write Barrier）

并发 GC 的挑战：GC 扫描期间，用户程序可能修改对象引用关系，导致本应存活的对象被误回收。

Go 使用 **混合写屏障（Hybrid Write Barrier，Go 1.8+）**：

go

```go
// 伪代码：赋值 *slot = ptr 时触发
func writeBarrier(slot *unsafe.Pointer, ptr unsafe.Pointer) {
    shade(*slot) // 将旧值标灰（删除写屏障）
    shade(ptr)   // 将新值标灰（插入写屏障）
    *slot = ptr
}
```

- **作用**：确保并发标记期间，无论引用如何变化，存活对象不会被误回收
- **只在堆上生效**：栈上的写操作不加写屏障（栈在 STW 阶段重新扫描）

###### GC 执行阶段（并发三色标记）

```go
① Mark Setup（STW，极短）
   - 开启写屏障
   - 暂停所有 goroutine，扫描栈上的 GC Root

② Mark（并发，与用户代码同时运行）
   - 后台 GC worker goroutine 执行三色标记
   - 用户 goroutine 分配内存时，可能被"借用"协助 GC（Mutator Assist）

③ Mark Termination（STW）
   - 停止所有 goroutine
   - 处理剩余的灰色对象，确保标记完成
   - 关闭写屏障

④ Sweep（并发）
   - 后台并发清扫白色对象，归还内存给 mheap
   - 惰性清扫：下次分配时顺带清扫，不强制 STW
```

###### GC 触发条件

| **触发方式**   | **说明**                                                     |
| -------------- | ------------------------------------------------------------ |
| **堆大小触发** | 堆内存增长到上次 GC 后的 **2倍**（由 `GOGC=100` 控制，默认100%） |
| **定时触发**   | 距上次 GC 超过 **2 分钟**，强制触发                          |
| **手动触发**   | 调用 `runtime.GC()`                                          |

###### GC 调优参数

```go
// GOGC：控制堆增长比例，默认 100（即堆翻倍时触发）
// 设为 200 表示堆增长到 3 倍才触发，减少 GC 频率但增加内存占用
GOGC=200 go run main.go

// GOMEMLIMIT（Go 1.19+）：设置堆内存软上限
// 当内存接近上限时，GC 会更积极地运行
GOMEMLIMIT=4GiB go run main.go

// 代码中动态设置
runtime/debug.SetGCPercent(200)
runtime/debug.SetMemoryLimit(4 * 1024 * 1024 * 1024)
```

###### 逃逸分析与 GC 的关系

- 栈上分配的对象**不受 GC 管理**，函数返回时自动回收，是最高效的分配方式
- 对象发生逃逸（如返回局部变量指针、传入 interface{}）才会分配到堆上，进入 GC 管辖

## 第四部分 运行时核心与工程实践
#### 系统调用（Syscall）

###### 两种系统调用路径

Go 将系统调用分为两类，处理方式不同：

| **类型**                         | **耗时预期**        | **处理方式**                           |
| -------------------------------- | ------------------- | -------------------------------------- |
| **轻量系统调用**（`RawSyscall`） | 极短（纳秒级）      | M 不通知 P，直接调用，不触发调度       |
| **阻塞系统调用**（`Syscall`）    | 可能很长（毫秒级+） | `entersyscall` 解绑 P，让 P 服务其他 G |

###### 阻塞系统调用流程

```go
// runtime/syscall_unix.go 简化版
func Syscall(trap, a1, a2, a3 uintptr) {
    entersyscall()          // 1. 解绑 P，记录调用点
    r := RawSyscall(...)    // 2. 真正的系统调用（阻塞在这里）
    exitsyscall()           // 3. 尝试重新绑定 P
}
entersyscall()
├── 保存当前 SP、PC 到 g.sched
├── 设置 g.status = _Gsyscall
└── 设置 p.status = _Psyscall（但还没真正解绑）

sysmon 检测到 P 在 _Psyscall 状态超过 20us
└── handoffp(p)：将 P 交给其他空闲 M

exitsyscall()
├── 尝试重新 acquire 原来的 P（CAS）
├── 失败则 acquirep(任意空闲 P)
└── 都失败：g 进全局队列，M 休眠
```

###### CGO 与系统调用

调用 CGO（C代码）时，情况更复杂：

- CGO 调用**必然**解绑 P（因为 C 代码不受 Go 调度器管理）
- 需要额外的线程切换开销，CGO 调用频繁时会显著影响性能
- `runtime.LockOSThread()` 可以将 G 锁定到固定 M，用于需要线程本地状态的场景（如 OpenGL、JNI）

#### 初始化流程

###### 程序启动顺序

```
操作系统加载 ELF/Mach-O
    ↓
_rt0_amd64（汇编入口，arch/OS 相关）
    ↓
runtime.rt0_go（汇编）
├── 初始化 g0（调度栈）和 m0（主线程）
├── 调用 runtime.args() 处理命令行参数
├── 调用 runtime.osinit() 获取 CPU 核心数
├── 调用 runtime.schedinit() 初始化调度器
│   ├── 初始化内存分配器（mallocinit）
│   ├── 根据 GOMAXPROCS 创建 P
│   └── 初始化 GC
├── 创建第一个 Goroutine，执行 runtime.main
└── 调用 mstart() 启动调度循环

runtime.main()
├── 启动 sysmon 后台线程
├── 执行所有 init() 函数（按包依赖顺序）
└── 调用用户的 main.main()
```

###### init 函数执行顺序规则

```
同一包内：按源文件名字典序，文件内从上到下
跨包：按依赖关系，被依赖的包先执行 init
同一文件多个 init：从上到下顺序执行

注意：init 函数不能被手动调用，也没有参数和返回值
```

#### 反射（Reflection）

###### 两个核心类型

```go
// reflect.Type：描述类型信息（静态）
t := reflect.TypeOf(x)
t.Kind()    // 底层种类：int, struct, slice, ptr...
t.Name()    // 类型名称
t.NumField() // 结构体字段数

// reflect.Value：描述值（动态，可操作）
v := reflect.ValueOf(x)
v.Type()        // 获取类型
v.Interface()   // 转回 interface{}
v.Elem()        // 解引用指针或获取接口内的值
v.FieldByName("Name") // 访问结构体字段
```

###### 反射三定律

1. **接口值 → 反射对象**：`reflect.TypeOf()` / `reflect.ValueOf()`
2. **反射对象 → 接口值**：`v.Interface()`
3. **修改反射对象需要可寻址**：必须传指针，且通过 `v.Elem()` 操作

```go
x := 1
v := reflect.ValueOf(&x).Elem() // 必须传指针再 Elem()
v.SetInt(2)                      // 才能修改原值
fmt.Println(x)                   // 输出 2
```

###### 性能注意事项

- 反射操作涉及接口装箱（boxing）和类型断言，比直接调用慢 **10~100 倍**
- 热路径（高 QPS 接口）避免使用反射
- 必须用时，可以缓存 `reflect.Type` 和字段索引，减少重复查找开销

#### 错误处理机制

###### error 接口

```go
type error interface {
    Error() string
}

// 自定义错误
type QueryError struct {
    Query string
    Err   error
}
func (e *QueryError) Error() string {
    return fmt.Sprintf("query %q: %v", e.Query, e.Err)
}
func (e *QueryError) Unwrap() error { return e.Err } // 支持 errors.Is/As 链式解包
```

###### errors.Is / errors.As（Go 1.13+）

```go
// errors.Is：判断错误链中是否包含目标错误（值比较）
if errors.Is(err, sql.ErrNoRows) { ... }

// errors.As：从错误链中提取特定类型（类型断言）
var qe *QueryError
if errors.As(err, &qe) {
    fmt.Println(qe.Query)
}

// fmt.Errorf 的 %w 动词：包装错误，保留链
err = fmt.Errorf("service layer: %w", originalErr)
```

###### panic / recover

```go
// panic：抛出运行时异常，沿调用栈向上传播
// recover：只能在 defer 函数中使用，捕获 panic

func safeCall(f func()) (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("recovered panic: %v", r)
        }
    }()
    f()
    return nil
}
```

- **panic 场景**：数组越界、nil 指针解引用、类型断言失败（不用 comma-ok）、显式调用 `panic()`
- **原则**：库代码不应 panic，应返回 error；panic 应只用于"不可能发生"的编程错误

## 第五部分 性能分析与工具链
#### pprof

###### 采集方式

```go
// 方式一：HTTP 接口（推荐，线上可用）
import _ "net/http/pprof"
go http.ListenAndServe(":6060", nil)

// 访问：http://localhost:6060/debug/pprof/
// 常用端点：
// /debug/pprof/goroutine  - goroutine 堆栈
// /debug/pprof/heap       - 堆内存分配
// /debug/pprof/profile    - 30s CPU 采样
// /debug/pprof/allocs     - 内存分配（含已回收）
// /debug/pprof/mutex      - 锁竞争

// 方式二：代码中直接写文件
f, _ := os.Create("cpu.pprof")
pprof.StartCPUProfile(f)
defer pprof.StopCPUProfile()
```

###### 分析命令

```bash
# 交互式分析
go tool pprof http://localhost:6060/debug/pprof/heap

# 常用命令
(pprof) top10          # 按消耗排序前10
(pprof) list funcName  # 查看某函数的逐行消耗
(pprof) web            # 在浏览器中查看火焰图（需安装 graphviz）

# 直接生成火焰图
go tool pprof -http=:8080 cpu.pprof
```

###### 各 Profile 类型对应问题

| **Profile**              | **诊断场景**             |
| ------------------------ | ------------------------ |
| `cpu`                    | CPU 占用高、热点函数     |
| `heap` (inuse_space)     | 内存泄漏、常驻内存大     |
| `allocs` (alloc_objects) | GC 压力大、内存分配过多  |
| `goroutine`              | goroutine 泄漏、死锁排查 |
| `mutex`                  | 锁竞争导致的性能瓶颈     |
| `block`                  | channel/锁 阻塞时间过长  |

#### Trace

###### 采集与查看

```go
// 采集
f, _ := os.Create("trace.out")
trace.Start(f)
defer trace.Stop()

// 查看（浏览器打开交互式时间线）
go tool trace trace.out
```

###### Trace vs pprof 的区别

- **pprof**：统计型，回答"哪里消耗多"，适合定位热点
- **trace**：时序型，回答"什么时候发生了什么"，适合诊断延迟毛刺、调度问题

trace 可以看到：

- 每个 P 上 goroutine 的调度时间线
- GC 各阶段的时间占比（STW 多长）
- 系统调用耗时
- 网络阻塞时间

#### 逃逸分析 (Escape Analysis)

###### 触发逃逸的常见场景

```go
// ① 返回局部变量的指针
func newUser() *User {
    u := User{} // u 逃逸到堆
    return &u
}

// ② 赋值给 interface{}（类型信息丢失，编译器无法确定大小）
var i interface{} = someStruct // someStruct 逃逸

// ③ 闭包捕获外部变量
func counter() func() int {
    n := 0 // n 逃逸，因为闭包生命周期可能超过 counter()
    return func() int { n++; return n }
}

// ④ 切片/map 动态扩容（编译期大小未知）
s := make([]int, n) // n 不是常量时，可能逃逸

// ⑤ 大对象（通常 > 64KB 直接分配到堆）
```

###### 查看逃逸分析结果

```bash
go build -gcflags="-m -m" ./...

# 输出示例：
# ./main.go:10:6: moved to heap: u    ← 逃逸
# ./main.go:15:2: n does not escape   ← 未逃逸（栈分配）
```

###### 逃逸对性能的影响

|          | **栈分配**         | **堆分配**           |
| -------- | ------------------ | -------------------- |
| 速度     | 极快（移动栈指针） | 较慢（需分配器处理） |
| GC 压力  | 无                 | 有（增加 GC 扫描量） |
| 生命周期 | 函数返回即释放     | GC 决定              |

#### 汇编基础

###### Plan 9 汇编（Go 使用的汇编方言）

```asm
// 函数定义格式
// TEXT packagename·funcname(SB), flags, $framesize-argsize
TEXT main·add(SB), NOSPLIT, $0-16
    MOVQ a+0(FP), AX    // 读取第一个参数（FP = Frame Pointer）
    MOVQ b+8(FP), BX    // 读取第二个参数
    ADDQ BX, AX          // AX = AX + BX
    MOVQ AX, ret+16(FP) // 写返回值
    RET
```

###### 常用伪寄存器

| **寄存器** | **含义**                                                     |
| ---------- | ------------------------------------------------------------ |
| `FP`       | Frame Pointer，函数参数和返回值的基址                        |
| `SP`       | Stack Pointer，指向局部变量区（Go 汇编中 SP 与硬件 SP 可能不同） |
| `PC`       | Program Counter，指令指针                                    |
| `SB`       | Static Base，全局符号基址（用于函数名、全局变量）            |

###### 查看 Go 代码对应的汇编

```bash
# 查看汇编输出
go tool compile -S main.go

# 或者
go build -gcflags="-S" main.go

# 反汇编二进制
go tool objdump -s "main\.main" ./binary
```

## 第六部分 附录

#### Go标准库

- **context**: 用于在 API 边界之间传递截止时间、取消信号和其他请求范围的值。在数据库操作中，它常用于控制查询超时或手动取消执行
- **errors**: 用于处理和创建错误。驱动程序经常需要定义或包装特定的数据库错误
- **fmt**: 格式化 I/O，比如打印日志信息或拼接简单的字符串
- **log/slog**: Go 1.21 引入的结构化日志库。用于输出带有键值对的日志，方便在生产环境中进行追踪和排查
- **math/rand**: 用于生成伪随机数。在数据库驱动中，常用于在多个集群节点之间进行**负载均衡**（随机选择一个节点连接）
- **sync & sync/atomic**: 并发控制原语。驱动程序是线程安全的，需要这些工具来管理连接池的状态、计数器或读写锁
- **time**: 处理时间段、打点器和超时逻辑
- **_ "time/tzdata"**: 这是一个特殊的**匿名导入**。它将时区数据库嵌入到二进制文件中。即使运行环境（如某些 Docker 镜像）没有时区文件，程序也能正确处理时区转换

###### Go语法

- := 表示声明 + 赋值；= 表示赋值

- opt *Options 表示指针传递；opt Options 表示值传递

- Error() 表示公共方法；error() 表示私有方法

- type Conn = driver.Conn 表示类型声明

- func (e *OpError) Error() 表示为 OpError 结构体实现了 Error() 方法，声明它是一个异常

- switch err := e.Err.(type) 表示根据变量拿类型进行匹配

- &Options 表示返回该变量的地址

- 包内透明：如果不同的 Go 文件声明了同一个 package，函数、结构体、变量、常量，可以直接访问

- 定义结构体

  ```go
  type clickhouse struct {
      opt    *Options
      connID int64
  
      idle connectionPooler
      open chan struct{}
  
      closeOnce *sync.Once
      closed    *atomic.Bool
  }
  ```

- Slice 声明和初始化：src := []int{1, 2, 3}；dst := make([]int, 2)

- Map 声明和初始化：var m map[string]int；m1 := make(map[string]int)

- if asyncOpt := queryOptionsAsync(ctx); asyncOpt.ok，声明变量后进行 if 条件判断

- (ch *clickhouse) Exec(ctx context.Context, query string, args ...any) error 表示 args 可传多个参数，返回值为error

- for index, opt := range opts 表示 for each 循环，index 表示下标，opt表示值

- defer cancel() 表示在程序执行结束后自动执行 cancel() 函数

- select 表示阻塞监听，随机挑选，谁快选谁，同时监听多个 case

  ```go
  select {
  case ch.open <- struct{}{}:
  case <-ctx.Done():
      return nil, context.Cause(ctx)
  }
  ```


## 第七部分 Clickhouse-Go源码

#### ClickHouse 核心组件

###### lib/column

- **列式存储定义**：ClickHouse 是列式数据库，这个包定义了不同数据类型（如 UInt32, DateTime）在内存中的表示和处理方式

###### lib/driver

- **驱动接口层**：它定义了用户调用的 API 标准（比如 `Conn` 接口），确保驱动符合 `clickhouse-go` 的设计规范
- **Conn**：执行SQL查询（Query、QueryRaw、Select）、执行命令（Create、Alter、Drop）、高性能批量写入（PrepareBatch）、异步插入、连接管理与健康（Ping、Close、ServerInfo）

###### lib/proto

- **交互协议**：这里实现了 ClickHouse 的二进制通讯协议。它负责将 Go 的数据结构编码（序列化）成 ClickHouse 能听懂的字节流，或反向解码
- **Progress**：查询进度更新，当你执行一个耗时很长的查询时，ClickHouse 会不断发回进度信息
- **Exception**：服务器异常，当 ClickHouse 返回错误时，驱动会将其解析为这个结构体，包含了错误码（Code）和堆栈信息
- **ProfileInfo**：性能剖析信息，记录了查询执行时的底层指标，比如使用了多少内存、读取了多少行数据、涉及了多少列等
- **ServerVersion**：服务器版本/握手信息，在连接建立初期，服务器会告知自己的版本号、时区、显示名称等

###### Go标准库接口方法

- **String()**：实现标准结构体打印方法
- **ServeHTTP(w http.ResponseWriter, r *http.Request)**：结构体例吗变成Web服务器
- **Read()**：结构体像“文件”或“网络流”一样被读取
- **Write(p []byte) (n int, err error)**：可以把数据往里写
- **MarshalJSON() ([]byte, error)**：按照自定义逻辑转Json
- **Error()**：定义该结构体是一个error

###### NativeTransport

- **执行查询**
  - **query / queryRow**：执行 SQL 并返回一行或多行数据
  - **exec**：执行不需要返回数据的命令（如 `CREATE TABLE`）
  - **prepareBatch**：这是 ClickHouse 的灵魂，用于高性能的批量写入
  - **asyncInsert**：支持 ClickHouse 特有的异步插入功能
- **连接状态管理**
  - **ping**：检查心跳，看数据库还活着没
  - **isBad**：判断这个连接是不是已经坏了（比如网络断了）
  - **connID**：每个连接的唯一身份证，方便排查日志
  - **connectedAtTime**：记录连接是什么时候创建的，用于判断连接是否过期（MaxLifetime）
- **生命周期管理**
  - **isReleased` / `setReleased**：标记这个连接是否已经回到了连接池（防止被重复使用）
  - **close**：彻底关掉底层的网络插座（Socket）
  - **freeBuffer**：**性能优化**，手动释放内存缓冲区，减少 GC 压力