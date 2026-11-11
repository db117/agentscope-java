

## AgentBase
> 核心入口

```mermaid
sequenceDiagram
    actor Caller as 调用方
    participant AgentBase
    participant ReActAgent

    Caller->>AgentBase: call(msgs)
    AgentBase-->>Caller: 返回 Mono<Msg>
    Caller->>AgentBase: subscribe()，开始执行

    AgentBase->>AgentBase: runInContext()：准备运行上下文、control、gateKey
    opt gateKey 不为空
        AgentBase->>AgentBase: serializeOnKey()：等待同 key 的前序调用
    end

    AgentBase->>AgentBase: runLifecycleBody()：检查并启动运行，登记中断目标
    AgentBase->>ReActAgent: acquireCallExecution(...)
    ReActAgent->>ReActAgent: beforeAgentExecution()：准备本次 scope
    ReActAgent-->>AgentBase: 返回 scope

    AgentBase->>ReActAgent: doCall(inputs)（scope 随 Reactor Context 传递）
    ReActAgent-->>AgentBase: 返回结果 Msg

    AgentBase->>ReActAgent: releaseCallExecution(scope, status)
    Note over AgentBase,ReActAgent: status 为 completed、failed 或 cancelled；取消时等清理完成再释放同 key 队列
    ReActAgent-->>AgentBase: 清理完成

    AgentBase->>AgentBase: retire request，执行 afterAgentExecution()
    opt 调用未取消
        AgentBase-->>Caller: 返回结果或错误
    end
```

| 节点 | 源码入口 | 主要职责 |
|---|---|---|
| 调用方 | 外部节点；框架入口是 `io.agentscope.core.agent.AgentBase#call(List<Msg>)` | 调用并订阅 `Mono`；订阅后才开始执行 |
| AgentBase | `io.agentscope.core.agent.AgentBase#runInContext(...)`、`#runLifecycleBody(...)`、`#executeCallBody(...)`、`#serializeOnKey(...)`、`#acquireExecution()`、`#releaseExecution(...)` | 管理运行准入、排队、scope 生命周期和最终收尾 |
| ReActAgent | `io.agentscope.core.ReActAgent#acquireCallExecution(...)`、`#beforeAgentExecution(...)`、`#doCall(List<Msg>)`、`#releaseCallExecution(...)` | 代表具体 Agent 准备本次执行状态、运行逻辑并清理状态 |
