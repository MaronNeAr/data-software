## Ssm框架+springboot

#### Spring：

核心jar：beans、core、context、expression、aop

@transational：注解里面的各个属性和咱们在上面讲的事务属性里面是一一对应的。用来设置事务的传播行为、隔离规则、回滚规则、事务超时、是否只读。

两种IOC容器：BeanFactory&ApplicationContext

BeanFactory：最基本的IOC容器 特点：加载配置文件不会创建对象 使用对象（getBean）时才创建对象

##### Bean作用域

1. 单例(Singleton)作用域：在整个应用程序中只创建一个Bean实例。每次请求该Bean时，都会返回同一个实例。
2. 原型(Prototype)作用域：每次请求该Bean时，都会创建一个新的实例。
3. 会话(Session)作用域：在Web应用程序中，每个会话都有一个对应的Bean实例。当用户与应用程序交互时，Bean实例将保持活动状态，直到会话结束。
4. 请求(Request)作用域：在Web应用程序中，每个HTTP请求都有一个对应的Bean实例。当请求完成时，Bean实例将被销毁。

##### Bean生命周期：

![image-20230315134237864](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134237864.png)                     

ApplicationContext：BeanFactory的子接口 特点：加载时就创建对象

 ![image-20230315134250935](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134250935.png)

IOC：将对象创建以及对象的互相调用过程交给spring管理，降低耦合度（工厂模式）

底层原理：1.xml解析/注解 2. 工厂模式（IOC容器） 3.反射

反射：通过Class.forName(bean标签class属性值)创建新实例

Bean管理：对象创建 注入属性

Bean管理两种方式：xml、注解

XML：  Bean标签属性：id 唯一标识 class 类全路径  

​       依赖注入DI：使用set注入 使用有参构造注入 

​       使用property注入<property name=”bookname” value=”书名”></property>

​        有参注入<constructor-arg name=”bookname” value=”书名”></ constructor-arg>

​       注入空值：<null/ > 特殊符号<![CDATA[<<字符串>>]]>

​       嵌套bean 内部、外部     数组用<array>   List用<list>  Map用<map>

IOC bean管理：

bean：为一个精心配置后的实例化对象

普通bean：配置文件中bean的类型就是返回类型

工厂bean：定义和返回类型可以不一样 实现FactoryBean接口以确定返回类型（泛型工厂模式）

Bean作用域 ：默认单实例对象 scope：singleton（单，加载即创建） prototype（多，getBean时才创建对象）request、session：创建对象的存放位置

Bean生命周期：通过构造器创建bean实例（无参构造）->为bean的属性设置值及其他bean的调用->初始化前将实例传给后置处理器->调用初始化的方法->初始化后将实例传给后置处理器->使用实例化对象->容器关闭，调用销毁方法(多实例对象app.close()时不销毁)

自动装配：byName：bean的id值需要和属性名一样 byType

##### Bean的生命周期及其作用域

1. Bean的生命周期：Bean的生命周期包括实例化、属性赋值、初始化、使用和销毁等阶段。Spring容器通过BeanFactory和ApplicationContext接口来管理Bean的生命周期。
2. Bean的作用域：Bean的作用域决定了Bean的生命周期和可见范围。Spring提供了多种作用域，包括Singleton、Prototype、Request、Session、Global Session和Application等。

##### BeanFactory和FactoryBean

BeanFactory是Spring框架中最基本的容器，它负责管理Bean的生命周期和依赖注入。FactoryBean是一种特殊的Bean，它的作用是创建和管理其他Bean，它本身也是一个Bean。FactoryBean接口定义了两个方法：getObject()和getObjectType()，分别用于获取FactoryBean创建的对象和对象类型。

##### Spring注解：

自动装配、set注入、构造器注入

使用Set注入：注解标注方法，参数为注入的bean，添加this

注解方式实现属性注入：@Autowired 根据类型注入

​                   @Qualifier（value = “”） 根据属性名称注入

​                   @Resource 不带括号类型 带括号名称  

​                   @Value

1、@Controller：用于标注控制器层组件

2、@Service：用于标注业务层组件

3、@Component : 用于标注这是一个受 Spring 管理的组件，组件引用名称是类名，第一个字母小写。可以使用@Component(“beanID”) 指定组件的名称

4、@Repository：用于标注数据访问组件，即DAO组件

5、 @Bean：方法级别的注解，主要用在@Configuration和@Component注解的类里，@Bean注解的方法会产生一个Bean对象，该对象由Spring管理并放到IoC容器中。引用名称是方法名，也可以用@Bean(name = "beanID")指定组件名

6、@Scope("prototype")：将组件的范围设置为原型的（即多例）。保证每一个请求有一个单独的action来处理，避免action的线程问题。

由于Spring默认是单例的，只会创建一个action对象，每次访问都是同一个对象，容易产生并发问题，数据不安全。

7、@Autowired：默认按类型进行自动装配。在容器查找匹配的Bean，当有且仅有一个匹配的Bean时，Spring将其注入@Autowired标注的变量中。

8、@Resource：默认按名称进行自动装配，当找不到与名称匹配的Bean时会按类型装配。

9、@transcational：声明式事务，所有方法异常时自动进行回滚操作。

 ![image-20230315134301227](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134301227.png)

 

使用ApplicationContext容器读取xml配置，从xml配置中getbean（）实例化。

 ![image-20230315134310680](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134310680.png)

 

##### AOP:不改变原始代码添加功能

底层原理：两种动态代理

有接口使用JDK动态代理：创建接口实现类的代理对象增强类的方法

没有接口使用CGLIB动态代理：（原始）创建子类重写方法以增强类的方法

​                         （CGLIB代理）创建子类的代理对象，增强类的方法

NewProxyInstance()方法：

（参数）类加载器、增强方法的类要实现的接口、创建对象，增加方法

##### AOP术语：

连接点：类中可以增强的方法         切入点：实际增强的方法

通知（增强）：实际增强的逻辑的部分（前置、后置、环绕、异常catch、最终finally）

切面：通知引用到切入点的过程

##### AspectJ：

切入点表达式：excution([权限修饰符][返回类型][类全路径][方法名称][参数列表])

事物：逻辑上的一组操作，数据库操作的最基本的单元

原子性：出现异常，事物回滚，撤回已经执行的sql语句

##### 声明式事务管理（底层为AOP）： 

PlatformTransactionManager：

Propagation：事物的传播行为（一个事物被另一个事物调用）

 ![image-20230315134328741](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134328741.png)

脏读：一个未提交事物读取到连一个未提交事物的数据

不可重复读：未提交事物提取到提交事物的修改的数据

虚读：未提交事物提取到提交事物的添加的数据

（解决）设置事物隔离级 别

  ![image-20230315134342568](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134342568.png)
 timeout：超时不提交就回滚

 ![image-20230315134351418](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134351418.png)

##### Spring5新特性：

 ![image-20230315134358034](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134358034.png)

 

#### SpringMVC：

 ![image-20230315134405883](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134405883.png)

RequestMapping：请求路径   逐级嵌套

   ![image-20230315134412710](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134412710.png)

报错：  404 没有path与之对应、请求头不匹配headers 

400 参数不匹配params 必需的参数未传输

​       405 请求方式不匹配method

SpringMVC 可以同名自动装配参数

@PathVariable("id")取占位符里面的内容 {id}

@RequestParam(“username”) 取出请求参数作为形参

required 是否必需 defaultValue 默认值（不用管required）

@RequestHeader() 请求头，同上   @Cookie() cookie，同上

Request setCharacterEncoding(“UTF-8) 设置字符集编码

域对象共享数据：使用servletAPI、ModelAndView、Model、map、ModelMap向request域对象共享数据（继承自BindingAwareModelMap） 利用addAttribute方法

向session、application域中共享数据 利用setAttribute方法

转发视图：intenalResourceView 引入jstl的依赖后自动转换为jstlView

​    若使用thymeleaf解析，则视图为ThymeleafView

forward：转发视图       地址栏为请求地址

redirect：重定向视图     地址栏为跳转页面的地址

Restful：表现层资源状态转移  

使用put&delete方法：表单内部设置隐藏域type=”hidden” name=”_method” value=”request_method”

HttpMessageConvert：请求报文

@RequestBody：将请求报文中的请求体转换为java对象

@ResponseBody：将java对象转换为响应体 RequestEntity<String> 获取整个请求报文

@RestController 为类添加Controller注解 + 为每一个方法添加 @ResponseBody方法

MultipartFile 上传文件的打包数据类型

拦截器interceptors 继承自HandlerInterceptors

过滤器作用于浏览器到DispatcherServlet之间    controller == handler

拦截器用来拦截控制器中的方法    控制器之前、之后、渲染页面之后

preHandle按配置顺序执行，postHandle以及afterCompletion按反序执行

多个拦截器有一个执行后，该拦截器之前的afterCompletion都执行，后面都不执行，并且所有的postHandle都不执行

 ![image-20230315134423078](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134423078.png)

 ![image-20230315134429939](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134429939.png)

##### SprinMVC生命周期：

首先查看是否开放静态资源的配置，如果开放的话导入Handle对象、拦截器、模板引擎等，然后在Handle里面的方法执行之前对用拦截器中的preHandle方法，如果其返回值为true，继续执行Handle里面的方法，然后执行poetHandle方法，接着将返回的ModelAndView通过视图解析器进行解析，之后调用afterCompetition方法，最后将render的结果返回给客户端。

![image-20230315134458300](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134458300.png) 

#### Mybatis：

支持sql数据库查询、存储和高级映射的框架 包括用户api接口以及数据处理

数据库驱动：com.mysql.jdbc.Driver url：jdbc:mysql://localhost:3306/userDB

 ![image-20230315134439467](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134439467.png)

ORM：关系型对象映射模型

 ![image-20230315134447742](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134447742.png)

表--实体类--mapper--映射文件:一一对应

获取SqlSession（数据库一级 缓存）: 

SqlSession sqlSession = ((new SqlSessionFactoryBuilder().build(is))).openSession();

以JDBC方式进行增删改查需要提交事物

${} 字符串拼接（需要单引号）    #{}占位符赋值  

多参数使用arg0、arg1、param1、param2 @param（“key”）Object value 覆盖了arg

只能使用${}  批量查询、按表查询

开启懒加载后，不需要多表查询时不用去加载另一个mapper   并设置需要多表查询的mapper的fetchType属性为立即加载

多对一用javaType    一对多用ofType 

 ![image-20230315134520062](C:\Users\hua'wei\AppData\Roaming\Typora\typora-user-images\image-20230315134520062.png)

SqlSessionFactory：二级缓存

逆向工程：根据数据库字段由框架生成mapper、实体类

#### Springboot:

Domain：写实体类的

Dao层（也叫MP层）：数据持久层，一般只配置MP即可

Controller层：控制层，用于处理来自前端的用户请求

Service层：业务层，主要用于实现业务逻辑

主要的依赖：spring-boot-starter、mybatis-plus-startwe、mysql、druid等

Yml配置：写数据库驱动、mysql连接端口、连接用户名、用户密码、Mp配置（指明数据库、显示日志等）