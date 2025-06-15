## java面试

##### String、StringBuffer、StringBuilder的区别

![b5a91fb8-18c1-400d-a8c1-18c4ef57db8b](file:///C:/Users/hua'wei/Pictures/Typedown/b5a91fb8-18c1-400d-a8c1-18c4ef57db8b.png)

java String类是个final类，不能被继承

##### Final和static：

final和const：const是编译时常量，final是运行时常量。

Final修饰的类不能被继承、修饰常量不能被更改

Static表示全局静态，与类有关，与实例无关

静态方法属于类，但不是类的对象

Map.entry<>遍历map;{map.getKey():map.getValue()}

Arrays排序：

Arrays.sort(arr)/Arrays.sort(arr,Collections.reverseOrder())

ArrayList排序：

arraylist.sort(Comparator. naturalOrder())/ArrayList.sort(Comparator. reverseOrder())

Queue：add（返回异常）offer（返回false）poll、element（头部、返回异常）、peek（头部元素，返回null）

Stack：add（返回false）、push（返回元素）、pop（返回元素）

![image-20230315134556126](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134556126.png)

Collection包括List（ArrayList、LinkedList）、Map（HashMap、TreeMap[SortedMap]）、Set(HashSet、TreeSet[SortedSet])、Queue（LinkedList）

Set<Map.Entry<>>set=map.entrySet()

For(Map.Entry<>me:set)me.getKey();me.getValue();

Iteator< Map.Entry<>>iter=set.iterator()

While (iter.hasNext()) me=iter.next(); me.getKey();me.getValue();

##### Optional简介

1. `of()`：创建一个包含非空值的 Optional 实例，如果传入的值为 null，会抛出 NullPointerException。
2. `ofNullable()`：创建一个 Optional 实例，包含传入的值，但如果传入的值为 null，则创建一个空的 Optional 实例。
3. `empty()`：创建一个空的 Optional 实例，不包含任何值。
4. `isPresent()`：判断 Optional 实例是否包含非空值，如果包含返回 true，否则返回 false。
5. `ifPresent(Consumer<? super T> consumer)`：如果 Optional 实例包含非空值，则执行传入的 Consumer 操作。
6. `orElse(T other)`：如果 Optional 实例包含非空值，则返回该值，否则返回传入的默认值 other。
7. `orElseGet(Supplier<? extends T> other)`：如果 Optional 实例包含非空值，则返回该值，否则通过传入的 Supplier 生成一个默认值。
8. `orElseThrow(Supplier<? extends X> exceptionSupplier)`：如果 Optional 实例包含非空值，则返回该值，否则通过传入的 Supplier 抛出一个异常。
9. `map(Function<? super T, ? extends U> mapper)`：对 Optional 中的值进行映射转换。
10. `flatMap(Function<? super T, Optional<U>> mapper)`：对 Optional 中的值进行映射转换，并返回一个新的 Optional 对象。

##### String和StringBuffer优缺点

在Java中，String是一个不可变对象，这意味着每次对字符串进行操作时，都会创建一个新的String对象。而StringBuffer是一个可变的字符串类，可以在不创建新对象的情况下修改字符串。

虽然StringBuffer的结构可以提高字符串操作的效率，但是它也有一些缺点。由于StringBuffer是一个可变的字符串类，它的内部实现使用了可变大小的字符数组，这些数组的大小可以根据需要进行动态调整。这样，在进行字符串操作时，需要频繁地进行内存分配和复制，这会带来一定的开销。

相比之下，String的内部实现采用了一种称为"共享池"的机制。它会将所有的字符串字面量都存储在一个全局的字符串池中，并且对于相同的字符串字面量，只会存储一份。这样，多个字符串对象可以共享同一个字符串字面量，从而节省了内存空间。

##### HashMap和Hashtable的区别

HashMap和Hashtable都实现了Map接口，但决定用哪一个之前先要弄清楚它们之间的分别。主要的区别有：线程安全性，同步(synchronization)，以及速度。

1. HashMap几乎可以等价于Hashtable，除了HashMap是非synchronized的，并可以接受null(HashMap可以接受为null的键值(key)和值(value)，而Hashtable则不行)。

2. HashMap是非synchronized，而Hashtable是synchronized，这意味着Hashtable是线程安全的，多个线程可以共享一个Hashtable；而如果没有正确的同步的话，多个线程是不能共享HashMap的。Java 5提供了ConcurrentHashMap，它是HashTable的替代，比HashTable的扩展性更好。

3. 另一个区别是HashMap的迭代器(Iterator)是fail-fast迭代器，而Hashtable的enumerator迭代器不是fail-fast的。所以当有其它线程改变了HashMap的结构（增加或者移除元素），将会抛出ConcurrentModificationException，但迭代器本身的remove()方法移除元素则不会抛出ConcurrentModificationException异常。但这并不是一个一定发生的行为，要看JVM。这条同样也是Enumeration和Iterator的区别。

4. 由于Hashtable是线程安全的也是synchronized，所以在单线程环境下它比HashMap要慢。如果你不需要同步，只需要单一线程，那么使用HashMap性能要好过Hashtable。

5. HashMap不能保证随着时间的推移Map中的元素次序是不变的。

![image-20230315134615034](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134615034.png)

HashMap和HashTable的区别：

![image-20230315134627198](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134627198.png)

HashTable更均匀 HashMap更高效

Synchronized：同步锁

Jdbc：

Connection：用于创建数据库连接

statement：用于执行sql语句

##### Java面向对象编程OOP

三大特点：封装、多态（重写、继承）、继承

封装、继承：增加代码的复用性 多态：灵活性、健壮性、可移植性

Java类的静态方法可以直接和类一起调用、非静态方法只能实例化调用

Java引用类型（强软弱虚（垃圾回收方式不同））

System.gc()java 自动回收

强引用：只要有对象指着就不会被回收

软引用：SoftReference<>空间够就分配，不够就被释放（cache）。

弱引用：WeakReference<>管你空间够不够用，直接释放

虚引用：任何时候都会被回收 主要用来跟踪对象呗垃圾回收的活动

ThreadLocal：线程隔离   线程私有的容器  应用：spring里面与数据库连接池建立connection操作 @transactional

B+树

中间节点存储的是指针，指向子节点的最大值或者最小值，叶节点存储具体的值

红黑树

红节点的子节点只能为黑节点，每个点到叶子结点的黑色节点树相同

应用：TreeMap、TreeSet、HashMap

##### Hashmap底层原理

数组：查询快、插入删除慢

链表：查询慢、插入删除快

1.7 数组+链表 1.8数组 + (链表 | 红黑树) 防DOS攻击

链表超过cap*factor（加载因子）数组扩容，扩容至64超8树化

原因：红黑树自平衡，到底层的距离都相同，相比AVL树旋转次数较少，每个分支的开销都是一样的

红黑树 红节点的子节点都是黑节点，叶子结点都是黑的（可为空）

多线程会出现：扩容死链（1.7）；数据错乱（1.7，1.8）

因为头插会造成依赖循环；

![image-20230315134642149](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134642149.png)

![image-20230315134648886](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134648886.png)

ArrayList初始容量为0，添加一个后变成10，之后扩容均为1.5倍 n + n>>1

addAll()扩容时选max{下次扩容，源list + 加添加的长度}
 FailFast不允许并发修改，即遍历时不能更改元素

FailSafe牺牲一致性可以遍历完 

![image-20230315134710895](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134710895.png)

##### ArrayList和LinkedList

动态数组——双向链表、查询快，插入删除慢——查询慢，插入删除快、全局操作——头部尾部操作

LinkedList只有头部插入快，其它均慢于ArrayList

Integer 8+4+4 包含锁信息和GCroot+方法区类对象+int

Serializable：序列化接口

Volatile：解决线程安全的可见性和有序性

单例模式：单例对象不能重复创建，在默认构造方法中实力不为空时抛出异常

破坏单例的方法：反射破坏、反序列化破坏、unsafe破坏

一般在JDK中使用：Runtime、Collections

1. 饿汉式：提前创建实例

2. 枚举饿汉式：枚举是一个特殊的类，枚举的变量是类中的公有静态变量

3. 懒汉式：在getInstance()时才创建对象，且加锁synchronized实现单例，放置其他线程调用

4. DCL（双检索）懒汉式：双次检查是否创建实例，防止多次创建，且只在开始时加锁

5. 内部类懒汉式

##### java栈溢出

形成原因：

1. **方法调用层级过多**：当方法调用的层级过多时，会导致栈空间不足，从而出现栈溢出。这种情况通常是由递归调用或者循环调用造成的。
2. **局部变量过多**：当方法中定义的局部变量过多时，会占用大量的栈空间，从而导致栈溢出。这种情况通常是由复杂的算法或者方法嵌套造成的。
3. **线程过多**：当系统中创建的线程过多时，每个线程都会占用一定的栈空间，从而导致栈空间不足，出现栈溢出。
4. **过多的方法参数**：当方法的参数过多时，会占用大量的栈空间，从而导致栈溢出。

解决方法：

1. **优化递归算法**：尽量避免使用过深的递归算法，可以通过循环或者迭代等方式来实现。
2. **增大方法调用栈的大小**：可以通过设置JVM参数-Xss来增大方法调用栈的大小，但是需要注意不要设置过大，否则可能会影响系统的性能。
3. **减少方法调用层级**：可以通过减少方法调用层级来避免StackOverflowError异常，比如可以将多个方法合并为一个方法，避免过多的方法调用。

##### 线程和进程

线程属于进程的一个部分，进程是一个具体运行的程序，线程是进程中的一个执行单元

每个进程都会有独立的空间，多个线程共享空间和数据

进程崩溃后不会影响其他进程，线程崩溃后整个进程都会死

##### 如何预防死锁

死锁：A占有资源1，同时申请资源2；而B占有资源2，同时申请资源1；

1、互斥条件

2、请求和保持条件：阻塞时释放已经获得的资源

3、不剥夺条件：不能获得全部资源则等待

4、环路等待条件：资源编号，进程获得编号小的资源才能获取到大的

##### 事物隔离级别

![image-20230315134722426](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134722426.png)

重载：不改变函数名、改名参数类型实现函数

##### 造成线程不安全的原因：

1.操作系统的随机调度/抢占式执行(万恶之源)-->无法改变

2.多个线程修改同一个变量(一个字都不能少)--->尽量避免

3.有些修改操作,不是原子的!(不可拆分的最小单位,就叫原子 即对应一条机器指令)--->通过加锁操作,把指令打包

4.内存可见性问题(内存改了,但是在优化的背景下,读不到,看不见)

##### 线程间通讯

1. **等待/通知机制**：通过wait()、notify()和notifyAll()三个方法实现。一个线程可以调用对象的wait()方法使自己进入等待状态，同时释放对象的锁。另一个线程可以调用对象的notify()或notifyAll()方法唤醒等待的线程。
2. **管道通信**：通过管道输入流和管道输出流进行线程间通信。一个线程可以通过管道输出流向管道输入流写入数据，另一个线程可以通过管道输入流从管道输出流读取数据。
3. **共享内存**：多个线程共享同一块内存区域，通过读写同一块内存区域实现线程间通信。需要注意的是，由于多个线程同时读写同一块内存区域可能会导致数据不一致的问题，因此需要采取一些同步措施，比如使用锁或者volatile关键字。
4. **信号量**：通过信号量来控制多个线程的并发访问数量。一个线程在访问某个资源时需要先获取信号量，如果信号量的计数器为0，则线程需要等待；如果信号量的计数器大于0，则线程可以获取信号量并继续执行，同时信号量的计数器会减1。当线程释放资源时，需要将信号量的计数器加1。

**线程不安全**：在随机调度之下,线程执行有多种可能,其中某些可能会导致代码出bug就称为线程不安全.

抽象类可以有默认的方法实现完全是抽象的。接口根本不存在方法的实现

抽象类和抽象方法

抽象类不能被实例化，必须要被继承，且必须要子类实现父类中的抽象方法

接口支持多继承，一个接口可以extends多个接口

常量：public static final

![image-20230315134731214](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134731214.png)

#### JAVA并发

并行：多个任务同时运行

并发：线程利用调度算法轮流执行

![image-20230315134809941](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134809941.png)

Runable是FunctionInterface（因为只有一个抽象方法）可以使用lamada

**创建新线程：**1、直接重写run方法 2、写一个Runable，实现run方法替换原有的run()

​           3、FutureTask可以将线程运行结果传递给其他的线程

线程状态：新建、可运行：新建后start、阻塞：未获取临界资源时、等待：获取到临界资源但条件不满足时、有时限等待用sleep

##### 线程池

线程池是一种用于管理和复用线程的机制，它可以在程序启动时创建一定数量的线程，并将这些线程保存在一个池中，当需要执行任务时，从池中取出一个线程来执行任务，任务执行完成后，线程并不会被销毁，而是返回到池中等待下一次执行任务。

使用线程池的主要目的是提高程序的性能和稳定性，具体来说，它有以下几个好处：

1. 降低线程创建和销毁的开销：线程的创建和销毁是比较耗费系统资源的操作，如果每次执行任务都要创建一个新线程，会导致系统开销较大。使用线程池可以避免这个问题，因为线程池中的线程可以被复用，从而降低了线程创建和销毁的开销。
2. 提高程序的响应速度：线程池中的线程可以立即执行任务，而不需要等待线程的创建和启动，从而提高了程序的响应速度。
3. 提高系统的稳定性：线程池可以限制系统中的并发线程数量，避免因线程过多导致系统资源耗尽的情况发生。同时，线程池可以对线程进行统一的管理和监控，例如线程的状态、执行时间等信息，从而提高了系统的稳定性和可维护性。

##### 线程池的核心参数：

corePoolSize：核心线程数目   maximumPoolSize：最大线程数目  keepAliveTime：等待时间  unit：时间单位    workQueue：阻塞队列    threadFactory：线程工厂  handler：拒绝策略（使用线程还是剔除线程）

wait和sleep：LOCK.wait() Thread.sleep()

![image-20230315134743770](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134743770.png)

**Lock和synchronized**：都是悲观锁

![image-20230315134757878](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134757878.png)

![image-20230315134847318](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134847318.png)

##### volatile关键字

1. 可见性：当一个线程修改了volatile变量的值时，其他线程可以立即看到这个修改。这是因为volatile变量的值会被写入到主内存中，其他线程读取该变量时会从主内存中读取最新的值。而普通变量则可能会存在线程间不可见的情况，即一个线程修改了变量的值，但其他线程并不能立即看到这个修改。
2. 有序性：当一个线程对volatile变量进行修改时，JVM会保证指令重排不会对该变量的读写顺序产生影响。这是因为JVM会在生成指令序列时，将volatile变量的读写指令插入到其他指令之间，以保证读写顺序的正确性。而普通变量则可能会存在指令重排的情况，即JVM为了提高程序性能，可能会对指令的执行顺序进行优化，这可能会导致变量的读写顺序不同于程序中的顺序。

Volatile能否保证线程安全：不能，只能保证可见性和有序性（摆烂器，不优化）

##### 悲观锁和乐观锁

悲观锁：顾名思义，就是比较悲观的锁，总是假设最坏的情况，每次去拿数据的时候都认为别人会修改，所以每次在拿数据的时候都会上锁，这样别人想拿这个数据就会阻塞直到它拿到锁。

乐观锁：总是假设最好的情况，每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这个数据，可以使用版本号机制和CAS算法实现。

![image-20230315134855626](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134855626.png)

保证原子性：悲观锁——上锁，乐观锁——新旧值比较

HashTable和concurrentHashMap

![image-20230315134902184](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134902184.png)

ThreadLocal可以实现【资源对象】的线程隔离，线程间隔离，线程内共享

##### 对ThreadLocal的理解

![image-20230315134910615](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134910615.png)

###### 应用场景：

1. 数据库连接管理：在Web应用中，每个请求都需要从数据库中获取数据，因此需要使用连接池来管理数据库连接。使用ThreadLocal可以为每个线程提供一个独立的数据库连接，从而避免了线程安全的问题。
2. Session管理：在Web应用中，每个用户都有一个独立的Session对象，用于保存用户的登录状态等信息。使用ThreadLocal可以为每个线程提供一个独立的Session对象，从而避免了线程安全的问题。
3. 日志管理：在多线程环境下，如果多个线程共享同一个日志对象，那么就需要考虑线程安全的问题。使用ThreadLocal可以为每个线程提供一个独立的日志对象，从而避免了线程安全的问题。
4. 数据缓存：在一些计算密集型的应用中，如果多个线程共享同一个数据缓存对象，那么就需要考虑线程安全的问题。使用ThreadLocal可以为每个线程提供一个独立的数据缓存对象，从而避免了线程安全的问题。

##### 类加载：

#### ![image-20230315134917323](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134917323.png)

![image-20230315134925890](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134925890.png)

可达性分析算法：要被Gcroot引用的不可回收，其他可回收

![image-20230315134933798](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134933798.png)

分代：年轻代、老年代、方法区

G1 GC：多线程、高并发、低暂停

##### 垃圾回收算法

标记清除法：标记后直接删除

标记整理法：标记后将所有可用的放一起，剩下的直接删除

Minor gc和major gc的区别

![image-20230315134941027](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134941027.png)

##### 双亲委派机制：把请求交给父类处理

双亲委派机制是 Java 类加载器的一种工作机制，它是一种层次化的加载器结构，由多个类加载器组成。当一个类加载器需要加载一个类时，它会先将这个任务委派给它的父类加载器去完成，如果父类加载器还存在父类加载器，那么这个任务会继续向上委派，直到最顶层的启动类加载器。只有当父类加载器无法完成这个任务时，子类加载器才会尝试自己去加载这个类。

* 如果一个类加载器收到了类加载请求，它并不会自己先加载，而是把这个请求委托给父类的加载器去执行
* 如果父类加载器还存在其父类加载器，则进一步向上委托，依次递归，请求最终将到达顶层的引导类加载器；
* 如果父类加载器可以完成类加载任务，就成功返回，倘若父类加载器无法完成加载任务，子加载器才会尝试自己去加载，这就是双亲委派机制
* 父类加载器一层一层往下分配任务，如果子类加载器能加载，则加载此类，如果将加载任务分配至系统类加载器也无法加载此类，则抛出异常

JVM提供了三层ClassLoader（类加载器）：

* Bootstrap classLoader:主要负责加载核心的类库(java.lang.*等)，构造ExtClassLoader和APPClassLoader。
* ExtClassLoader：主要负责加载jre/lib/ext目录下的一些扩展的jar。
* AppClassLoader：主要负责加载应用程序的主函数类

如果未加载过且不能加载就回到父类加载器

该机制的好处

![img](https://img-blog.csdnimg.cn/2020121722082798.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NvZGV5YW5iYW8=,size_16,color_FFFFFF,t_70)

##### Tomcat类加载机制

Tomcat 本身也是一个 java 项目，因此其也需要被 JDK 的类加载机制加载，也就必然存在 引导类加载器、扩展类加载器和应用(系统)类加载器。 Common ClassLoader 作为 Catalina ClassLoader 和 Shared ClassLoader 的 parent，而 Shared ClassLoader 又可能存在多个 children 类加载器 WebApp ClassLoader，一个 WebApp ClassLoader 实际上就对应一个 Web 应用，那 Web 应用就有可能存在 Jsp 页 面，这些 Jsp 页面最终会转成 class 类被加载，因此也需要一个 Jsp 的类加载器。 需要注意的是，在代码层面 Catalina ClassLoader、Shared ClassLoader、Common ClassLoader 对应的实体类实际上都是 URLClassLoader 或者 SecureClassLoader，一般 我们只是根据加载内容的不同和加载父子顺序的关系，在逻辑上划分为这三个类加载器； 而 WebApp ClassLoader 和 JasperLoader 都是存在对应的类加载器类的。 

当 tomcat 启动时，会创建几种类加载器： 

1 Bootstrap 引导类加载器 加载 JVM 启动所需的类，以及标准扩展类（位于 jre/lib/ext 下） 

2 System 系统类加载器 加载 tomcat 启动的类，比如 bootstrap.jar，通常在 catalina.bat 或者 catalina.sh 中指定。位于 CATALINA_HOME/bin 下。 

3 Common 通用类加载器 加载 tomcat 使用以及应用通用的一些类，位于 CATALINA_HOME/lib 下，比如 servlet-api.jar 

4 webapp 应用类加载器每个应用在部署后，都会创建一个唯一的类加载器。该类加载器 会加载位于 WEB-INF/lib 下的 jar 文件中的 class 和 WEB-INF/classes 下的 class 文件。

![img](https://images0.cnblogs.com/blog2015/449064/201506/141304597074685.jpg)

##### JVM内存模型

![9ce607e1-d976-41d4-89ad-dc9a08adc93b](file:///C:/Users/hua'wei/Pictures/Typedown/9ce607e1-d976-41d4-89ad-dc9a08adc93b.png)

线程共享：

* 方法区：虚拟机加载的类信息、常量、静态变量、JIT编译后的代码
* 堆区：存储着所有的实例对象（在堆区发生垃圾回收）

线程私有：

* 本地方法栈：本地方法栈和虚拟机栈类似，只不过本地方法栈为虚拟机使用本地方法（native）服务。
* 虚拟机栈：是栈帧，用于存储局部变量表、操作数栈、动态链接、方法出口等信息
* 程序计数器；当前进程以及进程队列

java的引用以及它所指向的数据存储在哪

引用：存储在堆区

数据：方法区

##### MYSQL——锁

锁：用于管理对共享资源的并发访问

lock（闩锁）：要求锁定的时间必须非常短，持续时间长性能会很差

latch：

* mutex互斥锁：
* rwLock读写锁：

![img](https://ask.qcloudimg.com/http-save/6869393/01e2iffwxd.jpeg?imageView2/2/w/1620)

锁的类型

* 共享锁（S Lock）：允许事务读一行数据
* 排它锁（X Lock）：允许事物删除或者更新一行数据
* 记录锁（Record Lock）：仅锁定一行记录（如固定）
* 间隙锁（Gap Lock）：锁定一个范围，但不包含记录本身
* 临键锁：锁定一个范围，并且锁定记录本身

##### 怎么把Mapper接口注册到Spring中

![image-20221109194232265](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20221109194232265.png)

##### Mysql的索引

B+树索引：非叶子节点存储键值，叶子结点存储具体的数据

Hash索引：哈希码、指针，但是数据量大哈希冲突严重。

全文索引：用于模糊查询的，大量数据查询比like快N倍

*Mysql数据和索引都是存在磁盘等外部设备的

##### Mysql存储引擎

* innoDB：支持acid的存储引擎
* MyISAM：具有较高的插入、查询速度，但没有事务
* MEMORY：hash索引

#### JVM

###### 隐式加载：指的是程序在使 用 new 等方式创建对象时，会隐式地调用类的加载器把对应的类 加载到 JVM 中。

###### 显式加载：指的是通过直接调用 class.forName() 方法来把所需的类加载到 JVM 中。

##### 类加载器

1. **Bootstrap Class Loader（启动类加载器）**：它是Java虚拟机（JVM）的一部分，负责加载JVM运行所需的核心类，如Java标准库（rt.jar）等。这个类加载器由C++实现，不是Java类加载器，它位于JVM的内部，无法在Java代码中直接访问。
2. **Extension Class Loader（扩展类加载器）**：它负责加载Java扩展类库（如JAR包中的扩展库）的类。扩展类加载器的父类加载器是启动类加载器。
3. **Application Class Loader（应用类加载器）**：它也被称为系统类加载器，负责加载应用程序classpath下的类。这是大多数Java应用程序默认的类加载器，它的父类加载器是扩展类加载器。

##### 垃圾收集器

###### 新生代

> Serial收集器：只能进行一个线程的垃圾回收，在收集时所有工作线程都停止工作，使用标记复制算法
>
> ParNew收集器：是Serial收集器的多线程版本，使用标记复制算法
>
> Parallel Scavenge收集器：多线程收集器，关注控制系统运行的吞吐量

###### 老年代

> SerialOld收集器：老年代的Serial，负责标记整理
>
> Parallel Old收集器：老年代的Parallel，也是标记整理
>
> CMS收集器：标记整理-并发收集器

###### G1收集器：初始标记，并发标记，复制标记，复制清除

##### JVM常用工具

###### jps：显示本地的Java进程，可以查看本地运行着几个Java程序

###### jinfo：Java运行时的环境参数

###### jstack：可以观察到JVM当前所有进程的运行状态和线程当前状态

###### jmap：可以查看JVM当前的物理内存占用状态

##### 内存结构

![image-20221109204749473](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20221109204749473.png)

方法区、永久代、元空间之间的关系

![image-20221110093134984](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20221110093134984.png)

##### JVM内存参数

- -Xmx：表示虚拟机最大容量
- -Xms：表示虚拟机最小容量
- -Xmn：表示虚拟机新生代容量
- -XX:SurvivorRatio：eden源与from区之比（新生代分为eden源、from区和to区）

##### 垃圾回收算法

* 标记清除法：标记有指向的，清除没有指向的
- 标记整理法：整理有指向的，比较适合老年代

- 标记复制法：向另一块区域直接复制，比较适合新生代

GC和分代回收算法

* GC目的：在于实现无用对象内存自动释放，减少内存碎片

* GC要点：![image-20221110100006642](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20221110100006642.png)

* 分代回收：![image-20221110101326822](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20221110101326822.png)
- GC规模：![image-20221110101704544](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20221110101704544.png)

三色标记法：黑色已标记（其与下属均被标记）、灰色标记中（下属还未标记）、白色还未标记

并发漏标问题：增量更新（记录被赋值元素）、原始快照（都被记录）

##### 垃圾回收器

![image-20221110102929610](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20221110102929610.png)

内存溢出：误用固定大小的线程池、误用带缓冲的线程池、一次查询太多、类太多

##### finalize的理解

![image-20221110121131127](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20221110121131127.png)

![image-20221110135327732](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20221110135327732.png)

##### Zset的使用场景

需要排序的场景，比如top10热点文章，或者排行榜

消息的延迟发送，判断时间进行发送

##### 设计模式

单例模式：只有一个实例，主要解决一个全局使用的类频繁地创建与销毁。

工厂模式：在工厂模式中，我们在创建对象时不会对客户端暴露创建逻辑，并且是通过使用一个共同的接口来指向新创建的对象。

代理模式：我们创建具有现有对象的对象，以便向外界提供功能接口。

#### JUC

JUC指的是Java Util Concurrent，它是Java并发编程中的工具类库。JUC提供了一系列的工具类和接口，用于辅助Java程序实现高效的并发编程。

作用：

> 1. 提供了一系列的线程安全的工具类和接口，如锁、原子类、线程池、并发集合等，可以帮助Java程序员更加轻松地实现高效、安全的并发编程。
> 2. 解决了Java早期版本中并发编程的一些问题，例如死锁、线程安全问题等。
> 3. 提高了Java程序的并发性能和效率。
> 4. 促进了Java平台的并发编程的发展和推广。

##### JUC线程池

Java中的线程池是一种线程管理机制，可以帮助我们更好地管理线程，避免线程过多导致系统资源耗尽的问题。线程池中会维护一定数量的线程，当需要执行任务时，会从线程池中取出一个线程来执行任务，任务执行完毕后，该线程会被放回线程池中，等待下一次任务的执行。

Java中的线程池主要由以下几个组成部分：

1. **线程池管理器（ThreadPoolExecutor）**：线程池管理器负责管理线程池中的线程，它会根据需要创建新的线程或销毁不需要的线程。
2. **任务队列（BlockingQueue）**：任务队列用于存储等待执行的任务，当线程池中的所有线程都在执行任务时，新的任务会被放入任务队列中等待执行。
3. **工作线程（Worker Thread）**：工作线程是线程池中的实际执行任务的线程。

##### CAS可以解决什么问题

CAS（Compare And Swap）是一种轻量级的同步机制，用于解决多线程环境下的并发问题。CAS操作包含三个操作数：内存位置（V）、期望值（A）和新值（B）。当且仅当V的值等于A时，CAS才会通过原子方式用新值B来更新V的值，否则不会执行任何操作。

CAS操作可以解决以下两种并发问题：

1. 针对共享变量的原子操作：在多线程环境下，多个线程可能同时对共享变量进行读取和写入操作，导致数据不一致的问题。使用CAS操作可以保证对共享变量的操作是原子的，从而避免了数据不一致的问题。
2. 针对锁的竞争问题：在多线程环境下，多个线程可能同时竞争同一个锁，导致线程阻塞和性能下降的问题。使用CAS操作可以实现非阻塞算法，避免了锁的竞争问题，从而提高了程序的性能

##### NIO和AIO

AIO和NIO都是Java中的I/O模型，不同之处在于它们的工作方式和底层实现。

NIO（Non-blocking I/O）是Java 1.4引入的一种I/O模型，它使用单线程轮询的方式实现多路复用，可以同时处理多个客户端连接。在NIO中，当一个连接有数据可读时，会通知线程进行读取操作，如果没有数据可读，则线程可以继续处理其他连接。NIO的主要特点是非阻塞式的I/O操作，可以提高系统的吞吐量和并发性能。

AIO（Asynchronous I/O）是Java 1.7引入的一种I/O模型，它使用操作系统提供的异步I/O机制，通过回调函数的方式实现数据的处理。在AIO中，当一个连接有数据可读时，操作系统会通知Java应用程序进行读取操作，Java应用程序只需要等待通知即可，不需要进行轮询操作。AIO的主要特点是异步I/O操作，可以提高系统的响应速度和资源利用率。

在底层实现上，NIO使用了Java的Selector类实现多路复用，而AIO使用了操作系统提供的异步I/O机制。NIO的缺点是在高并发情况下，单线程处理多个连接的效率可能会降低，而AIO则可以更好地处理高并发情况下的I/O操作。

总之，NIO适用于连接数较少、但数据处理量较大的场景，而AIO适用于连接数较多、但数据处理量较小的场景。

##### Java自动拆箱装箱和拆箱

Java中的自动装箱和拆箱是指Java编译器自动将基本类型和对应的包装类型进行转换的过程。具体来说，当需要使用包装类型的对象时，可以直接使用基本类型的值，编译器会自动将其转换为对应的包装类型；当需要使用基本类型的值时，可以直接使用包装类型的对象，编译器会自动将其转换为对应的基本类型。

自动装箱和拆箱的优势主要有以下几点：

1. **简化代码**：使用自动装箱和拆箱可以避免手动进行基本类型和包装类型之间的转换，从而简化代码，提高代码的可读性和可维护性。
2. **提高性能**：使用自动装箱和拆箱可以避免频繁的基本类型和包装类型之间的转换，从而提高程序的性能。
3. **支持泛型**：使用自动装箱和拆箱可以方便地支持泛型，因为泛型只能接受对象类型，而基本类型不是对象类型。使用自动装箱和拆箱可以将基本类型转换为对应的包装类型，从而方便地使用泛型。

