- **GOROOT**：GO源码包所在路径

  - **默认值**：`GOROOT=/usr/local/go`

- **GOPATH**：GO的项目默认路径

  - **默认值**：`export GOPATH=$HOME/go`

- **GOPROXY**：Go 下载依赖时，不直接去 GitHub，而是先去 GOPROXY 指定的镜像服务器取，更快更稳定

  - **当前配置**：`go env GOPROXY`

  - **默认值**：`GOPROXY=https://proxy.golang.org,direct`

  - **国内镜像**：`go env -w GOPROXY=https://goproxy.cn,direct`

- **Go的优势**

  - **简单部署**：可直接编译成机器码；不依赖其他库；直接运行即可部署
  - **静态类型**：变编译的时候检查出来的大多数问题
  - **天生并发**：充分利用多核，天生支持并发
  - **GO标准库**：runtime系统调度机制；高效的GC拉饥回收；丰富的标准库
  - **简单易学**：25个关键字；面向对象特征（继承、多态、封装）；跨平台

- **25个关键字**

  | 分类         | 关键字                                                       |
  | ------------ | ------------------------------------------------------------ |
  | **程序结构** | `package` `import` `func` `return` `var` `const` `type`      |
  | **流程控制** | `if` `else` `for` `range` `break` `continue` `goto` `fallthrough` `switch` `case` `default` |
  | **并发**     | `go` `chan` `select`                                         |
  | **复合类型** | `map` `struct` `interface`                                   |
  | **内存管理** | `defer`                                                      |

- **go mod版本控制工具**

  - **初始化模块**：`go mod init github.com/myname/myproject`
  - **下载缺失的依赖**：`go mod tidy`
  - **下载依赖到本地缓存**：`go mod download`
  - **查看依赖图**：`go mod graph`
  - **验证依赖完整性**：`go mod verify`

- **go defer应用场景**

  - **资源释放**：`defer f.Close() `

  - **锁的释放**：`defer mu.Unlock() `

  - **panic恢复**

    ```go
    func safeCall() {
        defer func() {
            if r := recover(); r != nil {
                fmt.Println("捕获 panic:", r)
                // 记录日志、返回错误等
            }
        }()
      
        panic("出错了！")  // 被 defer+recover 捕获，程序不会崩溃
    }
    ```

  - **计时/性能统计**：`defer timeTrack(time.Now(), "slowFunc")`

  - **日志追踪**：`defer fmt.Println("doSomething 结束")`

- **匿名函数**

  ```go
  func() {
  	if r := recover(); r != nil {
  		fmt.Println("捕获 panic:", r)
      // 记录日志、返回错误等
    }
  }()
  ```

- **Go早期调度器的缺点**

  - 创建、销毁、调度G都需要每个M获取锁，这就形成了激烈的锁竞争
  - M转移G会造成延迟和额外的系统负载
  - 系统调用（CPU在M之间的切换）导致频繁的线程阻塞和取消阻塞操作增加了系统开销

- **关闭channel**

  - channel不像文件一样需要经常去关闭，只有当你确实不需要发送任何数据了，或者你想显式的结束range循环之类的，才会关闭channel
  - 关闭channel后，无法向channel再发送数据（引发panic错误后导致接收立即返回0值）
  - 关闭channel后，可以继续从channel中接受数据
  - 对比nil channel，无论收发都会阻塞
