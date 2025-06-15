## 第一章 Spring架构

#### Spring整体架构与模块划分

###### 核心容器（Core Container）

- **spring-core**
  - **基础工具类**：如资源加载（`Resource`接口）、反射工具（`ReflectionUtils`）、类型转换（`ConversionService`）。
  - **核心接口**：`BeanFactory`（IoC容器的基本定义）。
- **spring-beans**
  - **Bean的定义与依赖管理**：`BeanDefinition`（描述Bean的元数据）、`BeanWrapper`（Bean属性操作）。
  - **Bean的实例化与生命周期管理**：`AbstractAutowireCapableBeanFactory`（Bean创建的核心类）。
- **spring-context**
  - **应用上下文**：`ApplicationContext`（扩展自`BeanFactory`，提供国际化、事件发布、资源加载等高级功能）。
  - **核心实现类**：`ClassPathXmlApplicationContext`（XML配置的上下文）、`AnnotationConfigApplicationContext`（注解驱动的上下文）。
- **spring-expression（SpEL）**：Spring表达式语言，支持在运行时查询和操作对象图，如`@Value("#{systemProperties}")`。

###### AOP与Instrumentation

- **spring-aop**：动态代理，基于JDK动态代理或CGLIB生成AOP代理对象，如`AopProxy`（代理对象生成器）、`Advisor`（切面逻辑）。
- **spring-aspects**：集成AspectJ，支持`@AspectJ`注解风格的切面定义。
- **spring-instrument**：类加载器增强，用于服务器级的类植入（如Tomcat的`InstrumentableClassLoader`）。

###### 数据访问与集成（Data Access/Integration）

- **spring-jdbc**：JDBC抽象层，`JdbcTemplate`简化数据库操作，避免手动处理连接和异常。
- **spring-tx**：事务管理，`PlatformTransactionManager`定义事务操作，支持声明式事务（`@Transactional`）。
- **spring-orm**：ORM框架整合，对Hibernate、JPA等的支持，如`HibernateTemplate`。
- **spring-oxm**：对象-XML映射，支持JAXB、XStream等，用于XML与Java对象的转换。
- **spring-jms**：消息服务，简化JMS API的使用，如`JmsTemplate`。

###### Web层

- **spring-web**：基础Web功能，如HTTP客户端、Servlet监听器、`WebApplicationContext`（Web应用上下文）。
- **spring-webmvc**：MVC框架，如`DispatcherServlet`（前端控制器）、`@Controller`、`@RequestMapping`注解驱动开发。
- **spring-webflux**（Spring 5+）：响应式Web支持，基于Reactor库实现非阻塞式编程模型，核心类如`WebHandler`。
- **spring-websocket**：WebSocket通信，支持实时双向通信，如`WebSocketHandler`。

###### 其他模块

- **spring-test**：集成测试支持，`@SpringJUnitConfig`加载上下文，`MockMvc`模拟HTTP请求。
- **spring-messaging**：消息协议抽象，支持STOMP、WebSocket子协议，用于构建消息驱动的应用。
- **spring-context-support**：第三方库集成，如缓存（Ehcache）、邮件（JavaMail）、调度（Quartz）。
- **spring-framework-bom**：依赖管理，统一管理Spring模块的版本，避免Maven/Gradle依赖冲突。

## 第二章 IoC容器的设计与实现

#### BeanFactory与ApplicationContext的核心区别

###### 设计目标与定位

- **BeanFactory**
  - **定位**：IoC 容器的基础接口，提供 **最底层的容器功能**，是 Spring 框架的基石。
  - **设计目标**：实现 Bean 的 **定义、加载、实例化、依赖注入** 等核心功能；保持轻量级，关注容器的基础职责，不集成非必要功能。

- **ApplicationContext**
  - **定位**：`BeanFactory` 的扩展接口，是 Spring 的 **高级容器**，面向企业级应用。
  - **设计目标**：在 `BeanFactory` 基础上，集成 **事件发布、国际化、资源管理、AOP 支持** 等企业级功能，提供注解驱动。

###### 功能特性对比

- **BeanFactory 的核心功能**
  - **Bean 的实例化与依赖注入**:通过 `getBean()` 方法获取 Bean 实例；支持构造器注入（Constructor Injection）和 Setter 注入。
  - **Bean 生命周期管理**：支持 `init-method` 和 `destroy-method` 回调。
  - **层级容器（Hierarchical Containers）**：通过 `HierarchicalBeanFactory` 接口支持父子容器。

- **ApplicationContext 的扩展功能**

  - **事件发布机制（Event Publishing）**通过 `ApplicationEventPublisher` 接口发布事件（如 `ContextRefreshedEvent`）。

  - **国际化支持（MessageSource）**：通过 `MessageSource` 接口实现多语言资源加载。

  - **资源访问抽象（ResourceLoader）**：支持统一资源定位（如 `classpath:`、`file:`、`http:` 前缀）。

  - **环境配置（Environment Abstraction）**：管理 Profiles（`@Profile`）和属性文件（`@PropertySource`）。

  - **AOP 与事务集成**：自动注册 `AnnotationAwareAspectJAutoProxyCreator`；声明式事务的自动代理（`@Transactional`）。

  - **便捷的配置方式**：支持 XML、Java 注解（`@Component`、`@Configuration`）和 Java Config。

  - ```java
    public interface ApplicationContext extends 
        EnvironmentCapable,          // 环境配置
        ListableBeanFactory,         // 扩展 BeanFactory（支持按类型获取 Bean）
        HierarchicalBeanFactory,     // 层级容器
        MessageSource,               // 国际化
        ApplicationEventPublisher,   // 事件发布
        ResourcePatternResolver {    // 资源加载
    }

###### Bean 加载机制

- **BeanFactory**：延迟加载（Lazy Initialization），只有在调用 `getBean()` 时才会实例化 Bean。
- **ApplicationContext**：预加载单例 Bean，在容器启动时（`refresh()` 方法）完成所有非延迟单例 Bean 的实例化。

- **循环依赖处理**：通过 **三级缓存** 解决循环依赖，但需要开发者手动处理 `BeanPostProcessor` 的依赖关系。

  ```java
  private final Map<String, Object> singletonObjects = ...;       // 一级缓存（完整 Bean）
  private final Map<String, Object> earlySingletonObjects = ...;  // 二级缓存（早期引用）
  private final Map<String, ObjectFactory<?>> singletonFactories = ...; // 三级缓存（工厂对象）
  ```

###### 核心区别

| **对比项**    | **BeanFactory**                  | **ApplicationContext**             |
| :------------ | :------------------------------- | :--------------------------------- |
| **功能定位**  | 基础容器，仅提供 IoC/DI 核心功能 | 高级容器，集成企业级扩展功能       |
| **Bean 加载** | 延迟加载（按需实例化）           | 预加载单例 Bean（启动时完成）      |
| **扩展能力**  | 无事件、国际化、AOP 等支持       | 支持事件、国际化、资源抽象、AOP 等 |
| **配置方式**  | 仅支持 XML                       | 支持 XML、注解、Java Config        |
| **实际使用**  | 极少直接使用                     | Spring 应用的标准容器              |

#### 资源定位与Resource体系

###### Resource接口与实现类

- `ClassPathResource`：类路径下的资源
- `FileSystemResource`：文件系统资源
- `UrlResource`： URL资源（HTTP、FTP等）
- `ServletContextResource`：Web应用上下文资源
- `ByteArrayResource`：内存字节数组资源

```java
public interface Resource extends InputStreamSource {
    boolean exists();          // 资源是否存在
    boolean isReadable();      // 资源是否可读
    boolean isOpen();          // 资源是否为流形式（如网络资源）
    URL getURL() throws IOException;
    File getFile() throws IOException;
    String getDescription();   // 资源描述（如文件路径）
}
```

###### 资源定位的核心接口

- **ResourceLoader**：根据路径字符串（如`classpath:app.xml`）返回对应的`Resource`对象。
- **ResourcePatternResolver**：支持通配符（如`classpath*:config/*.xml`）匹配多个资源。
- **ApplicationContext与资源加载**：所有`ApplicationContext`均实现了`ResourceLoader`接口，可直接调用`getResource()`方法。

###### 资源定位流程（以XML配置为例）

- **构造方法传入配置文件路径**：`new ClassPathXmlApplicationContext("classpath:application.xml")`
- **解析路径为Resource数组**：使用`PathMatchingResourcePatternResolver`解析路径，生成`Resource[]`
- **加载并读取资源**：`XmlBeanDefinitionReader`读取`Resource`中的XML配置；解析Bean定义并注册到`BeanFactory`。

#### BeanDefinition的加载与解析

###### BeanDefinition的核心属性

- **Bean的类名**（`beanClassName`）
- **作用域**（`scope`，如`singleton`、`prototype`）
- **是否延迟加载**（`lazyInit`）
- **初始化/销毁方法**（`initMethodName`、`destroyMethodName`）
- **依赖关系**（通过构造函数参数或属性注入）
- **工厂方法**（`factoryMethodName`，用于静态工厂或实例工厂创建Bean）

###### XML配置的加载与解析流程

- **资源定位与读取**
  - **入口类**：`XmlBeanDefinitionReader`（负责读取XML文件并解析为BeanDefinition）。
  - **资源定位**：通过`ResourceLoader`（如`ClassPathResource`）加载XML文件。
  - **文档解析**：使用`DocumentLoader`将XML文件解析为`Document`对象（基于DOM或SAX）。
  - **BeanDefinition解析**：遍历Document中的元素（如`<bean>`标签），生成BeanDefinition。

- **XML解析的核心类**
  - **DefaultBeanDefinitionDocumentReader**：遍历XML文档中的根元素（`<beans>`）及其子元素（`<bean>`等）。
  - **BeanDefinitionParserDelegate**：具体解析每个`<bean>`标签，处理属性（如`id`、`class`、`scope`）和子元素。

###### 注解配置的加载与解析流程

- **组件扫描与注解处理器**
  - **入口类**：`ClassPathBeanDefinitionScanner`（负责扫描类路径下的注解类）。
  - **核心注解**：`@Component`（及其派生注解`@Service`等）、`@Configuration`、`@Bean`、`@Autowired`、`@Value`
- **组件扫描流程**
  - **配置扫描路径**：通过`@ComponentScan(basePackages = "com.example")`指定包路径。
  - **类路径扫描**：使用`ClassPathScanningCandidateComponentProvider`筛选候选类。
  - **生成BeanDefinition**：对带有`@Component`的类生成`ScannedGenericBeanDefinition`。
- **注解解析的核心类**
  - **AnnotatedBeanDefinitionReader**：处理`@Configuration`类中的`@Bean`方法，将其转换为`ConfigurationClassBeanDefinition`。
  - **AutowiredAnnotationBeanPostProcessor**：处理`@Autowired`和`@Value`注解，实现依赖注入。

#### Bean的实例化过程

###### Bean实例化整体流程

- **实例化（Instantiation）**：创建Bean的原始对象（通过构造函数或工厂方法）。
- **属性填充（Population）**：依赖注入（DI）及设置属性值。
- **初始化（Initialization）**：调用初始化方法（如`init-method`）及应用后置处理器。
- **销毁（Destruction）**（可选）：容器关闭时调用销毁方法。

###### 实例化阶段（Instantiation）

- **目标**：根据Bean定义创建Bean的原始对象。

- **默认构造函数**：无参构造函数（最常见方式）。

- **静态工厂方法**：通过`factory-method`指定静态方法。

- **实例工厂方法**：通过`factory-bean`和`factory-method`指定实例方法。

- **三级缓存机制**：

  - **一级缓存（singletonObjects）**：存放完全初始化的单例Bean。

  - **二级缓存（earlySingletonObjects）**：存放早期暴露的Bean（未完成属性填充）。

  - **三级缓存（singletonFactories）**：存放Bean的工厂对象，用于生成早期引用。

###### 属性填充（Population）

- **目标**：为Bean注入依赖的属性和值。

- **Setter注入**：通过`<property>`标签或`@Autowired`注解。
- **构造器注入**：通过`<constructor-arg>`标签或构造函数参数上的`@Autowired`。

- **自动装配（Autowiring）**

  - **按类型（byType）**：根据类型匹配候选Bean。

  - **按名称（byName）**：根据属性名匹配Bean名称。

  - **注解驱动**：通过`@Autowired`、`@Resource`或`@Inject`实现。

###### 初始化阶段（Initialization）

- **目标**：执行初始化逻辑，使Bean达到可用状态。
- **Aware接口回调**：调用`BeanNameAware.setBeanName()`、`BeanFactoryAware.setBeanFactory()`等。
- **BeanPostProcessor前置处理**：调用`postProcessBeforeInitialization()`（如`@PostConstruct`处理）。
- **自定义初始化方法**：调用`InitializingBean.afterPropertiesSet()`或XML中配置的`init-method`。
- **BeanPostProcessor后置处理**：调用`postProcessAfterInitialization()`（如AOP代理的生成）。
- **AOP代理生成**：`postProcessAfterInitialization()`阶段，`AbstractAutoProxyCreator`会为需要代理的Bean生成动态代理对象

###### 销毁阶段（Destruction）

- **目标**：容器关闭时释放资源。
- **实现DisposableBean接口**：重写`destroy()`方法。
- **XML配置`destroy-method`**：指定自定义销毁方法。
- **注解`@PreDestroy`**：标记销毁前执行的方法。

#### 依赖注入（DI）的实现

###### 依赖注入的三种方式

| **方式**       | **实现形式**                           | **适用场景**                       |
| :------------- | :------------------------------------- | :--------------------------------- |
| **构造器注入** | 通过构造函数参数传递依赖对象           | 强制依赖、不可变对象、避免循环依赖 |
| **Setter注入** | 通过Setter方法设置依赖对象             | 可选依赖、需要动态更新依赖         |
| **字段注入**   | 通过反射直接注入字段（如`@Autowired`） | 简化代码，但破坏封装性             |

###### 依赖注入的源码实现流程

- Spring的依赖注入流程主要发生在Bean实例化后的**属性填充阶段**（`populateBean()`方法）。

- **注解驱动的依赖注入**

  - **核心处理器：`AutowiredAnnotationBeanPostProcessor`**

  - **职责**：处理`@Autowired`、`@Value`、`@Inject`等注解。
  - **字段注入**：通过反射直接设置字段值。
  - **方法注入**：调用Setter方法或任意方法。

- **XML配置的依赖注入**
  - **Setter注入**：解析`<property>`标签的`name`和`value`/`ref`。
  - **构造器注入**：解析`<constructor-arg>`标签的`index`、`type`和`value`/`ref`。

###### 循环依赖的处理

- **一级缓存（singletonObjects）**：存放完全初始化的Bean。
- **二级缓存（earlySingletonObjects）**：存放提前暴露的Bean（未完成属性填充）。
- **三级缓存（singletonFactories）**：存放Bean的工厂对象，用于生成早期引用。
- **循环依赖解决流程示例**
  - 实例化A（`createBeanInstance()`），并将A的工厂对象放入三级缓存。
  - 填充A的属性时发现依赖B，开始实例化B。
  - 实例化B后，填充B的属性时发现依赖A，从三级缓存中获取A的早期引用。
  - B完成初始化后，A继续完成属性填充和初始化。

###### 不同注入方式的对比

| **维度**       | **构造器注入**          | **Setter注入**     | **字段注入**     |
| :------------- | :---------------------- | :----------------- | :--------------- |
| **不可变性**   | 支持（final字段）       | 不支持             | 不支持           |
| **循环依赖**   | 需配合`@Lazy`或调整设计 | 三级缓存天然支持   | 同Setter注入     |
| **可测试性**   | 高（依赖明确）          | 中（需调用Setter） | 低（依赖反射）   |
| **代码简洁性** | 冗余（参数较多时）      | 适中               | 高（无样板代码） |

#### Bean的生命周期

###### 实例化（Instantiation）

- **目标**：创建Bean的原始对象。
- **触发时机**：容器启动时（单例Bean）或每次请求时（原型Bean）。
- **实现方式**：
  - **构造函数**：通过无参或有参构造函数实例化。
  - **工厂方法**：静态工厂（`factory-method`）或实例工厂（`factory-bean`）。

###### 属性赋值（Population）

- **目标**：注入依赖的属性和值。
- **Setter注入**：通过`<property>`标签或`@Autowired`注解。
- **构造器注入**：通过`<constructor-arg>`标签或构造函数参数上的`@Autowired`。

###### 初始化（Initialization）

- **目标**：执行初始化逻辑，使Bean进入可用状态。
- **Aware接口回调**
  - **BeanNameAware**：设置Bean的名称。
  - **BeanFactoryAware**：设置Bean所属的BeanFactory。
  - **ApplicationContextAware**：设置ApplicationContext（扩展更多功能，如资源访问）。
- **BeanPostProcessor的前置处理**
  - **接口**：`BeanPostProcessor.postProcessBeforeInitialization()`
  - **功能**：在初始化方法前执行自定义逻辑。
  - **典型应用**：处理`@PostConstruct`注解。
- **初始化方法执行**
  - **方式1**：实现`InitializingBean`接口的`afterPropertiesSet()`方法。
  - **方式2**：XML配置的`init-method`指定方法。
  - **方式3**：使用`@PostConstruct`注解标记方法。
- **BeanPostProcessor的后置处理**
  - **接口**：`BeanPostProcessor.postProcessAfterInitialization()`
  - **功能**：在初始化方法后执行自定义逻辑。
  - **典型应用**：生成AOP代理对象（`AbstractAutoProxyCreator`）。

###### 使用（In Use）

- **目标**：Bean已就绪，可被应用程序使用。
- **单例Bean**：在整个容器生命周期内复用。
- **原型Bean**：每次请求生成新实例。

###### 销毁（Destruction）

- **目标**：容器关闭时释放资源。
- **实现DisposableBean接口**：重写`destroy()`方法。
- **XML配置的`destroy-method`**：指定自定义方法。
- **使用`@PreDestroy`注解**：标记销毁前执行的方法。

#### 容器扩展

###### BeanFactoryPostProcessor

- **定位**：Spring 容器的扩展点，允许在 **Bean 实例化之前** 修改 Bean 的定义（`BeanDefinition`）。
- **核心功能**：修改已加载的 Bean 的属性值；添加或删除 Bean 定义；调整 Bean 的作用域、初始化方法等元数据。
- **执行时机**：在 `BeanFactory` 加载完所有 Bean 定义后，但在创建 Bean 实例之前。

###### PropertyPlaceholderConfigurer

- **作用**：将 Bean 定义中的占位符（如 `${jdbc.url}`）替换为配置文件（如 `.properties`）中的实际值。
- **使用场景**：解耦配置与代码，支持多环境配置切换。

## 第三章 AOP的实现原理

#### AOP核心概念

###### 核心概念解析

- **Aspect（切面）**： 封装横切关注点的模块，包含多个 **Advice** 和 **Pointcut**，如日志切面、事务切面、权限校验切面。
- **Join Point（连接点）**：程序执行过程中的一个点（如方法调用、异常抛出），可插入切面逻辑的位置。
- **Advice（通知）**：在特定连接点执行的动作（如前置、后置、环绕处理），如记录方法执行时间、事务提交或回滚。
- **Pointcut（切点）**：通过表达式匹配一组连接点，定义哪些连接点会被切面处理。
- **Target Object（目标对象）**：被代理的原始对象（包含业务逻辑的 Bean）。
- **Proxy（代理）**：由 Spring 生成的代理对象，包装目标对象以插入切面逻辑。
- **Weaving（织入）**：将切面代码与目标对象关联的过程（编译时、类加载时或运行时）。

###### Advice（通知）类型

- **@Before（前置通知）**：在目标方法执行前触发；适用于参数校验、权限控制。
- **@After（后置通知）**：在目标方法执行后触发（无论是否抛出异常），适用于资源清理（如关闭文件流）。
- **@AfterReturning（返回后通知）**：在目标方法 **正常返回后** 触发，可访问返回值。
- **@AfterThrowing（异常通知）**：在目标方法 **抛出异常后** 触发，可捕获特定异常类型。
- **@Around（环绕通知）**：包裹目标方法，控制其执行流程（类似过滤器），需手动调用 `proceed()` 执行目标方法。

###### Pointcut（切点）表达式

| **表达式**                                 | **说明**                                        |
| :----------------------------------------- | :---------------------------------------------- |
| `execution(* com.example.service.*.*(..))` | 匹配 `com.example.service` 包下所有类的所有方法 |
| `@annotation(com.example.anno.Log)`        | 匹配被 `@Log` 注解标记的方法                    |
| `within(com.example.service.UserService)`  | 匹配 `UserService` 类中的所有方法               |
| `args(java.lang.String)`                   | 匹配参数类型为 `String` 的方法                  |

###### 代理机制

- **JDK 动态代理**
  - **条件**：目标对象实现了至少一个接口。
  - **原理**：基于接口生成代理类，调用 `InvocationHandler.invoke()` 插入切面逻辑。
- **CGLIB 动态代理**
  - **条件**：目标对象未实现接口（或配置强制使用 CGLIB）。
  - **原理**：通过继承目标类生成子类代理，覆盖父类方法。

###### AOP 与 AspectJ 的关系

| **维度**     | **Spring AOP**           | **AspectJ**                                |
| :----------- | :----------------------- | :----------------------------------------- |
| **织入时机** | 运行时动态代理           | 编译时或类加载时（支持更丰富的连接点）     |
| **性能**     | 略低（运行时生成代理）   | 更高（编译时优化）                         |
| **功能范围** | 仅支持方法级别的连接点   | 支持字段、构造器、静态代码块等连接点       |
| **使用场景** | 轻量级应用，无需复杂切面 | 企业级复杂切面需求（如性能监控、安全检查） |

#### JDK动态代理与CGLIB代理的底层实现

###### JDK 动态代理

- **核心原理**

  - **基于接口**：要求目标对象必须实现至少一个接口。

  - **反射机制**：通过 `java.lang.reflect.Proxy` 和 `InvocationHandler` 动态生成代理类。

  - **代理对象行为**：代理类实现目标接口，并将方法调用转发到 `InvocationHandler`。

- **实现步骤**

  - **定义接口与实现类**

    ```java
    public interface UserService {
        void saveUser(User user);
    }
    
    public class UserServiceImpl implements UserService {
        public void saveUser(User user) { /* 业务逻辑 */ }
    }
    ```

  - **实现 `InvocationHandler`**

    ```java
    public class JdkProxyHandler implements InvocationHandler {
        private Object target; // 目标对象
    
        public JdkProxyHandler(Object target) {
            this.target = target;
        }
    
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            // 前置增强
            System.out.println("Before method: " + method.getName());
            // 调用目标方法
            Object result = method.invoke(target, args);
            // 后置增强
            System.out.println("After method: " + method.getName());
            return result;
        }
    }
    ```

  - **生成代理对象**

    ```java
    UserService target = new UserServiceImpl();
    UserService proxy = (UserService) Proxy.newProxyInstance(
        target.getClass().getClassLoader(),
        target.getClass().getInterfaces(), // 必须为接口数组
        new JdkProxyHandler(target)
    );
    proxy.saveUser(new User()); // 调用代理方法
    ```

- **源码关键点**
  - **`Proxy.newProxyInstance()`**：动态生成代理类的字节码，其类名通常为 `$Proxy0`、`$Proxy1` 等。
  - **代理类结构**：代理类继承 `Proxy` 并实现目标接口，所有方法调用均委托给 `InvocationHandler.invoke()`。

###### CGLIB 动态代理

- **核心原理**

  - **基于继承**：通过生成目标类的子类作为代理类（即使目标类未实现接口）。

  - **字节码操作**：使用 ASM 库直接修改字节码，生成新类。

  - **方法拦截**：通过 `MethodInterceptor` 接口实现方法增强。

- **实现步骤**

  - **定义目标类**

    ```java
    public class UserService {
        public void saveUser(User user) { /* 业务逻辑 */ }
    }
    ```

  - **实现 `MethodInterceptor`**

    ```java
    public class CglibMethodInterceptor implements MethodInterceptor {
        public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
            // 前置增强
            System.out.println("Before method: " + method.getName());
            // 调用目标方法（通过 FastClass 机制，避免反射）
            Object result = proxy.invokeSuper(obj, args);
            // 后置增强
            System.out.println("After method: " + method.getName());
            return result;
        }
    }
    ```

  - **生成代理对象**

    ```java
    Enhancer enhancer = new Enhancer();
    enhancer.setSuperclass(UserService.class); // 设置父类
    enhancer.setCallback(new CglibMethodInterceptor());
    UserService proxy = (UserService) enhancer.create(); // 生成代理对象
    proxy.saveUser(new User()); // 调用代理方法
    ```

- **源码关键点**
  - **Enhancer 类**：负责生成代理类的字节码。
  - **FastClass 机制**：为代理类和目标类生成索引，直接通过索引调用方法，**避免反射**（性能优于 JDK 代理）。
  - **代理类结构**：代理类继承目标类，重写父类方法，并在方法中调用 `MethodInterceptor.intercept()`。

###### JDK 代理与 CGLIB 代理对比

| **维度**       | **JDK 动态代理**       | **CGLIB 动态代理**                       |
| :------------- | :--------------------- | :--------------------------------------- |
| **代理方式**   | 基于接口               | 基于继承                                 |
| **目标类要求** | 必须实现接口           | 可为任意类（非 final）                   |
| **性能**       | 生成快，调用慢（反射） | 生成慢，调用快（FastClass）              |
| **方法覆盖**   | 仅代理接口方法         | 代理所有非 final 方法                    |
| **依赖**       | 内置 JDK 支持          | 需引入 CGLIB 库（Spring 已默认包含）     |
| **代理类名**   | `$Proxy0`、`$Proxy1`   | `UserService$$EnhancerByCGLIB$$12345678` |

###### Spring AOP 的代理选择策略

- **默认行为**：若目标类实现接口 → 使用 JDK 动态代理；若目标类未实现接口 → 使用 CGLIB 动态代理。
- **强制使用 CGLIB**：通过 `@EnableAspectJAutoProxy(proxyTargetClass = true)` 配置，强制对所有类使用 CGLIB。
- **排除 final 方法**：CGLIB 无法代理 `final` 方法，需确保目标方法可被重写。

#### 代理对象的生成流程

###### 核心类与接口

- **ProxyFactory**
  - **作用**：代理对象的配置工厂，用于设置目标对象、切面（Advisor）、通知（Advice）等。
  - **继承关系**：`ProxyFactory` → `ProxyCreatorSupport` → `AdvisedSupport`。
- **AopProxy**
  - **接口**：定义代理对象的生成方法 `getProxy()`。
  - **JdkDynamicAopProxy**：基于 JDK 动态代理实现。
  - **ObjenesisCglibAopProxy**：基于 CGLIB 动态代理实现（Spring 优化后的版本）。
- **AdvisedSupport**：封装代理配置信息（目标对象、切面列表、代理接口等），供 `AopProxy` 使用。

###### 代理生成流程

- **配置阶段（ProxyFactory 初始化）**：开发者通过 `ProxyFactory` 设置代理所需的元数据。

  ```java
  // 1. 创建ProxyFactory并配置
  ProxyFactory proxyFactory = new ProxyFactory();
  proxyFactory.setTarget(new UserServiceImpl());       // 目标对象
  proxyFactory.addAdvice(new LoggingAdvice());         // 添加通知（Advice）
  proxyFactory.setInterfaces(UserService.class);       // 指定代理接口（可选）
  // 2. 生成代理对象
  UserService proxy = (UserService) proxyFactory.getProxy();
  ```

- **代理生成阶段（AopProxy 创建代理）**：选择代理策略（JDK或CGLIB）、创建AopProxy实例、生成代理对象。

#### 通知（Advice）的执行链路

###### 责任链模式的核心思想

- **定义**：将多个处理器（拦截器）按顺序连接成链，每个处理器决定是否将请求传递给下一个处理器。
- **优点**：解耦处理器之间的依赖，支持动态扩展处理逻辑。
- **在 Spring AOP 中的应用**：将多个通知（如 `@Around`）转换为拦截器（`MethodInterceptor`），形成链式结构，按顺序执行。

###### 拦截器链的构建

- **拦截器链的组成**：每个 `Advisor`（切面） 会被适配为一个 `MethodInterceptor`（方法拦截器）。
- **链的构建过程**：Spring 在创建代理对象时，将所有 `Advisor `转换为 `MethodInterceptor`，并按优先级排序，形成拦截器链。

###### 拦截器链执行流程

```java
               +---------------------+
               | MethodInvocation    |
               | (ReflectiveMethod)  |
               +----------+----------+
                          | proceed()
                          v
+----------------+     +----------------+     +----------------+     +----------------+
| Interceptor 1  | --> | Interceptor 2  | --> | Interceptor 3  | --> | Target Method  |
| (@Before)      |     | (@Around)      |     | (@After)       |     |                |
+----------------+     +----------------+     +----------------+     +----------------+
```

#### AspectJ注解驱动的AOP解析过程

###### 切面类的识别与注册

- **@Aspect注解**：标记一个类为切面。
- **@Component或@Bean**：确保切面类被Spring容器管理。
- **AnnotationAwareAspectJAutoProxyCreator**：继承自`AbstractAutoProxyCreator`的后置处理器，负责初始化后生成Proxy。

###### 切点表达式解析与匹配

- **AspectJExpressionPointcut**：封装AspectJ切点表达式，实现`Pointcut`接口的`getClassFilter()`和`getMethodMatcher()`。
- **表达式解析**：使用AspectJ的`PointcutParser`将字符串表达式转换为抽象语法树（AST）。
- **匹配逻辑**：在运行时检查目标类和方法是否匹配切点表达式。

###### 通知方法适配为Advice

| **注解**            | **Advice类型**         | **适配器类**                  |
| :------------------ | :--------------------- | :---------------------------- |
| **@Before**         | `MethodBeforeAdvice`   | `AspectJMethodBeforeAdvice`   |
| **@After**          | `AfterAdvice`          | `AspectJAfterAdvice`          |
| **@AfterReturning** | `AfterReturningAdvice` | `AspectJAfterReturningAdvice` |
| **@AfterThrowing**  | `ThrowsAdvice`         | `AspectJAfterThrowingAdvice`  |
| **@Around**         | `MethodInterceptor`    | `AspectJAroundAdvice`         |

###### 构建Advisor并整合到代理

- **收集Advisor**：在`AnnotationAwareAspectJAutoProxyCreator`中，查找所有切面类的Advisor。
- **匹配目标Bean**：判断当前Bean是否需要被代理（即是否存在匹配的Advisor）。
- **生成代理对象**：根据配置（JDK或CGLIB）生成代理对象，并将Advisor转换为拦截器链。

###### 动态代理生成与拦截链执行

- **代理对象调用方法**：代理对象拦截目标方法调用。
- **构建拦截器链**：所有匹配的`Advisor`转换为`MethodInterceptor`，按优先级排序。
- **链式执行**：通过`ReflectiveMethodInvocation`递归调用拦截器，直至执行目标方法。

## 第四章 Spring事务管理

#### 事务抽象层：PlatformTransactionManager与事务属性



#### 声明式事务的实现原理（@Transactional注解解析）



#### 事务传播机制与隔离级别的源码追踪



#### 事务同步与回滚逻辑



## 第五章 Spring MVC设计与实现

#### DispatcherServlet的初始化与请求处理流程

###### 初始化阶段

- **Servlet 生命周期触发**：当 Web 容器（如 Tomcat）启动时，根据注解/配置，DispatcherServlet 的 `init()` 方法被调用。
- **初始化 WebApplicationContext**
  - **根 WebApplicationContext**：由 `ContextLoaderListener` 加载，包含 Service、DAO 等非 Web 层 Bean。
  - **DispatcherServlet 子上下文**：专属于 Servlet，包含 Controller、ViewResolver 等 Web 层 Bean，继承根上下文。
- **初始化策略组件**
  - **HandlerMapping**：将请求映射到处理器（Controller 方法），如 `RequestMappingHandlerMapping`。
  - **HandlerAdapter**：执行处理器方法，适配不同处理器类型，如 `RequestMappingHandlerAdapter`。
  - **HandlerExceptionResolver**：处理请求过程中抛出的异常。
  - **ViewResolver**：解析逻辑视图名到具体视图（如 JSP、Thymeleaf）。
  - **LocaleResolver**：解析客户端区域信息（国际化）。
  - **ThemeResolver**：解析主题信息。
  - **RequestToViewNameTranslator**：请求到视图名的默认转换。
  - **FlashMapManager**：管理 Flash 属性（重定向时的临时数据存储）。
  - **MultipartResolver**：处理文件上传请求。

- **默认组件加载规则**
  - **按类型查找**：从容器中查找对应类型的 Bean（如 `ViewResolver`）。
  - **默认策略**：若未找到，加载 `DispatcherServlet.properties` 中定义的默认实现类。

###### 请求处理阶段

- **请求到达与分发**：当 HTTP 请求到达时，Servlet 容器的 `service()` 方法触发，最终调用

- **获取处理器执行链（HandlerExecutionChain）**

  - **HandlerMapping的作用**：根据请求 URL 匹配对应处理器（Controller方法），并收集关联拦截器（`HandlerInterceptor`）。
  - **匹配优先级**：`RequestMappingHandlerMapping`（基于 `@RequestMapping`）优先于 `BeanNameUrlHandlerMapping`。

- **获取处理器适配器（HandlerAdapter）**

  - **适配器模式**：不同处理器（如基于注解的 `@Controller`、传统的 `Controller` 接口）需要不同的适配器执行。

  - **常用适配器**：

    > `RequestMappingHandlerAdapter`：处理 `@RequestMapping` 方法。
    >
    > `HttpRequestHandlerAdapter`：处理 `HttpRequestHandler`（如静态资源处理）。
    >
    > `SimpleControllerHandlerAdapter`：处理 `Controller` 接口实现类。

- **执行处理器方法**
  - **参数解析与绑定**：`HandlerMethodArgumentResolver` 解析方法参数（如 `@RequestParam`、`@RequestBody`）。
  - **返回值处理**：`HandlerMethodReturnValueHandler` 处理返回值（如 `@ResponseBody` 转 JSON）。

- **视图渲染**
  - **ViewResolver**：解析视图名（如 `"home"`）为 `View` 对象（如 `InternalResourceView`）。
  - **View**：渲染模型数据（如填充 JSP 中的 `${message}`）。
- **异常处理**
  - **HandlerExceptionResolver**：捕获处理器方法或拦截器抛出的异常，生成错误视图或状态码（如 `@ExceptionHandler`）。
  - **默认实现**：`ExceptionHandlerExceptionResolver`（处理 `@ExceptionHandler` 方法）、`ResponseStatusExceptionResolver`（处理 `@ResponseStatus` 注解）。

- **拦截器（Interceptor）的执行顺序**
  - **preHandle()**：请求处理前执行（如权限校验）。
  - **postHandle()**：处理器方法执行后，视图渲染前执行（如修改模型数据）。
  - **afterCompletion()**：整个请求完成后执行（如资源清理）。

#### HandlerMapping与HandlerAdapter的职责解析

###### HandlerMapping：请求与处理器的映射器

- **核心职责**

  - **请求路由**：根据HTTP请求的URL、请求方法（GET/POST等）、请求头等信息，找到对应的处理器（Handler）。

  - **处理器链构建**：返回一个`HandlerExecutionChain`对象，包含目标处理器及其关联的拦截器（`HandlerInterceptor`）。

  - **多策略支持**：支持不同类型的映射策略（如基于注解、基于XML配置、基于Bean名称等）。

- **常见实现类**

  - **RequestMappingHandlerMapping**：处理`@RequestMapping`注解（包括`@GetMapping`、`@PostMapping`等衍生注解）。
  - **BeanNameUrlHandlerMapping**：根据Bean名称与URL匹配（如Bean名以`/`开头）。
  - **SimpleUrlHandlerMapping**： 通过XML或Java配置显式映射URL到处理器（如静态资源处理）。

- **工作流程**

  - **请求匹配**：遍历所有注册的`HandlerMapping`，调用其`getHandler()`方法，直到找到匹配的处理器。
  - **拦截器绑定**：将匹配的处理器与配置的拦截器组合成`HandlerExecutionChain`。
  - **优先级控制**：通过`Order`注解或实现`Ordered`接口调整多个`HandlerMapping`的执行顺序。

###### HandlerAdapter：处理器的适配执行器

- **核心职责**

  - **处理器适配**：将不同类型的处理器（如`@Controller`、`HttpRequestHandler`）统一适配为可执行的逻辑。

  - **方法调用**：反射调用处理器方法，处理参数绑定、返回值转换等细节。

  - **异常处理**：捕获处理器执行过程中的异常，转换为统一的处理流程。

- **常见实现类**

  - **RequestMappingHandlerAdapter**：适配基于`@RequestMapping`的处理器方法（最常用）。
  - **HttpRequestHandlerAdapter**：适配`HttpRequestHandler`接口（如处理静态资源的`ResourceHttpRequestHandler`）。
  - **SimpleControllerHandlerAdapter**：适配实现`Controller`接口的传统处理器。

- **工作流程**

  - **适配器选择**：根据处理器类型选择对应的`HandlerAdapter`。
  - **参数解析**：通过`HandlerMethodArgumentResolver`解析请求参数（如`@RequestParam`、`@RequestBody`）。
  - **方法执行**：反射调用处理器方法，获取返回值。
  - **返回值处理**：通过`HandlerMethodReturnValueHandler`处理返回值（如`@ResponseBody`转JSON）。

#### 视图解析与渲染

###### ViewResolver（视图解析器）

- **作用**：将控制器返回的 **逻辑视图名**（如 `"home"`）解析为具体的 **View 对象**。
- **核心方法**：`View resolveViewName(String viewName, Locale locale)`。
- **实现类**：
  - **InternalResourceViewResolver**：解析 JSP、HTML 等内部资源视图。
  - **ThymeleafViewResolver**：解析 Thymeleaf 模板。
  - **ContentNegotiatingViewResolver**：根据请求的媒体类型（如 `Accept` 头）协商视图。
  - **JsonViewResolver**：返回 JSON 视图（如结合 `@ResponseBody`）。

###### View（视图）

- **作用**：负责将模型数据（`Model`）渲染为具体的响应内容（如生成 HTML、写入 JSON）。
- **核心方法**：`void render(Map<String, ?> model, HttpServletRequest request, HttpServletResponse response)`。
- **实现类**：
  - **InternalResourceView**：渲染 JSP 页面。
  - **ThymeleafView**：渲染 Thymeleaf 模板。
  - **MappingJackson2JsonView**：将模型数据转为 JSON 响应。
  - **AbstractPdfView**：生成 PDF 文件。

###### 视图解析与渲染流程

- **控制器返回视图名**：控制器方法返回 `String` 类型的视图名（如 `return "home";`）或 `ModelAndView` 对象。
- **DispatcherServlet 委托 ViewResolver 解析**：`DispatcherServlet` 遍历所有注册的 `ViewResolver`，调用 `resolveViewName()` 方法，直到找到第一个非 `null` 的 `View` 对象。
  - **优先级控制**：通过 `Order` 注解或实现 `Ordered` 接口调整 `ViewResolver` 的执行顺序。
- **View 渲染模型数据**：获取 `View` 对象后，调用其 `render()` 方法，将模型数据与响应结合：
- **响应返回客户端**：最终生成的 HTML、JSON 或其他内容通过 `HttpServletResponse` 返回客户端。

#### 参数绑定与数据转换

###### 参数绑定

- **作用**：将外部输入（如 HTTP 请求参数、配置文件值）映射到方法参数或对象属性。
- **场景**：控制器方法通过`@RequestParam`绑定请求参数；配置文件通过`@Value`注入属性值；AOP切面中拦截方法参数进行修改验证。

###### 数据转换

- **作用**：将字符串或其他类型的输入数据转换为目标类型（如 `String` → `Date`）。
- **核心组件**：
  - **Converter<S, T>**：通用类型转换接口（如 `String` → `Integer`）。
  - **Formatter\<T\>**：面向区域（Locale）的格式化接口（如 `Date` ↔ `String`）。
  - **ConversionService**：统一管理所有转换器，提供类型转换服务。

###### Converter 与 Formatter

- **Converter（类型转换器）**：适用于通用的类型转换逻辑，无需考虑区域（Locale）。
- **Formatter（格式化器）**：需考虑区域化的格式化（如日期、货币）。
- **自动生效**：Spring 在参数绑定时自动调用 `ConversionService` 完成转换。

#### 异常处理机制

###### 核心组件与职责

- **HandlerExceptionResolver**：解析异常并生成错误视图或响应，是异常处理的顶层接口。
- **@ExceptionHandler**：注解在方法上，标记该方法用于处理特定类型的异常（通常结合 `@ControllerAdvice`）。
- **@ControllerAdvice**：定义全局异常处理类，集中处理多个控制器的异常。
- **DefaultHandlerExceptionResolver**：Spring 默认实现，处理标准 Spring MVC 异常。
- **ResponseStatusExceptionResolver**：根据 `@ResponseStatus` 注解设置 HTTP 状态码和错误信息。
- **ExceptionHandlerExceptionResolver**：处理 `@ExceptionHandler` 注解标记的方法，最常用的异常处理器。

###### 异常处理流程

- **查找匹配的 @ExceptionHandler**：在抛出异常的控制器类中查找 `@ExceptionHandler` 方法；若未找到，在 `@ControllerAdvice` 全局类中查找。
- **遍历 HandlerExceptionResolver链**：Spring 内置的解析器按以下顺序尝试处理异常：
  - **ExceptionHandlerExceptionResolver**：处理 `@ExceptionHandler` 方法。
  - **ResponseStatusExceptionResolver**：处理 `@ResponseStatus` 注解。
  - **DefaultHandlerExceptionResolver**：处理标准 Spring 异常。
- **生成错误响应**：
  - 解析器返回 `ModelAndView`（如错误页面）或直接修改 `HttpServletResponse`（如设置状态码）。
  - 若所有解析器均无法处理异常，由 Servlet 容器（如 Tomcat）返回默认错误页（如 500 页面）。

## 第六章 Spring数据访问

#### JDBC模板类的设计与资源管理（JdbcTemplate）



#### 事务管理在DAO层的整合



#### Spring对ORM框架的支持（Hibernate、MyBatis整合原理）



## 第七章 Spring注解驱动开发

#### 注解扫描的实现（ComponentScan与ClassPathBeanDefinitionScanner）



#### 条件装配（@Conditional与Condition接口）



#### @Configuration与@Bean的底层处理



#### 注解驱动的AOP与事务配置



## 第八章 Spring Boot核心原理

#### 自动配置的实现（Spring Boot Starter与@EnableAutoConfiguration）



#### 条件注解的扩展（@ConditionalOnClass、@ConditionalOnProperty等）



#### 内嵌Web容器（Tomcat、Jetty）的启动流程



#### Spring Boot的“约定优于配置”设计思想



## 第九章 Spring中的设计模式

#### 模板方法模式（JdbcTemplate、RestTemplate）



#### 观察者模式（ApplicationEvent与Listener）



#### 策略模式（ResourceLoader与Resource）



#### 适配器模式（HandlerAdapter）



## 第十章 Spring的扩展机制

#### 自定义BeanPostProcessor与BeanFactoryPostProcessor



#### 实现FactoryBean与BeanFactoryAware



#### 自定义作用域（Scope）与生命周期回调



#### Spring SPI机制：spring.factories与自动装配