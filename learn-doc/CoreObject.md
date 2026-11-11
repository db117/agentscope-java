# 核心流程

## 对象

- Msg

> 最核心的对象，agents, users, and tools 直接沟通的主要单元

```mermaid
flowchart LR
    Msg["Msg<br/>对话消息基类"]
    Msg --> Fields["公共字段"]
    Fields --> Id["id：消息唯一标识"]
    Fields --> Name["name：发送者名称"]
    Fields --> Role["role：发送者角色"]
    Fields --> Metadata["metadata：附加信息"]
    Fields --> Timestamp["timestamp：创建时间"]
    Fields --> Usage["usage：模型用量"]
    Fields --> Content["content：内容块列表"]
    Msg --> Subtypes["消息子类型"]
    Subtypes --> User["UserMessage<br/>用户消息，USER"]
    Subtypes --> Assistant["AssistantMessage<br/>助手消息，ASSISTANT"]
    Subtypes --> System["SystemMessage<br/>系统指令，SYSTEM"]
    Subtypes --> ToolMsg["ToolResultMessage<br/>工具结果消息，TOOL"]
    Content --> Blocks["ContentBlock"]
    Blocks --> Text["TextBlock：文本"]
    Blocks --> Thinking["ThinkingBlock：思考内容"]
    Blocks --> ToolUse["ToolUseBlock：工具调用请求"]
    Blocks --> ToolResult["ToolResultBlock：工具执行结果"]
    Blocks --> Hint["HintBlock：提示信息"]
    Blocks --> Data["DataBlock：通用媒体数据"]
    Blocks --> Image["ImageBlock：图片，兼容类型"]
    Blocks --> Audio["AudioBlock：音频，兼容类型"]
    Blocks --> Video["VideoBlock：视频，兼容类型"]
```

- AgentState

> agent 当前”瞬时”运行状态的完整快照

```mermaid
flowchart LR
    AgentState["AgentState<br/>单个 Agent 可恢复的运行状态"]
    AgentState --> Identity["会话身份"]
    Identity --> SessionId["sessionId：会话标识，未提供时自动生成"]
    Identity --> UserId["userId：用户标识，可为空"]
    AgentState --> Conversation["对话内容"]
    Conversation --> Summary["summary：滚动摘要"]
    Conversation --> Context["context：对话消息列表 List&lt;Msg&gt;"]
    AgentState --> Run["当前运行信息"]
    Run --> ReplyId["replyId：当前回复标识"]
    Run --> CurIter["curIter：当前推理轮次"]
    Run --> Interrupted["shutdownInterrupted：是否因关闭而中断"]
    AgentState --> Permission["permissionContext：工具权限状态"]
    Permission --> Mode["mode：权限模式"]
    Permission --> WorkingDirs["workingDirectories：额外工作目录"]
    Permission --> Rules["allowRules / denyRules / askRules：工具规则"]
    AgentState --> Tool["toolContext：工具运行状态"]
    Tool --> ReadCache["readFileCache：文件读取缓存"]
    Tool --> CacheLimits["maxCacheFiles / maxCacheBytes：缓存上限"]
    Tool --> ActivatedGroups["activatedGroups：已启用的工具组"]
    Tool --> DiscoveredTools["discoveredTools：已发现的工具"]
    Tool --> SpawnRegistry["spawnRegistry：子 Agent 注册信息"]
    AgentState --> Tasks["tasksContext：任务状态"]
    Tasks --> TaskList["tasks：任务列表"]
    Tasks --> Requirements["requirements：任务要求"]
    Tasks --> Verifications["verifications：要求验证结果"]
    Tasks --> Scope["scope：任务标识、目标和来源"]
    Tasks --> TaskVersions["revision / contractVersion / subjectVersion：版本信息"]
    AgentState --> Plan["planModeContext：计划模式状态"]
    Plan --> PlanActive["planActive：计划模式是否开启"]
    Plan --> PlanFile["currentPlanFile：当前计划文件"]
```

- RuntimeContext

> 单次 Agent 调用的运行时上下文，不参与持久化

```mermaid
flowchart LR
    RuntimeContext["RuntimeContext<br/>单次 Agent 调用的运行时上下文"]
    RuntimeContext --> Identity["调用身份"]
    Identity --> SessionId["sessionId：会话标识"]
    Identity --> UserId["userId：用户标识"]
    Identity --> RunId["runId：本次调用的追踪标识<br/>未指定时自动生成"]
    RuntimeContext --> State["agentState：当前调用对应的 AgentState<br/>调用期间可用；调用外可能为空"]
    RuntimeContext --> Attributes["调用期间共享的扩展属性<br/>线程安全；不持久化"]
    Attributes --> StringAttrs["stringAttributes：按字符串 key 存取"]
    Attributes --> TypedAttrs["typedAttributes：按类型和可选 key 存取"]
    RuntimeContext --> Tool["工具相关配置"]
    Tool --> ExecContext["toolExecutionContext：工具依赖注入与上下文存储"]
    Tool --> RequestConfig["toolRequestConfig：本次调用的工具配置差异<br/>为空时使用 Agent 的共享工具集"]
    RuntimeContext --> Helpers["创建与转换"]
    Helpers --> Empty["empty()：创建空上下文"]
    Helpers --> From["builder(source).from(source)：派生上下文<br/>默认沿用 source 的 runId"]
    Helpers --> Resolve["resolveAgentState()：获取当前调用状态<br/>必要时回退到 Agent 状态"]
    Helpers --> AsTool["asToolExecutionContext()：将属性与工具存储合并"]

```

- AgentEvent

> 事件是整个框架处理的核心

```mermaid
flowchart LR
    AE["AgentEvent<br/>抽象事件基类"]
    AE --> Common["通用字段"]
    Common --> Id["id：事件唯一标识"]
    Common --> CreatedAt["createdAt：创建时间"]
    Common --> Type["type：事件类型"]
    Common --> Source["source：来源 Agent"]
    Common --> Execution["execution：执行身份"]
    Common --> Metadata["metadata：附加信息"]
    AE --> Lifecycle["Agent 生命周期"]
    Lifecycle --> Start["AgentStartEvent：运行开始"]
    Lifecycle --> End["AgentEndEvent：运行结束"]
    Lifecycle --> Result["AgentResultEvent：最终结果"]
    AE --> Control["运行控制"]
    Control --> MaxIters["ExceedMaxItersEvent：达到迭代上限"]
    Control --> Stop["RequestStopEvent：请求停止"]
    AE --> Model["模型调用"]
    Model --> ModelStart["ModelCallStartEvent：模型调用开始"]
    Model --> ModelEnd["ModelCallEndEvent：模型调用结束"]
    AE --> Content["内容块"]
    Content --> TextStart["TextBlockStartEvent：文本块开始"]
    Content --> TextDelta["TextBlockDeltaEvent：文本增量"]
    Content --> TextEnd["TextBlockEndEvent：文本块结束"]
    Content --> ThinkingStart["ThinkingBlockStartEvent：思考块开始"]
    Content --> ThinkingDelta["ThinkingBlockDeltaEvent：思考增量"]
    Content --> ThinkingEnd["ThinkingBlockEndEvent：思考块结束"]
    Content --> DataStart["DataBlockStartEvent：数据块开始"]
    Content --> DataDelta["DataBlockDeltaEvent：数据增量"]
    Content --> DataEnd["DataBlockEndEvent：数据块结束"]
    Content --> Hint["HintBlockEvent：提示信息"]
    AE --> Tools["工具调用与结果"]
    Tools --> CallStart["ToolCallStartEvent：工具调用开始"]
    Tools --> CallDelta["ToolCallDeltaEvent：工具调用增量"]
    Tools --> CallEnd["ToolCallEndEvent：工具调用结束"]
    Tools --> ToolResultStart["ToolResultStartEvent：工具结果开始"]
    Tools --> Progress["ToolProgressEvent：工具执行进度"]
    Tools --> ResultText["ToolResultTextDeltaEvent：文本结果增量"]
    Tools --> ResultData["ToolResultDataDeltaEvent：数据结果增量"]
    Tools --> ToolResultEnd["ToolResultEndEvent：工具结果结束"]
    Tools --> Denied["AllToolsDeniedEvent：工具调用均被拒绝"]
    AE --> External["用户确认与外部执行"]
    External --> Confirm["RequireUserConfirmEvent：请求用户确认"]
    External --> ConfirmResult["UserConfirmResultEvent：用户确认结果"]
    External --> RequireExec["RequireExternalExecutionEvent：请求外部执行"]
    External --> ExecResult["ExternalExecutionResultEvent：外部执行结果"]
    AE --> Other["子 Agent 与扩展"]
    Other --> Subagent["SubagentExposedEvent：子 Agent 事件"]
    Other --> Custom["CustomEvent：自定义事件"]
```

- MiddlewareBase

> 中间件接口

```mermaid
flowchart LR
    MB["MiddlewareBase<br/>中间件接口"]
    MB --> Control["参与与顺序控制"]
    Control --> Active["activePoints()<br/>选择启用的扩展点；默认全部启用"]
    Control --> Order["order()<br/>数值越大越靠外；默认 1"]
    MB --> Onion["环绕式拦截<br/>Flux&lt;AgentEvent&gt; + next"]
    Onion --> OnAgent["onAgent()<br/>拦截整次 Agent 调用"]
    Onion --> OnReasoning["onReasoning()<br/>拦截推理与模型流处理"]
    Onion --> OnActing["onActing()<br/>拦截工具执行"]
    Onion --> OnModelCall["onModelCall()<br/>拦截原始模型调用"]
    MB --> Pipeline["顺序变换"]
    Pipeline --> OnSystemPrompt["onSystemPrompt()<br/>依次变换 system prompt"]
    MB --> Notification["一次性通知"]
    Notification --> OnStateReady["onAgentStateReady()<br/>AgentState 就绪后、输入进入流程前通知"]
```
内置中间件

```mermaid
flowchart LR
    Root["内置 MiddlewareBase"]

    Root --> Core["agentscope-core"]
    Core --> Shutdown["GracefulShutdownMiddleware<br/>在安全检查点处理关闭与中断"]
    Core --> FinalFilter["FinalAnswerFilterMiddleware<br/>只输出最终推理轮的文本"]
    Core --> TaskReminder["TaskReminderMiddleware<br/>注入待办与任务要求提示"]
    Core --> ToolSearch["ToolSearchMiddleware<br/>按需发现可用工具"]
    Core --> DynamicSkill["DynamicSkillMiddleware<br/>按调用动态加载技能"]
    Core --> Otel["OtelTracingMiddleware<br/>记录 Agent、模型和工具调用的 trace"]

    Root --> Harness["agentscope-harness<br/>通过 HarnessRuntimeMiddleware 扩展"]
    Harness --> Conventions["ContextConventionsMiddleware<br/>注入框架上下文约定"]
    Harness --> Workspace["WorkspaceContextMiddleware<br/>加载工作区指令和参考资料"]
    Harness --> AtPath["AtPathExpansionMiddleware<br/>展开用户输入中的 @path 文件引用"]
    Harness --> Compaction["CompactionMiddleware<br/>压缩较长对话上下文"]
    Harness --> Eviction["ToolResultEvictionMiddleware<br/>将过大的工具结果移出上下文"]
    Harness --> Flush["MemoryFlushMiddleware<br/>在调用后提取并保存长期记忆"]
    Harness --> Maintenance["MemoryMaintenanceMiddleware<br/>定期整理长期记忆"]
    Harness --> Sandbox["SandboxLifecycleMiddleware<br/>管理沙箱会话生命周期"]
    Harness --> Trace["AgentTraceMiddleware<br/>记录推理与工具执行日志"]
    Harness --> Inbox["InboxMiddleware<br/>读取消息队列并注入会话"]
    Harness --> AsyncTool["AsyncToolMiddleware<br/>将长时间运行的工具转为后台任务"]
    Harness --> Subagents["SubagentsMiddleware<br/>提供托管子 Agent 能力"]
    Harness --> DynamicSubagents["DynamicSubagentsMiddleware<br/>按推理步骤动态加载子 Agent"]
    Harness --> Teams["TeamsMiddleware<br/>注入团队上下文并提供团队工具"]
    Harness --> HarnessSkill["HarnessSkillMiddleware<br/>管理动态技能目录与运行时加载"]
    Harness --> SkillUsage["SkillUsageMiddleware<br/>记录技能使用次数"]
    Harness --> SkillCurator["SkillCuratorMiddleware<br/>后台整理技能"]
    Harness --> PlanMode["PlanModeMiddleware<br/>计划模式下拦截修改性工具调用"]
```
