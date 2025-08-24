## Component

#### ApiGovernance

###### Domain(orchestra)

- **Entity**
  - **Field**：包含字段名、依赖字段以及计算类型，如`(t1.a - t2.b) / t1.a AS field`。
  
  - **DagNode**：DAG节点，包含输入节点、输出节点以及ApiOrchestraNode（API编排节点）。
  
  - **ApiRule**：API规则，包含API ID、请求表达式映射、结果输出映射及默认请求参数。
  
  - **JoinRule**：Join规则，包含Join节点ID、左Join逻辑、右Join逻辑、连接类型和连接条件。
  
  - **AggregateRule**：聚合规则，包含计算字段集合以及聚合字段集合。
  - **CompositeCalcRule**：组合计算规则，包含返回字段及其计算表达式。
  
  - **EndRule**：节点结束规则，包含节点输出字段、对输出字段过滤以及节点输入和输出字段的映射。
  
- **DagContext**：DAG上下文，包含开始节点、节点Map（视图ID及其查询结果集）、DAG执行工厂、API信息、API上下文、API执行方法。

- **DagAppService**：DAG服务，若父节点未执行完毕，则异步并行执行父节点任务，然后执行当前节点任务；若当前节点非结束节点，则异步执行当前节点，否则同步执行当前节点，便于调用方收集结果。

- **Excutor**
  - **ApiNodeFactory**：API节点工厂，包含生成API上下文方法、生成API实例方法、表达式创建OS请求方法、创建返回结果方法。
  - **DagNodeExecutor**：DAG节点执行器，声明了执行方法。
  - **ApiNodeExecutor**：首先根据DAG节点获取API编排规则及业务API信息，进而通过ApiNodeFactory生成API上下文，通过API上下文执行API查数方法获取数据结果，最后将查数结果通过API规则的OutputMapper进行映射返回节点输出结果。
  - **JoinNodeExecutor**：Join操作执行器，支持`INNER JOIN/LEFT JOIN/RIGHT JOIN/FULL JOIN`。执行时首先根据连接条件condition（主要指ON逻辑，如log_date = '20250101'，左右连接条件包含字段、数值、表达式等）依次判断左右表每行数据是否匹配，将匹配的数据行进行连接，如果是`LEFT JOIN/RIGHT JOIN/FULL JOIN`，未匹配则连接空值。
  - **AggregateNodeExecutor**：聚合操作执行器，执行时首先解析AggregateRule中计算字段和聚合字段的依赖字段，然后存储NodeId（Node可以理解成子视图，相当于大数据领域的fragment）及其依赖的字段映射，接着根据聚合字段生成MapKey进行数据分组，最后对聚合数据进行计算操作（`SUM/AVG/MAX/MIN`）。如`SELECT SUM(t1.id) AS pv FROM table t1 GROUP BY date`中，从Node(t1)中获取id字段，使用依赖字段映射生成别名后进行数据聚合计算。
  - **CompositeCalcNodeExecutor**：基于MVEL表达式计算字段展示值，如`SELECT a.amount + b.amount AS amount`。
  - **StartNodeExecutor**：节点开始处理逻辑，加载所有依赖API最大可用分区。
  - **EndNodeExecutor**：节点结束处理逻辑，字段裁剪、数据过滤、结果映射。
  

###### Interface

- **ApiInstance**：API实例信息，包含API ID、版本号、服务树应用、QPS、缓存配置、状态、应用场景等。

- **ApiResp**：API数据返回结果，包含数据列表、分页信息等。

- **AsyncApi**：异步API，仅包含任务ID。

- **AsyncSQLApi**：异步SQL API，包含引擎类型、执行SQL、调用类型等。

- **CubeModel**：Cube模型，支持根据维度组合进行Cube返回指标数据（需要补充）。

- **DegradeInfo**：数据降级，包含降级状态以及降级时间。

- **MixtureQuery**：混合查询，混合实时和离线的查询结果进行输出。

- **OperatorField**：生成字段过滤条件，包括字段名称、操作符号（$=,>,<,in,between,like,rlike$），字段数值。

  ```java
  AdvFilter.builder().field("log_date").operator("=").values(List.of("20250101")).build()
  ```

- **OsHeader**：OneService API请求头，包含apiId、appKey、secret。

- **PageVo**：分页返回结果，包含页号、分页大小、分页偏移量、数据条数。

- **QueryReq**：查询请求参数，包含OsHeader、请求条件、AdvFilter、返回字段、排序字段、分页字段、结果集过滤和二次计算。

  - **二次计算**：查到数据后对数据进行Group By、SUM、AVG等聚合运算。

  - **AdvFilter**：组合and和or运算，支持通过express进行嵌套。

    ```
    AdvFilter advFilter = AdvFilter.builder()
      .type("or")
      .express(List.of(
        AdvFilter.builder().field("log_date").operator("=").values(List.of("20250101")).build(),
        AdvFilter.builder().field("log_date").operator("=").values(List.of("20250831")).build()
      )).build();
    ```

###### Infrastrusture

- **Api**
  - **BapiMockApi**：ApiMock数据HTTP调用。
  - **OsApiService**：OneService HTTP调用，包含API元信息查询、在线查询、Dispatch分析任务、异步任务、最大分区。
  - **MetaApiRpcService**：Meta-Api RPC调用，提供API元信息查询。
  - **TaiShanRpcService**：TaiShanKv服务，提供缓存和点查场景查询服务。

- **Factory**
  - **ApiInstanceFactory**：API实例工厂，包含通过API实例获取模型名称、备链路引擎是否可查询、是否分页查询、获取查询/计算引擎（MySQL、Iceberg、Presto、Flink、Tidb、Spark、Clickhouse、Hive、TaishanKv、Kafka、Mongo、File）。
  - **CubeModelFactory**：Cube模型工厂，通过API上下文和API模型生成Cube模型上下文，包括纬度组合列表和请求维度组合。
  - **EngineModelReqFactory**：模型构建API工厂，通过请求参数resp得出指标列表和维度列表构建配置信息，最终配置信息包含请求指标列表、目标指标列表、结果集过滤条件、请求维度列表、维度基本过滤条件、维度高级过滤条件、分页和排序配置。
  - **EngineQueryReqFactory**：SQL构建API工厂，通过数据库ID、查询引擎、集群名称、数据库名称、查询SQL等构建配置信息。
  - **EngineTableReqFactory**：表构建API工厂，构建信息包括API ID、查询引擎、库名、表名、过滤条件、返回字段、排序字段、分区信息、字段映射、高级过滤条件、HAVING条件、分页配置。
  - **ExprOpVOFactory**：表达式操作工厂，包含检查过滤条件、检查请求参数、创建过滤器等方法。
  - **RangeFieldFactory**：范围字段工厂，对时间字段进行范围筛选。
  - **ResultCalculateFactory**：结果计算工厂，根据结果集进行聚合计算（SUM/AVG/MIN/MAX）。
  - **ResultOrderFactory**：结果排序工厂，进行升序降序操作。
  - **TaiShanResultHandleFactory**：Taishan结果处理工厂，执行结果集过滤及排序。


###### RateLimiter

- **RRateLimiter**：可以根据trySetRate方法，设置一定时间区间内的令牌数，如`trySetRate(RateType.OVERALL, 100, 1, RateIntervalUnit.SECONDS)`表示1s内向桶中放入100个令牌。

  ```java
  trySetRate(RateType rateType, long rate, long rateInterval, RateIntervalUnit unit)
  ```

- **远程限流**：通过API ID和API QPS生成RedisKey获取RRateLimiter对象，根据QPS设置全局分布式限流器；当有请求进入时，通过API最大QPS - 限流器可用令牌数（availablePermits）计算当前API请求QPS，并tryAcquire(1)判断当前请求是否限流。

- **本地限流**：API最大QPS超过阈值（一般为线上服务限流）时，使用ConcurrentHashMap<String, Semaphore>在本地存储配额；当有请求进入时，先判断本地map 中是否有配额，如不存在或者配额不足则请求远程获取配额。

- **远程配额**：不同于远程限流逻辑，远程限流一次只请求1个令牌，但是远程获取配额一次会获取多个令牌（10-100）存储本地。

- **缓存状态**：当批量获取配额失败或者单请求限流时，使用ConcurrentHashMap缓存API ID及限流截止时间戳。

###### Cache

- **ApiCache**：基于Redis缓存**版本号**，基于Caffeine本地缓存**版本号**，基于taishanKv缓存API结果数据。
  - ConcurrentMap将存储所有存入的数据，直到你显式将其移除。
  - Caffeine将通过给定的配置，自动移除“不常用”的数据，以保持内存的合理占用。

- **缓存流程**：当有请求进入时，首先查询缓存数据，有缓存则返回，否则调用回源方法查数，查数后异步写缓存。
- **查询缓存**：首先通过Caffeine查询是否本地缓存版本号（或降级版本号），没有则去Redis回源查询版本号，接着基于taishanKv通过缓存版本号 + 请求加密生成的key查询API结果缓存并返回。
- **写入缓存**：异步写缓存，首先基于Redis缓存生成的版本号，接着在taishanKv中通过缓存版本号 + 加密key写入缓存。

###### Application

- **RpcV3LogAgentAspect**：日志代理切面，上报API执行日志，包含请求开始日志、请求结束日志、鉴权失败日志、限流日志。
- **Assembler**
  - **ApiAliasFieldAssembler**：API别名转换，包含模型请求参数别名转换、填充排序字段及高级筛选字段。
  - **ApiMaxPartitionRespAssembler**：API最大分区转换，用于转换API最大分区。
  - **ApiRespAssembler**：API响应转换，通过QueryResp转换为ApiResp。
  - **AsyncAssembler**：异步请求转换。
  - **CalculateQueryReqAssembler**：计算查询请求转换。
  - **CalculateQueryRespAssembler**：计算查询响应转换。
  - **DegradeInfoAssembler**：降级信息转换。
  - **MixtureQueryRespAssembler**：混合计算响应转换。
  - **OpenApiRespAssembler**：OpenApi响应转换。
  - **OperatorFieldAssembler**：操作符字段转换。
  - **QueryReqAssembler**：查询请求转换。
  - **QueryRespAssembler**：查询响应转换。
  - **SqlRespAssembler**：SQL响应转换。

- **OpenApiContext**：API上下文，包含API ID、请求参数、二次计算请求参数、API实例信息、协议类型、实例QPS、编排信息。
- **Service**
  - **DataQueryBeforeHandleAppService**：数据查询前置处理服务，用户未请求及非结果集计算字段不处理。
  - **DataQueryAppService**：数据查询服务，封装引擎查数逻辑，将EngineQueryReq下发到engine应用进行数据查询。
  - **DataQueryAfterHandleAppService**：数据查询后置处理服务，包括结果集计算、响应字段裁剪。
  - **DataStatusAppService**：数据状态服务，包含获取API最后分区、API数据状态。
  - **DataDegradeAppService**：数据降级服务，包含离线请求降级（替换时间类型请求参数）、任务最大分区查询。
  - **PageQueryAppService**：分页查询服务，包含分页数据查询和分页总数查询。
  - **MockQueryAppService**：Mock查询服务，当实际数据未就绪时，提供Mock数据。
  - **TemplateBuildSqlAppService**：模板构建SQL服务，通过SQL模板参数透传生成SQL，类似于Mybatis的Mapper映射。
  - **OrchestraBuildQueryAppService**：编排构建查询服务，执行API编排逻辑，从开始节点进行执行，对其依赖节点进行拓扑排序，依次执行节点数据任务，最终返回编排结果。

###### Auth

- **ApiAuth**：API调用校验，调用方必须和所申请的appId、appKey、secret相一致。
- **PreventSqlInjection**：防止SQL注入，对SQL注入进行检查，并能够快速编译生成安全（非注入）SQL进行查询。

#### Translate

###### Domain

- **Model**
  - **LogicModel**：逻辑模型，包括引擎模型、模型元信息、是否真正翻译tdm、字段信息、是否模型快照等。
  - **LogicRbm**：rbm加速逻辑，包含查询维度、主键维度、查询指标和需要构建的逻辑表。
  - **LogicTag**：标签逻辑，包含分区字段和标签字段。
  - **ParserModel**：解析模型，包括模型信息及其模型字段信息。
  - **ControlModel**：控制模型，是否星座模型、是否跨源、是否基数探查。
  
- **ParserService**

  - **ParserService**：解析服务，（待补充）

  - **TranslateRbmService**：

  - **TranslateTemplateService**：SQL模版翻译服务，主要包含taishanKv翻译服务以及SQL模版翻译服务。

    > SQL模版翻译：首先使用占位符解析${name}的类型和默认值，然后遍历paramFilters匹配字段名，值不存在时使用默认值，两者都为空则抛异常，接着对字段进行类型转换，最后进行模版替换构建，单值直接替换，多值拼接为逗号分隔。
    >
    > taishanKv翻译：首先合并指标和维度字段，然后对Cube模型进行特殊处理（包括验证维度组合的合规性、补充缺失维度的默认值），接下来获取分区字段配置自动补全过滤条件的默认值，最后根据是否开启扫描模式选择不同模版进行模版替换。
    >
    > SQL模版动态替换：包括分页参数处理、排序方式设置以及维度条件过滤，如果是分区表需要单独处理分区逻辑。

  - **TranslateTableService**

    > **引擎路由**：对TAISHANKV引擎复用`translateTaiShan`逻辑，其他引擎走标准SQL翻译流程。
    >
    > **字段分类处理**：计算字段通过依赖字段以及计算表达式生成；虚拟字段单独分组处理；普通字段直接加入Select列表。
    >
    > **SELECT**：通过响应resp以及别名映射组合普通字段、虚拟字段（包括计算字段依赖字段）以及计算字段。
    >
    > **FROM**：支持模板表名或物理表名。
    >
    > **WHERE**：解析基础过滤条件和高级过滤条件。
    >
    > **GROUP BY**：自动生成分组条件，非计算字段进行group by。
    >
    > **HAVING**：处理聚合后过滤。

  - **TranslatePageService**

    > **总条数查询**：直接复用原始查询SQL包装成COUNT查询总条数。
    >
    > **分页构造**：起始位计算`(pageNum-1)*pageSize`，Iceberg使用OFFSET模板，其他引擎使用标准LIMIT语法。

  - **TranslateModelUnionService**：多模型联合查询逻辑，构建SELECT子句(维度+普通指标)，通过metaRpcService获取复合指标定义，递归构建复合指标表达式，使用AST模板生成最终SQL。

###### Infrastructure

- **MetaRpcService**：元信息RPC服务，包含业务字段查询、业务模型查询及模型字段查询。
- **ASTReplaceUtil**：模板替换工具，主要用于将对象属性值动态注入到字符串模板中。
- **ASTTranslateUtil**：SQL翻译工具类，主要负责模型查询场景的SQL片段生成与组装，包括LIMIT、ORDER BY、WHERE筛选（=、>、>=、<、<=、in、contains、between等）以及多模型跨源指标计算。
- **SqlParseUtil**：SQL解析工具类，专注于动态参数替换和类型安全转换。
- **TdmTimeUtil**：处理TDM（时间驱动模型）场景下的日期分区下推逻辑，计算wtd、mtd、qtd、mtd。

###### Application

- **Assembler**

  - **TranslateReqFieldAssembler**：字段映射转换的核心装配器，包含字段映射转换、多维处理以及递归处理。
  - **TranslateSqlAliasAssembler**：SQL别名处理器，包括SQL别名映射转换、SQL重构以及动态适配。
  - **TranslateModelFieldAssembler**：模型字段技术表达式处理器，包含高级字段解析、表达式处理和模版引擎。

- **Service**

  - **LevelSelect**：ClickHouse 一级 SELECT 语句生成器，包含多类型字段处理、表达式生成和FULL JOIN适配器。

  - **ModelASTService**：模型抽象树解析服务，包含SQL表达式解析、AST树操作（三层SQL处理）和函数映射。

    > **一级查询（基础查询层）**：基础字段表别名处理、FULL JOIN字段CASE WHEN表达式生成、实时计算字段类型转换。
    >
    > **二级查询（过滤聚合层）**：构建WHERE过滤条件、GROUP BY聚合逻辑。
    >
    > **三级查询（衍生计算层）**：构建高级计算表达式，如嵌套指标计算、时间窗口函数处理和跨模型字段处理。
    >
    > **SQL表达式解析**：将业务模型中的维度/指标表达式转化为标准SQL。
    >
    > **AST树操作**：通过抽象语法树实现SQL模板的动态替换与优化。
    >
    > **函数映射**：处理不同数据源间的函数兼容性问题。

  - **ModelRbmProduceASTService**：RBM查询生产服务，包括维度基数探查优化、物化试图预计算、跨层指标继承。

  - **ModelRbmConsumeASTService**：RBM查询消费服务，包含动态JOIN优化、主键维度识别和基数探查。

  - **TagProduceASTService**：标签生产抽象数服务，包含以下业务：

    > **单数据源标签**：直接生成带过滤条件的SELECT语句。
    >
    > **多数据源合并**：通过UNION ALL合并后GROUP BY去重。
    >
    > **历史标签回溯**：利用log_date/log_hour分区字段实现时间切片。
    >
    > **异构数据源整合**：处理不同来源的字段类型差异。

  - **TranslateService**：翻译服务总控调度器，主要负责多引擎翻译路由、模板缓存优化和翻译过程监控。

  - **TranslateClickhouseService**：ClickHouse查询生成服务，主要负责多层SQL构造**、**复合指标处理和跨模型关联查询，支持OLAP场景下的复杂分析查询。

  - **TranslateTaiShanService**：TaishanKv查询生成服务，主要负责维度排列组合、TaishanKey智能构建和多模型查询支持。

  - **TranslateJoinService**：多表关联查询生成服务，包含多表JOIN构建、动态别名管理、嵌套条件解析和跨引擎适配。

  - **TranslateDetailService**：明细查询SQL生成服务，包含多表别名管理、字段映射转换、FULL JOIN适配以及跨引擎兼容。

#### Engine



#### CacheSupport

###### Config

- **CaffeineCacheSupportConfig**：Caffeine本地缓存支持配置，包括分批回源条数、降级缓存和回源方法。
- **RedissonSupportConfig**：Redis支持配置，包含缓存TTL时间、时间单位、缓存分片数。

###### Support

- **CaffeineCacheSupport**：Caffeine缓存，包含单key查询、批量查询、异步查询、写缓存。

  - **异步查询**：请求包含Caffeine本地缓存、请求ID集合、异步回源方法、缓存配置；默认读主缓存，失效读降级缓存，缓存不存在则调用回源方法查数并写缓存。

    ```java
    getsAsync(Cache<CK, V> cache, Collection<CK> ids, Function<Collection<CK>, CompletableFuture<Map<CK, V>>> asyncBackSourceFunc, CaffeineCacheSupportConfig<CK, CK, V> config);
    ```

- **RedissonSupport**：Redis缓存，包括单key查询、批量查询、异步查询、异步写入、异步删除方法。

## Application

- 