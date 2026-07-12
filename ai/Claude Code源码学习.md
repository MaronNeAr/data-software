## 第一部分 核心架构设计

#### 核心设计思想

###### Agentic Loop（代理循环）

- **自主代理循环**：用户输入 → 处理用户输入 → 构建消息 → 调用 Claude API → 解析响应 → 
  提取 Tool Use → 工具编排执行 → 将结果返回模型 → 循环...

- 模型在循环中自主决定何时读取代码、执行命令、修改文件，直到完成任务。

###### 工具即能力（Tools as Capabilities）

- **文件操作**：FileReadTool、FileWriteTool、FileEditTool、NotebookEditTool
- **搜索与发现**：GlobTool、GrepTool、ToolSearchTool
- **系统执行**：BashTool、PowerShellTool
- **网络访问**：WebFetchTool、WebSearchTool
- **Agent与多智能体**：AgentTool、TaskStopTool、SendMessageTool、TeamCreateTool、TeamDeleteTool
- **任务管理**：TaskCreateTool、TaskGetTool、TaskUpdateTool、TaskListTool
- **后台任务输出**：TaskOutputTool
- **计划模式**：EnterPlanModeTool、 ExitPlanModeV2Tool
- **用户交互**：AskUserQuestionTool、TodoWriteTool
- **Skill与工作流**：SkillTool、WorkflowTool
- **MCP协议**：ListMcpResourcesTool、ReadMcpResourceTool
- **Git工作树**：EnterWorktreeTool、ExitWorktreeTool
- **Anthropic内部专属**：REPLTool、ConfigTool、TungstenTool、SuggestBackgroundPRTool、BriefTool
- **触发器/调度**：CronCreateTool、CronDeleteTool、CronListTool、RemoteTriggerTool
- **主动/后台模式**：SleepTool、MonitorTool
- **实验性/条件启用**：WebBrowserTool、TerminalCaptureTool、LSPTool、CtxInspectTool、SnipTool、ListPeersTool、OverflowTestTool、VerifyPlanExecutionTool、SendUserFileTool
- **通知/订阅**：PushNotificationTool、SubscribePRTool
- **测试专用**：TestingPermissionTool
- **合成输出**：SyntheticOutputTool

###### 终端原生

- **布局引擎** — 基于 Yoga Layout 的 Flexbox 实现
- **终端 I/O** — 完整的 ANSI/VT100 解析器、tokenizer
- **事件系统** — 键盘、鼠标、焦点事件统一处理
- **组件库** — Box, Text, ScrollBox, Button, Link 等
- **文本能力** — 颜色、双向文本、搜索高亮、Tab 对齐

###### 多 Agent 协作（Multi-Agent Architecture）

- 主 Agent 可通过 `AgentTool` 派生独立的子 Agent
- 子 Agent 可选在独立 Git worktree 中工作，避免文件冲突
- **Workflow 系统** — 编排大规模 Agent 团队并行/流水线执行
- Agent 间通过 `SendMessage` 通信
- 支持自定义 Agent 定义（通过用户配置目录）

###### 分层权限模型

| 模式           | 行为                               |
| -------------- | ---------------------------------- |
| `default`      | 按需询问用户确认                   |
| `accept-edits` | 自动批准编辑，询问危险操作         |
| `bypass`       | 跳过所有确认（高风险，需明确开启） |
| `plan`         | 仅允许只读操作，用于探索和规划     |

###### 上下文弹性管理

- **自动压缩（auto-compact）** — 当对话逼近 token 限制时，自动摘要早期回合
- **响应式压缩（reactive compact）** — 基于内容重要性动态决定保留策略
- **微压缩（micro-compact）** — 对单个工具调用结果进行无损摘要
- **Feature Flag 驱动的差异构建**：通过 `feature()` 函数 + Bun 的 dead code elimination 特性，在编译期根据用户类型消除不同代码路径，保持单一代码库同时交付内部(anthropic)和外部产品版本

###### MCP 协议原生支持

- MCP 工具自动发现与调用
- MCP 资源读取
- OAuth 认证流程
- MCP 配置热加载

#### 项目结构

```
ClaudeCode/
├── main.tsx                # 应用主入口（~800KB），初始化全系统
├── setup.ts                # 启动前置准备（Session、Worktree、Hook 等）
├── query.ts                # AI 交互核心逻辑
├── QueryEngine.ts          # 查询引擎，管理代理循环生命周期
├── Tool.ts                 # Tool 基类和类型定义
├── tools.ts                # 工具注册与导出
├── Task.ts                 # 任务基类和生命周期管理
├── commands.ts             # 斜线命令注册
├── context.ts              # System/User Context 构建
├── entrypoints/            # CLI、SDK 入口
│   ├── cli.tsx             # CLI 命令行入口
│   ├── sdk/                # Agent SDK
│   └── init.ts             # 初始化逻辑
├── components/             # React UI 组件（146+）
│   ├── App.tsx             # 应用根组件
│   ├── agents/             # Agent 相关 UI
│   └── design-system/      # 设计系统组件
├── state/                  # 全局状态管理
│   ├── AppState.tsx        # React Provider + Hooks
│   ├── AppStateStore.ts    # 状态 Store 定义
│   └── store.ts            # Store 实现
├── tools/                  # 工具实现（45+）
│   ├── AgentTool/          # 子 Agent 启动工具
│   ├── BashTool/           # Shell 执行
│   ├── FileEditTool/       # 文件编辑
│   ├── FileReadTool/       # 文件读取
│   ├── FileWriteTool/      # 文件写入
│   ├── GrepTool/           # 文本搜索
│   ├── GlobTool/           # 文件匹配
│   ├── SkillTool/          # 技能调度
│   ├── WebFetchTool/       # HTTP 请求
│   ├── WebSearchTool/      # 网页搜索
│   └── ...                 # 更多工具
├── tasks/                  # 后台任务实现
├── services/               # 领域服务
│   ├── api/                # Claude API 调用层
│   ├── mcp/                # MCP 客户端/服务端
│   ├── compact/            # 上下文压缩
│   ├── analytics/          # 遥测与特性开关
│   ├── plugins/            # 插件系统
│   ├── tools/              # 工具执行引擎
│   ├── lsp/                # LSP 集成
│   └── oauth/              # OAuth 认证
├── skills/                 # 技能定义
├── commands/               # 斜线命令实现
├── ink/                    # 自研终端渲染引擎
│   ├── components/         # 基础 Ink 组件
│   ├── hooks/              # Ink 专用 Hooks
│   ├── layout/             # 布局引擎（Yoga）
│   ├── events/             # 终端事件系统
│   └── termio/             # 终端 I/O 处理
├── bridge/                 # Code ↔ Desktop 桥接
├── utils/                  # 工具函数库（330+）
├── hooks/                  # React Hooks（87+）
├── constants/              # 常量定义
├── types/                  # TypeScript 类型定义
├── migrations/             # 配置/数据迁移脚本
├── keybindings/            # 键盘绑定
├── vim/                    # Vim 模式支持
├── plugins/                # 内置插件
├── buddy/                  # 伙伴系统
├── memdir/                 # 持久化记忆目录
├── voice/                  # 语音支持
└── server/                 # 服务端入口
```

## 第二部分 Claude Code核心组件

### Agentic Loop

```
QueryEngine (QueryEngine.ts)   ← 外层：会话生命周期管理
		└── query() (query.ts)     ← 内层：单次查询的 Agentic Loop
```

###### 整体架构

```
                   ┌────────────────────────────┐
  用户输入 ────────▶│      QueryEngine           │
                   │  .submitMessage(prompt)    │
                   │                            │
                   │  1. processUserInput       │ ← 处理斜线命令、Skill 注入
                   │  2. fetchSystemPrompt      │ ← 构建 System Prompt
                   │  3. load skills/plugins    │ ← 加载技能和插件
                   │                            │
                   │  ┌── query() ───────────┐  │
                   │  │  while(true) {       │  │
                   │  │    压缩/裁剪上下文     │  │
                   │  │    callModel() ──────┼──┼──▶ Anthropic API
                   │  │    解析 Tool Use      │  │
                   │  │    runTools() ───────┼──┼──▶ 工具执行引擎
                   │  │    attachments/通知   │  │
                   │  │    if continue       │  │
                   │  │    else return       │  │
                   │  │  }                   │  │
                   │  └──────────────────────┘  │
                   │                            │
                   │  4. 返回 result             │
                   └────────────────────────────┘
```

###### QueryEngine（外层 — 会话管理器）

- **QueryEngine**：是一个 **long-lived 状态容器**，一次会话创建一个实例。它的职责是"拥有和持有"对话状态，而不是执行循环本身

- **核心状态（构造函数初始化，跨 turn 保持）**

  ```
  class QueryEngine {
    private mutableMessages: Message[]        				// 完整消息历史（跨 turn 累积）
    private totalUsage: NonNullableUsage      				// 累计 token 用量
    private permissionDenials: SDKPermissionDenial[] 	// 权限拒绝记录
    private readFileState: FileStateCache     				// 文件读取缓存（含 memory）
    private discoveredSkillNames: Set<string> 				// turn 内发现的技能
    private loadedNestedMemoryPaths: Set<string> 			// 已加载的内嵌 memory
  }
  ```

- **submitMessage() 核心流程**

  ```
  submitMessage(prompt)
    │
    ├─ 1. 构建 System Prompt
    │     fetchSystemPromptParts() → defaultSystemPrompt + userContext + systemContext
    │     可选: memoryMechanicsPrompt / appendSystemPrompt
    │
    ├─ 2. 处理用户输入
    │     processUserInput(prompt) → { messages, shouldQuery, allowedTools, model }
    │     这一步会处理 /slash 命令、Skill 调用、$ARGUMENTS 替换
    │
    ├─ 3. 持久化 transcript
    │     recordTranscript(messages) — 确保即使中途崩溃也可 --resume
    │
    ├─ 4. --- 此时决定是否需要查询模型 ---
    │     if (!shouldQuery): 返回本地命令结果（如 /help、/cost）
    │
    ├─ 5. 加载 skills 和 plugins
    │     getSlashCommandToolSkills() + loadAllPluginsCacheOnly()
    │
    ├─ 6. 发送 system-init 消息给 SDK
    │     yield buildSystemInitMessage({ tools, mcpClients, model, ... })
    │
    └─ 7. 进入内层循环 ★
          for await (const message of query({ messages, systemPrompt, ... }))
            │
            ├─ 分类处理每条消息:
            │   assistant → accumulate + normalize + yield to SDK
            │   user      → accumulate + turnCount++ + yield to SDK
            │   progress  → accumulate + normalize + yield to SDK
            │   attachment → structured_output? max_turns_reached? queued_command?
            │   stream_event → accumulate usage, capture stop_reason
            │   system    → compact_boundary (释放 GC) / api_error / snip
            │
            ├─ 检查终止条件:
            │   maxBudgetUsd 超限? → result:error_max_budget_usd
            │   maxTurns 超限?    → result:error_max_turns
            │   structured output 重试超限? → result:error_max_structured_output_retries
            │
            └─ 循环结束后:
                判断 isResultSuccessful() → result:success 或 result:error_during_execution
  ```

  - **Transcript**：对话的完整历史记录，以 JSONL 格式持久化到磁盘，主要作用是用于对话恢复（--resume）
  - **Transcript写入点**：用户消息立即写、流式消息边收边写、对话结束 flush

###### query()（内层 — Agentic Loop 核心）

这是真正的代理循环，一个 `while(true)` 循环约 1730 行。每个循环迭代对应模型的**一次 API 调用 + 工具执行**。

- **循环状态机**

  ```
  type State = {
    messages: Message[]                          // 当前迭代的消息
    toolUseContext: ToolUseContext               // 工具上下文（随 turn 变化）
    autoCompactTracking: AutoCompactTrackingState // 压缩追踪
    maxOutputTokensRecoveryCount: number          // 输出 token 超限恢复计数
    hasAttemptedReactiveCompact: boolean          // 已尝试反应式压缩？
    pendingToolUseSummary: Promise<...>           // 上一次的 tool use summary
    turnCount: number                             // 当前 turn 计数
    transition: Continue | undefined              // 上一轮做了什么过渡（诊断用）
  }
  ```

- **单次迭代的完整流程**

  ```
  while(true) — 迭代 #N
    │
    │  【阶段 0: 上下文预处理】
    ├─ getMessagesAfterCompactBoundary() — 只保留上次压缩之后的消息
    ├─ applyToolResultBudget()           — 对超长 tool_result 做截断
    ├─ snipCompactIfNeeded()             — 裁剪历史（HISTORY_SNIP）
    ├─ microcompactMessages()            — 微压缩：删除冗余的 cache 编辑记录
    ├─ applyCollapsesIfNeeded()          — 上下文折叠（CONTEXT_COLLAPSE）
    │
    │  【阶段 1: 自动压缩】
    ├─ autoCompactIfNeeded()             — 如果接近 token 限制:
    │   ├─ 调用 Haiku 模型对历史对话做摘要
    │   ├─ 生成 compact_boundary 消息
    │   ├─ 更新 taskBudgetRemaining
    │   └─ buildPostCompactMessages()    — 用摘要消息替换被压缩的历史
    │
    │  【阶段 2: 调用模型 ★★★】
    ├─ deps.callModel({                  — 这是与 Claude API 交互的地方
    │     messages: prependUserContext(messages, userContext),
    │     systemPrompt: fullSystemPrompt,
    │     tools, thinkingConfig, model, signal, ...
    │   })
    │   │
    │   │   streaming 模式下:
    │   │   ├─ message_start  → 重置当前消息的 usage
    │   │   ├─ content_block_start/delta/stop → 逐个 block 到达
    │   │   ├─ 识别 tool_use blocks → push 到 toolUseBlocks[]
    │   │   ├─ StreamingToolExecutor 提前开始执行工具（不等待完整响应）
    │   │   └─ message_delta → 捕获 stop_reason, usage
    │   │
    │   │   错误恢复:
    │   │   ├─ 413 prompt-too-long → 触发 collapse drain / reactive compact
    │   │   ├─ max_output_tokens   → 递增 maxOutputTokensOverride 重试
    │   │   └─ 模型 fallback       → 切换模型重新请求
    │   │
    │   └─ yield 每个 assistant message / tool result 给调用方
    │
    │  【阶段 3: 判断是否继续】
    ├─ if (!needsFollowUp):            — 模型没有请求工具调用
    │   ├─ handleStopHooks()           — 执行 Stop Hook
    │   ├─ checkTokenBudget()          — Token budget 检查（+500k auto-continue）
    │   └─ return { reason: 'completed' }
    │
    │  【阶段 4: 执行工具】
    ├─ 两种路径:
    │   ├─ StreamingToolExecutor:      — 流式执行（部分工具已在模型输出时提前开始）
    │   └─ runTools():                 — 传统执行（等完整响应后统一执行）
    │
    │   runTools() 内部:
    │   ├─ partitionToolCalls()        — 按并发安全划分批次
    │   ├─ 只读工具 (Read, Grep, Glob) → 并发执行
    │   └─ 写操作工具 (Write, Edit, Bash) → 串行执行
    │
    │  【阶段 5: 后处理】
    ├─ generateToolUseSummary()        — 异步生成 tool use 摘要（Haiku）
    ├─ getAttachmentMessages()         — Memory 附件、文件变更通知
    ├─ pendingMemoryPrefetch.consume() — 消费预取的 memory 结果
    ├─ skillPrefetch.collect()         — 消费技能发现预取结果
    ├─ refreshTools()                  — 刷新 MCP 工具
    │
    │  【阶段 6: 准备下一次迭代】
    ├─ if (maxTurns && nextTurnCount > maxTurns): return
    ├─ state = {                        — 组装新的 State
    │     messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
    │     toolUseContext, turnCount: nextTurnCount,
    │     transition: { reason: 'next_turn' }
    │   }
    └─ continue                         — 回到 while(true) 顶部
  ```

###### 关键设计决策

- **流式工具执行（StreamingToolExecutor）**

  ```
  传统:  模型输出 ──────────────────▶ 工具1 ▸ 工具2 ▸ 工具3
  流式:  模型输出 block1 ▸ 工具1
         模型输出 block2 ▸ 工具2    ← 并行
         模型输出 block3 ▸ 工具3
  ```

  - **收益**：I/O 密集型工具（网络请求、文件读取）在模型还在生成后续 block 时就已经完成

- **多层压缩策略**

  ```
  Microcompact → Snip → Context Collapse → Auto Compact → Reactive Compact(轻)
  ```

  - **Microcompact**：无损删除冗余的 cache-edit 工具调用记录
  - **Snip**：按 token 预算裁剪历史

  - **Context Collapse**：智能折叠语义上完整的历史段落

  - **Auto Compact**：主动（proactive）摘要，在 token 耗尽前行动

  - **Reactive Compact**：被动（reactive）摘要，收到 413 错误后才触发

- **错误恢复链**

```
max_output_tokens 超限:
  1. 提升 maxOutputTokens 到 64k 重试（不修改消息）
  2. 注入 "continue where you left off" meta message 重试（最多 3 次）
  3. 放弃，将错误暴露给用户

prompt-too-long (413):
  1. 尝试 drain staged collapses（轻量，保留细节）
  2. 尝试 reactive compact（重量，生成摘要）
  3. 放弃，返回错误

模型 overload:
  1. 切换到 fallback 模型
  2. 清除本次的 tool_use/tool_result 重新请求
```

- **Memory 预取机制**：使用 `using` 关键字（TC39 Explicit Resource Management）在循环入口启动 memory 预取，后台异步加载相关 memory 文件，在工具执行完毕后消费结果，**不阻塞关键路径**。

- **多轮对话的 attachments 注入**每次工具执行完成后，向下一轮注入：

  - **Memory attachments** — 与当前上下文相关的 memory 文件

  - **技能发现** — 模型可能需要的技能提示

  - **文件变更通知** — 外部工具修改的文件

  - **任务通知** — 后台任务完成的通知

  - **依赖注入设计（QueryDeps）**

```typescript
type QueryDeps = {
  callModel: typeof queryModelWithStreaming  // API 调用
  microcompact: typeof microcompactMessages  // 微压缩
  autocompact: typeof autoCompactIfNeeded    // 自动压缩
  uuid: () => string                         // UUID 生成
}
```

通过 `params.deps` 注入，生产环境用 `productionDeps()`，测试环境注入 mock —— 避免 spyOn 的模块级副作用。

###### 数据流总结

```
用户按 Enter
    │
    ▼
┌────────────────────────────────────────────────────┐
│  QueryEngine.submitMessage(prompt)                 │
│                                                    │
│  System Prompt ─────┐                            	 │
│  User Context ──────┼──▶ 消息组装                   │
│  Skill 注入 ────────┘                               │
│                                                    │
│  ┌── query() loop ─────────────────────────────┐   │
│  │                                             │   │
│  │  压缩层(Microcompact→Snip→Collapse→Auto)     │   │
│  │         │                                   │   │
│  │         ▼                                   │   │
│  │  ┌──────────────┐     ┌────────────────┐    │   │
│  │  │  Claude API  │────▶│  Tool Executor  │   │   │
│  │  │  (streaming) │◀────│  (并行/串行)     │   │   │
│  │  └──────────────┘     └────────────────┘    │   │
│  │         │                    │              │   │
│  │         ▼                    ▼              │   │
│  │  assistant_messages    tool_results         │   │
│  │         │                    │              │   │
│  │         └────────┬───────────┘              │   │
│  │                  ▼                          │   │
│  │         needsFollowUp?                      │   │
│  │         ├─ YES → continue (下一轮迭代)        │   │
│  │         └─ NO  → return { reason }          │   │
│  └─────────────────────────────────────────────┘   │
│                                                    │
│  yield SDKMessage 流给调用方                         │
│  返回 result: { stop_reason, usage, cost, ... }     │
└────────────────────────────────────────────────────┘
```

###### 关键文件索引

| 文件 | 行数 | 职责 |
|------|------|------|
| `QueryEngine.ts` | ~1295 | 会话管理、SDK 接口、状态持有 |
| `query.ts` | ~1730 | 核心 Agentic Loop |
| `query/config.ts` | ~47 | 不可变配置快照（gates, sessionId） |
| `query/deps.ts` | ~41 | 依赖注入定义（callModel, compaction, uuid） |
| `query/stopHooks.ts` | ~400+ | Stop Hook 执行逻辑 |
| `query/tokenBudget.ts` | ~94 | Token 预算追踪与自动续跑 |
| `services/tools/toolOrchestration.ts` | ~ | 工具分区并发/串行执行 |
| `services/tools/StreamingToolExecutor.ts` | ~ | 流式工具执行器 |
| `services/api/claude.ts` | ~ | Anthropic API 调用封装 |
| `services/compact/autoCompact.ts` | ~ | 自动上下文压缩 |

###### Terminal 返回类型

`query()` 循环退出时返回一个 `Terminal` 对象，包含退出原因：

| reason | 含义 |
|--------|------|
| `completed` | 模型自然结束（无 tool_use） |
| `aborted_streaming` | 用户中断模型流式输出 |
| `aborted_tools` | 用户中断工具执行 |
| `max_turns` | 达到最大 turn 数限制 |
| `blocking_limit` | token 用量达到硬限制（auto-compact 关闭时） |
| `prompt_too_long` | prompt 过长，恢复失败 |
| `image_error` | 图片尺寸/格式错误 |
| `model_error` | 模型调用异常 |
| `stop_hook_prevented` | Stop Hook 阻止继续 |
| `hook_stopped` | Hook 阻止继续 |

Continue 类型记录了每次迭代间的过渡：

| reason | 含义 |
|--------|------|
| `next_turn` | 正常进入下一轮 |
| `collapse_drain_retry` | 上下文折叠排空后重试 |
| `reactive_compact_retry` | 反应式压缩后重试 |
| `max_output_tokens_escalate` | 提升 output token 上限重试 |
| `max_output_tokens_recovery` | 注入恢复消息重试 |
| `stop_hook_blocking` | Stop Hook 注入阻塞消息后重试 |
| `token_budget_continuation` | Token budget 自动续跑 |

#### Tools

###### 文件操作

- **FileReadTool**：读取文件内容
- **FileWriteTool**：写入文件（创建/覆盖）
- **FileEditTool**：精确字符串替换编辑
- **NotebookEditTool**：Jupyter Notebook (.ipynb) 编辑

###### 搜索与发现

- **GlobTool**：文件名/路径模式匹配
- **GrepTool**：代码内容正则搜索
- **ToolSearchTool**：MCP 工具搜索（按需启用）

###### 系统执行

- **BashTool**：执行 Shell 命令
- **PowerShellTool**：PowerShell 命令执行（条件启用）

###### 网络访问

- **WebFetchTool**：HTTP 请求，抓取网页内容
- **WebSearchTool**：网页搜索引擎查询

###### Agent 与多智能体

- **AgentTool**：启动子 Agent 处理复杂任务
- **TaskStopTool**：停止运行中的后台任务/Agent
- **SendMessageTool**：Agent 间消息通信
- **TeamCreateTool**：创建 Agent 团队（Agent Swarms）
- **TeamDeleteTool**：删除 Agent 团队

###### 任务管理

- **TaskCreateTool**：创建结构化任务
- **TaskGetTool**：获取任务详情
- **TaskUpdateTool**：更新任务状态/依赖
- **TaskListTool**：列出所有任务

###### 后台任务输出

- **TaskOutputTool**：获取后台运行任务的输出

###### 计划模式

- **EnterPlanModeTool**：进入计划模式（先探索再实现）
- **ExitPlanModeV2Tool**：退出计划模式，提交方案供审批

###### 用户交互

- **AskUserQuestionTool**：向用户提问（单选/多选）
- **TodoWriteTool**：写入结构化待办事项列表

###### Skill 与工作流

- **SkillTool**：调用斜线命令技能
- **WorkflowTool**：执行多 Agent 工作流脚本（条件启用）

###### MCP 协议

- **ListMcpResourcesTool**：列出 MCP 服务器资源
- **ReadMcpResourceTool**：读取 MCP 资源内容

###### Git 工作树

- **EnterWorktreeTool**：进入独立 Git 工作树
- **ExitWorktreeTool**：退出 Git 工作树

###### 触发器/调度

- **CronCreateTool**：创建定时/循环任务
- **CronDeleteTool**：取消定时任务
- **CronListTool**：列出所有定时任务
- **RemoteTriggerTool**：远程触发器（条件启用）

###### 内部与主动模式

- **BriefTool**：上下文简报摘要
- **SleepTool**：模型主动休眠等待（PROACTIVE / KAIROS）
- **MonitorTool**：后台监控（条件启用）

###### Anthropic 内部专属

- **REPLTool**：交互式 REPL（Python/JS 运行时）
- **ConfigTool**：运行时配置管理
- **TungstenTool**：内部工具
- **SuggestBackgroundPRTool**：后台 PR 建议

###### 通知/订阅（条件启用）

- **PushNotificationTool**：推送通知
- **SubscribePRTool**：订阅 PR Webhook

###### 实验性工具（条件启用）

- **WebBrowserTool**：浏览器自动化
- **TerminalCaptureTool**：终端输出捕获
- **LSPTool**：LSP 语言服务器集成
- **CtxInspectTool**：上下文检查调试
- **SnipTool**：对话历史裁剪
- **ListPeersTool**：列出同级 Agent
- **OverflowTestTool**：溢出测试
- **VerifyPlanExecutionTool**：计划验证
- **SendUserFileTool**：发送文件给用户

###### 测试专用

- **TestingPermissionTool**：权限测试（仅 `NODE_ENV=test`）

###### 合成输出（内部机制）

- **SyntheticOutputTool**：结构化输出强制（非模型直接调用）





## 附录

#### CLI Start Parameters

###### Session Management

- **claude --continue** / **claude -c**：会自动恢复当前目录最近的一次对话
- **claude --resume**：打开交互式选择器，浏览历史会话
- **claude --resume <session-id>**：直接恢复到指定会话
- **claude --resume "bug fix"**：搜索包含 "bug fix" 的会话
- **claude --resume <id> --fork-session**：恢复历史但创建新 session，不影响原始记录
- **claude --session-id <uuid>**：使用固定 session ID
- **claude -n "修复登录bug"** / **claude --name "修复登录bug"**：为本次会话命名（在 /resume 列表中可见）
- **claude --no-session-persistence**：本次不保存 transcript（仅 `-p` 模式）
- **claude --from-pr [number]**：从关联的 PR 恢复会话

###### Pipeline/Script Mode

- **claude -p "提示词"** / **claude --print "提示词"**：非交互模式，输出结果后退出（适合管道）
- **claude -p --output-format json "..."**：输出为 JSON 格式
- **claude -p --output-format stream-json "..."**：实时流式 JSON 输出
- **claude -p --input-format stream-json**：流式 JSON 输入
- **claude -p --json-schema '{...}' "..."**：要求模型按 JSON Schema 输出结构化结果
- **claude -p --max-turns 10 "..."**：限制最多 10 轮 agentic 循环
- **claude -p --max-budget-usd 5 "..."**：限制最多花费 $5
- **claude -p --replay-user-messages**：回显输入消息用于确认（配合 stream-json）

###### Permission Control

- **claude --permission-mode plan**：只读模式（仅允许 Read/Grep/Glob 工具）
- **claude --permission-mode accept-edits**：自动批准编辑，危险操作仍需确认
- **claude --permission-mode bypass**：跳过所有确认（仅限可信沙箱环境）
- **claude --dangerously-skip-permissions**：完全跳过权限检查（仅限离线沙箱）
- **claude --allow-dangerously-skip-permissions**：允许跳过权限但不默认启用

###### Tool & Context Control

- **claude --tools "Bash,Read,Edit"**：只暴露指定工具给模型
- **claude --tools ""**：禁用所有内置工具
- **claude --allowedTools "Bash(git:*)"** / **claude --allowed-tools "..."**：允许工具但限制参数范围（支持 glob 模式）
- **claude --disallowedTools "WebSearch,WebFetch"** / **claude --disallowed-tools "..."**：禁止特定工具
- **claude --add-dir ../other-project**：额外目录授权访问
- **claude --system-prompt "你是一个 Go 专家"**：替换系统提示词（仅 `-p` 模式）
- **claude --append-system-prompt "..."**：追加自定义系统提示词
- **claude --system-prompt-file /path/to/prompt.md**：从文件读取系统提示词
- **claude --disable-slash-commands**：禁用所有 skill

###### Model & Performance

- **claude --model sonnet**：使用 Sonnet 模型
- **claude --model opus**：使用 Opus 模型
- **claude --effort low**：设置推理努力程度（low / medium / high / max）
- **claude --fallback-model haiku**：主模型过载时自动降级到指定模型（仅 `-p` 模式）
- **claude --thinking enabled**：启用 extended thinking（enabled / adaptive / disabled）
- **claude --verbose**：详细输出模式

###### Debug & Dev

- **claude --debug**：启用调试日志
- **claude --debug "api,hooks"**：按类别过滤调试输出
- **claude --debug-to-stderr**：调试日志输出到 stderr
- **claude --debug-file /path/to/log**：调试日志写入文件
- **claude --verbose**：详细输出模式

###### Plugin & MCP

- **claude --mcp-config ./mcp.json**：加载 MCP 服务器配置
- **claude --mcp-config ./a.json ./b.json**：加载多个 MCP 配置文件
- **claude --strict-mcp-config**：仅使用 CLI 指定的 MCP，忽略所有其他 MCP 配置
- **claude --plugin-dir ./my-plugins**：从目录加载插件（可重复传入多个目录）
- **claude --settings ./custom.json**：加载额外设置文件
- **claude --setting-sources user,project,local**：指定加载哪些来源的配置

###### Agent & Team

- **claude --agent "reviewer"**：使用指定自定义 Agent
- **claude --agents '{"reviewer":{"description":"Reviews code","prompt":"..."}}'**：内联定义自定义 Agent
- **claude --betas "feature-1,feature-2"**：启用 Beta 功能（API key 用户）

###### Misc

- **claude --bare**：极简模式：跳过 hooks / LSP / 插件同步 / 自动 memory / CLAUDE.md 自动发现
- **claude --ide**：自动连接 IDE
- **claude --chrome** / **--no-chrome**：启用/禁用 Claude in Chrome 集成
- **claude --file file_id:relative_path**：启动时下载文件资源

#### Slash Command

###### Session Management

- **/resume**：浏览并恢复历史会话
- **/rename "新名称"**：重命名当前会话
- **/clear**：清除当前对话上下文（保留 session）
- **/compact**：手动触发上下文压缩

###### Configuration & Info

- **/config**：查看/修改配置
- **/cost**：查看当前会话的 API 花费
- **/context**：查看当前上下文 token 用量
- **/model**：切换模型
- **/effort**：设置推理努力程度
- **/fast**：切换 fast mode
- **/status**：查看当前状态
- **/doctor**：诊断环境问题

###### Version Control

- **/commit**：生成 commit message 并提交
- **/branch**：创建并切换到新分支
- **/diff**：查看当前改动
- **/pr** / **/commit-push-pr**：创建 Pull Request

###### Agent & Team

- **/agents**：管理自定义 Agent
- **/workflows**：查看/管理工作流

###### Code Quality

- **/review**：审查当前 PR 或 diff
- **/code-review**：代码审查
- **/security-review**：安全审查
- **/simplify**：简化/重构代码
- **/verify**：验证代码改动是否有效

###### Research

- **/deep-research**：深度调研主题

###### Tools & Integration

- **/mcp**：管理 MCP 服务器
- **/chrome**：启用 Claude in Chrome 集成
- **/ide**：连接 IDE
- **/add-dir**：添加授权目录
- **/plugin**：管理插件
- **/init**：初始化项目 CLAUDE.md

###### Debug & Dev

- **/debug**：切换调试日志
- **/debug-tool-call**：查看工具调用详情
- **/backfill-sessions**：回填历史 session 数据
- **/bughunter**：Bug 排查工具

###### Misc

- **/help**：显示帮助
- **/feedback**：提交反馈
- **/upgrade**：升级 Claude Code
- **/color**：切换颜色主题
- **/copy**：复制最后一条回复
- **/desktop**：与 Claude Desktop App 交互

#### Practical Recipes

```bash
# 场景1: 用 Claude 做代码审查（只读 + 限制花费）
claude -p --permission-mode plan --max-budget-usd 2 "审查 src/ 下最近的改动"

# 场景2: 脚本中批量处理
for file in src/*.ts; do
  claude -p --bare --output-format json \
    "检查 $file 是否有明显的 bug 或安全问题" >> audit.json
done

# 场景3: 恢复历史会话并 fork 出新分支
claude --resume --fork-session --name "继续重构 auth"

# 场景4: 在 CI 中限制模型行为
claude -p \
  --tools "Read,Grep,Glob" \
  --permission-mode plan \
  --max-turns 5 \
  --max-budget-usd 1 \
  "分析最近 commit 的代码质量"

# 场景5: 临时使用自定义配置
claude --settings ./ci-claude.json --mcp-config ./tools.json
```
