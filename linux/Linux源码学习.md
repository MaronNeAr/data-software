# Linux源码学习

## 第一部分 进程与计算管理（生命的诞生与调度）

#### fork 系统调用



#### clone 系统调用



## 第二部分 内存管理（虚拟与物理的魔术）

#### brk / sbrk 改变堆栈边界



#### mmap / munmap 内存映射



## 第三部分 虚拟文件系统 VFS 和存储I/O

#### open 系统调用

- `open.c`：文件打开的策略层，负责"打开一个文件"这件事的整体编排

  ```c
  系统调用入口
    open() / openat() / openat2()
          │
          ▼
  参数处理和验证
    do_sys_open()
    do_sys_openat2()
    build_open_flags()
          │
          ▼
  文件对象生命周期
    alloc_empty_file()   分配 file
    fput() / fput_close() 释放 file
          │
          ▼
  fd 管理
    get_unused_fd_flags()
    fd_install()
          │
          ▼
  其他文件操作系统调用
    truncate() / ftruncate()
    chmod() / fchmod()
    chown() / fchown()
    access() / faccessat()
    chdir() / fchdir()
  ```

- `namei.c`：路径解析的执行层，负责"给定一个路径字符串，找到对应的 inode"

  ```c
  路径解析核心
    path_init()          确定解析起点
    link_path_walk()     逐级解析路径分量
    open_last_lookups()  处理最后分量
    terminate_walk()     清理解析状态
          │
          ▼
  目录项操作
    lookup_dcache()      查 dentry 缓存
    lookup_slow()        缓存未命中，查文件系统
          │
          ▼
  符号链接处理
    follow_link()        展开符号链接
    follow_dotdot()      处理 ".."
          │
          ▼
  文件创建/删除
    vfs_create()
    vfs_unlink()
    vfs_rename()
    vfs_mkdir() / vfs_rmdir()
    vfs_link() / vfs_symlink()
          │
          ▼
  权限检查
    inode_permission()
    may_open()
  ```

- `open.c`和`namei.c`调用关系

  ```c
  fs/open.c                        fs/namei.c
  ─────────────────                ──────────────────
  do_sys_openat2()
    │
    ├─ build_open_flags()
    │
    └─ do_file_open()
           │
           └─ path_openat()  	─────▶  path_init()
                  │           ─────▶  link_path_walk()
                  │           ─────▶  open_last_lookups()
                  │           ─────▶  do_open()
                  │                      │
                  │           ─────▶     may_open()
                  │           ─────▶     vfs_open()
                  │
                  └─ alloc_empty_file()
                  └─ terminate_walk()
  ```

- 系统调用宏：系统调用宏是 Linux 内核开发者为了安全、规范、且自动化地向用户态暴露内核接口而设计的一套顶级 C 语言宏家族

- `SYSCALL_DEFINE3(open, const char __user *, filename, int, flags, umode_t, mode)`

- `const char __user *`：指向字符常量的指针

- `__user`：表示明确标记一个指针指向的是“用户态虚拟内存”，警告内核绝对不能直接在内核态通过解引用（ `*ptr`）去读写它

- `filename`：文件路径

- `flags`：打开标志（如O_RDONLY、O_WRONLY、O_CREAT等）

  | 常见标志     | 含义               |
  | ------------ | ------------------ |
  | `O_RDONLY`   | 只读               |
  | `O_WRONLY`   | 只写               |
  | `O_RDWR`     | 读写               |
  | `O_CREAT`    | 不存在则创建       |
  | `O_TRUNC`    | 打开时清空文件     |
  | `O_APPEND`   | 追加写入           |
  | `O_NONBLOCK` | 非阻塞模式         |
  | `O_CLOEXEC`  | exec 时自动关闭 fd |

- `mode`：文件权限（在创建文件时使用）

  ```c
  // 权限位示例
  S_IRUSR = 0400  // owner 可读
  S_IWUSR = 0200  // owner 可写
  S_IXUSR = 0100  // owner 可执行
  S_IRGRP = 0040  // group 可读
  S_IROTH = 0004  // others 可读
  ```

- `do_sys_open()`：系统调用进入内核后，真正进入“统一 open 后端”的一层包装函数

- `build_open_how(flags, mode)`：把传统的 flags 和 mode 转化成 struct open_how，清洗 flags、规范 mode、兼容0_PATH

- `open_how`：Linux 为了解决老旧的 `openat` 无法再追加新功能、以及防范**容器和沙箱逃逸漏洞**，resolve字段代表路径解析掩码

  - 数据结构

    ```c
    // include/uapi/linux/openat2.h
    struct open_how {
        __u64 flags;    // 即原来的 open flags（O_RDONLY、O_CREAT 等）
        __u64 mode;     // 即原来的 mode（权限位，如 0644）
        __u64 resolve;  // 新增：控制路径解析行为 ← 核心新增字段
    };
    ```

  - resolve 字段

    | 标志                    | 作用                                         |
    | ----------------------- | -------------------------------------------- |
    | `RESOLVE_NO_XDEV`       | 禁止路径解析跨越挂载点                       |
    | `RESOLVE_NO_MAGICLINKS` | 禁止跟随 `/proc/self/fd/` 类魔法链接         |
    | `RESOLVE_NO_SYMLINKS`   | 禁止跟随任何符号链接                         |
    | `RESOLVE_BENEATH`       | 路径必须在指定目录**之下**，防止 `../` 逃逸  |
    | `RESOLVE_IN_ROOT`       | 把 `dfd` 当作进程的虚拟根目录（类似 chroot） |
    | `RESOLVE_CACHED`        | 只走缓存路径，若需要 I/O 则直接失败          |

- `0_PATH`：是 Linux 的一个特殊 `open` 标志，它不是真的“打开文件内容”，而是打开一个只表示路径的文件描述符

- `do_sys_openat2()`：用来把 `struct open_how` 转换成内核可执行的打开参数，并最终调用 `do_file_open()`

  - `build_open_flags(how, &op)`：将`open_how`转化为`open_flags`

    ```c
    struct open_flags {
        int open_flag;      // 翻译并清洗后的核心控制位（内核真正认可的 O_xxx）
        umode_t mode;       // 文件的创建权限掩码（如 0644）
        int acc_mode;       // 极其关键：内核内部专属的“访问权限掩码”（MAY_READ, MAY_WRITE）
        int intent;         // 内核的“意图标志”（比如是纯查找 LOOKUP_OPEN，还是顺便要创建 LOOKUP_CREATE）
        int lookup_flags;   // 路径查找控制位（比如是否允许跟踪软链接）
    };
    ```

  - `CLASS(filename, name)(filename)`：从用户空间复制 `filename` 到内核临时 `struct filename`，等价于`struct filename *name = getname(filename);`登记一个清理动作，函数任意出口自动执行（类似于 Go 中的 defer）

  - `FD_ADD`：负责将文件对象安装到进程的文件描述符表，返回 fd 数字

    ```c
    // include/linux/file.h
    #define FD_ADD(flags, file) \
        fd_install_fresh(get_unused_fd_flags(flags), file)
    ```

    - `get_unused_fd_flags(flags)`：在当前进程的文件描述符表里找一个**空闲槽位**，返回其下标
    - `fd_install_fresh(fd, file)`：把 file 写入刚刚分配的槽位

  - `do_file_open`：将用户路径和 `open_flags` 进行路径查找与打开操作，成功返回已打开的 `struct file *`，失败返回 `ERR_PTR(-errno)`

- `open()`和`openat()`

  - `int open(const char *pathname, int flags, mode_t mode);`
  - `int openat(int dirfd, const char *pathname, int flags, mode_t mode);`
  - openat 函数比 open 多了 dirfd，表示目录文件描述符
  - 如果 filename 是绝对路径，dirfd 直接呗忽略，效果和 open() 一样
  - 如果 filename 是相对路径，内核不再去看进程的 CWD，而是以 dirfd 指向的特定目录为基准，去解析后面的相对路径

- `openat()`和`openat2()`

  - `int openat(int dirfd, const char *pathname, int flags, mode_t mode);`
  - `int openat2(int dirfd, const char *pathname, struct open_how *how, size_t size);`
  - 把控制参数打包成了 open_how，引入了`how->resolve`（路径解析掩码），调用者可以直接干预、限制内核 VFS 路径解析行为的至高特权

- `do_file_open()`：把路径字符串解析成一个可用的 struct file *，核心是调用 path_openat()

  - 整体流程

    ```c
    pathname ("/home/user/foo.txt")
        │
        ▼
    set_nameidata        初始化解析上下文
        │
        ▼
    path_openat (RCU)    无锁快速解析
        │
        ├─ 成功 ──────────────────────────▶ struct file*
        │
        ├─ -ECHILD（RCU冲突）
        │      ▼
        │  path_openat (普通)    加锁解析
        │      │
        │      ├─ 成功 ──────────────────▶ struct file*
        │      │
        │      └─ -ESTALE（缓存过期）
        │             ▼
        │         path_openat (REVAL)  强制验证
        │             │
        │             └──────────────▶ struct file*
        │
        ▼
    restore_nameidata    清理上下文
    ```

  - `set_nameidata(&nd, dfd, pathname, NULL)`初始化路径解析上下文，把 dfd 和 pathname 填进去，作为解析的起点

    ```c
    struct nameidata {
        struct path    path;      // 当前解析到的位置
        struct qstr    last;      // 当前路径分量（如 "user"）
        struct path    root;      // 根目录
        struct inode  *inode;     // 当前 inode
        unsigned int   flags;     // 解析标志
        int            last_type; // 最后一个分量的类型
        unsigned       depth;     // 符号链接嵌套深度
        int            total_link_count;
        // ...
    };
    ```

  - LOOKUP_RCU（Read-Copy-Update）：无锁快速路径，是内核中一种**无锁并发读**机制

  - LOOKUP_REVAL：强制重新验证，内核维护一个 dentry cache，把路径解析结果缓存在内存里

- `path_openat()`：把 `nameidata`转化成一个完全初始化好的 `struct file *`，是路径解析和文件打开的**核心执行体**

  - `__O_TMPFILE`：匿名临时文件。没有目录项的匿名文件；不在任何目录中，其他进程看不到；进程退出或 fd 关闭后自动消失
  - `O_PATH`：路径引用。不真正打开文件，只获取路径引用，不检查文件读写权限
  - `path_init(nd, flags)`：确定解析起点，返回待解析的路径字符串
  - `link_path_walk(s, nd)`：负责按当前目录查找 /home/user/，`foo.txt` 留给 `open_last_lookups`进行处理
  - `terminate_walk(nd)`：清理路径解析过程中持有的引用（dentry 引用、RCU 临界区等）

- `do_path()`：路径解析已经完成，inode 已经找到，`do_open` 负责最后一公里：权限检查、处理各种 flag 语义、真正打开文件

  - `FMODE_CREATED`：文件是刚刚新建的（O_CREAT 且之前不存在）

  - `FMODE_OPENED`：文件已经被某个快速路径打开了

  - `audit_inode`：Linux 审计子系统（`auditd`）用这个来追踪文件访问行为，安全合规场景下必须记录

  - `mnt_idmap`：区间映射表，把"磁盘上存的 uid/gid"翻译成"当前挂载上下文里实际的 uid/gid"

  - `O_CREAT`：如果目标文件不存在则创建它；创建时使用 `open()` 的第三个参数 `mode` 设置权限

  - `O_EXCL`：原子性检查文件是否存在，常用于锁文件，排他性的创建

  - `sticky`位检查：检查 sticky 目录下的创建权限（只有目录所有者或有写权限的用户才能在 sticky 目录中创建/删除他人文件），失败则返回相应错误

  - `d_can_lookup`：检查 path.dentry 是否是目录，是否可以按照文件名查找

  - `do_truncate`：是否执行截断操作

  - `O_TRUNC`：打开文件时把文件内容清空（长度截为0）

  - `acc_mode`：表示这次 open 需要什么访问权限，是个位掩码，为 0 表示跳过权限检查

    ```c
    #define MAY_READ    0x04   // 需要读权限
    #define MAY_WRITE   0x02   // 需要写权限
    #define MAY_EXEC    0x01   // 需要执行权限
    ```

  - 权限校验：新文件跳过截断和权限检查，旧文件老老实实走完整流程

  - `d_is_reg`：检查该 denstry 是不是一个普通文件

  - `mnt_want_write`：申请挂载点写权限

- `denstry`：是 Linux VFS 层的一个内存对象，表示路径中的一个分量

  ```c
  磁盘                    内存（VFS层）
  
                      dentry ("foo.txt")
                      ├─ d_name = "foo.txt"    ← 文件名
                      ├─ d_parent → dentry("user")  ← 父目录
                      ├─ d_inode ──────────────────────┐
                      └─ d_op                          │
                                                       ▼
                      inode (inode号 42)
  ext4 inode 42  ←── ├─ i_mode  (权限)
                      ├─ i_size
                      ├─ i_fop  (file_operations)
                      └─ i_mapping (页缓存)
  
                      file (每次open一个)
                      ├─ f_path.dentry → dentry
                      ├─ f_inode       → inode
                      ├─ f_pos         (当前读写位置)
                      └─ f_op          → inode->i_fop
  ```

  | 结构       | 代表什么               | 生命周期                     |
  | ---------- | ---------------------- | ---------------------------- |
  | **dentry** | 路径树中的一个名字节点 | 可缓存（dcache），与路径绑定 |
  | **inode**  | 文件本身（数据、权限） | 与磁盘上的 inode 对应        |
  | **file**   | 一次 open 的上下文     | 每次 open 创建，close 销毁   |

#### close / rename 系统调用



#### read / write 系统调用



#### pread / pwrite 系统调用



#### fsync / fallocate 系统调用



#### 文件系统

- 进程的文件描述符表

  ```
  task_struct
      │
      └─▶ files_struct
                │
                └─▶ fdtable
                       │
                       └─▶ fd[]
                             [0] ──▶ struct file (stdin)
                             [1] ──▶ struct file (stdout)
                             [2] ──▶ struct file (stderr)
                             [3] ──▶ struct file (刚 open 的文件)
                             [4]  =  NULL
                             ...
  ```

  - fd 只是一个数组下标，用户态拿着这个整数，内核用它索引到真正的文件对象
  - Unix "一切皆文件" 能成立的基础结构——所有 IO 资源都通过同一张表、同一套 fd 接口统一管理

- inode：文件的本体

  ```c
  struct inode {
      umode_t          i_mode;    // 文件类型和权限
      kuid_t           i_uid;     // 所有者
      kgid_t           i_gid;
      loff_t           i_size;    // 文件大小
      struct timespec  i_mtime;   // 修改时间
      unsigned long    i_ino;     // inode 号（全局唯一）
      
      const struct inode_operations *i_op;   // mkdir, link, lookup...
      const struct file_operations  *i_fop;  // read, write, mmap...
      
      struct address_space *i_mapping;  // 页缓存，文件数据在这里
  };
  ```

  - 文件系统内唯一，存储文件元数据，不存文件名
  - 硬链接 = 多个目录项指向同一个 inode
  - 文件被 unlink 后 inode 不立即释放，引用计数归零才释放

- Linux文件类型

  | 类型     | 对应检查函数                | 例子               |
  | -------- | --------------------------- | ------------------ |
  | 普通文件 | `d_is_reg`                  | `foo.txt`、`a.out` |
  | 目录     | `d_is_dir` / `d_can_lookup` | `/home/user`       |
  | 符号链接 | `d_is_symlink`              | `ln -s` 创建的     |
  | 字符设备 | `d_is_chr`                  | `/dev/tty`         |
  | 块设备   | `d_is_blk`                  | `/dev/sda`         |
  | 管道     | `d_is_fifo`                 | `mkfifo` 创建的    |
  | socket   | `d_is_sock`                 | Unix domain socket |

## 第四部分 网络与进程间通信

#### socket



#### bind / listen / accept



#### epoll_create / epoll_ctl / epoll_wait



#### pipe



## 第五部分 现代 Linux 应用

#### io_uring



