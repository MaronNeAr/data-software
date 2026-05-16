## Linux源码学习

###### open系统调用

- **系统调用宏**：系统调用宏是 Linux 内核开发者为了**安全、规范、且自动化地向用户态暴露内核接口**而设计的一套顶级 C 语言宏家族

- `SYSCALL_DEFINE3(open, const char __user *, filename, int, flags, umode_t, mode)`

- `const char __user *`：指向字符常量的指针

- `__user`：表示明确标记一个指针指向的是“用户态虚拟内存”，警告内核绝对不能直接在内核态通过解引用（ `*ptr`）去读写它

- `filename`：文件路径

- `flags`：打开标志（如O_RDONLY、O_WRONLY、O_CREAT等）

- `mode`：文件权限（在创建文件时使用）

- `do_sys_open()`：系统调用进入内核后，真正进入“统一 open 后端”的一层包装函数

- `build_open_how(flags, mode)`：把传统的 flags 和 mode 转化成 struct open_how，清洗 flags、规范 mode、兼容0_PATH

  - `open_how`：Linux 为了解决老旧的 `openat` 无法再追加新功能、以及防范**容器和沙箱逃逸漏洞**，resolve字段代表路径解析掩码

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

  - `CLASS(filename, name)(filename)`：从用户空间复制 `filename` 到内核临时 `struct filename`

  - `FD_ADD`：实现了链式调用与资源安全绑定的合二为一

  - `do_file_open`：将用户路径和 `open_flags` 进行路径查找与打开操作，成功返回已打开的 `struct file *`，失败返回 `ERR_PTR(-errno)`
