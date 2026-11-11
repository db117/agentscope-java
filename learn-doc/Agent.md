

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

## ReActAgent 核心流程

```mermaid
sequenceDiagram
    participant U as 调用方
    participant B as buildAgentStream
    participant F as buildIdentifiedAgentStream
    participant C as MiddlewareChain
    participant S as Flux.create / sink
    participant L as runLifecycle / ReAct
    U ->> B: 订阅 call() 或 streamEvents() 返回的 Publisher
    rect rgba(88, 128, 205, 0.10)
        B ->> B: Flux.deferContextual 准备运行上下文
        B ->> F: buildIdentifiedAgentStream(msgs, effective, doCallFn)
        F ->> C: build(ON_AGENT, ..., onAgent, core).apply(AgentInput)
        C -->> F: 返回 onAgent 包装后的 Flux
        F -->> B: 返回事件流
        opt 需要解析模型
            B ->> B: resolveModelAtEntry → fromCallable(getModel)<br/>.subscribeOn(boundedElastic)
        end
        rect rgba(68, 155, 140, 0.12)
            B ->> S: 直接订阅 events，或通过 thenMany(events) 订阅
            S -->> U: sink.next(AgentStartEvent)
            rect rgba(190, 136, 65, 0.14)
                S ->> L: contextWrite 传入上下文和事件出口<br/>手动 subscribe() 到生命周期 Mono
                L ->> L: 准备会话 → preCall → doCallFn
                loop ReAct 迭代
                    L ->> L: reasoning → 按需 acting → 下一轮或返回
                    L -->> S: publishEvent
                    S -->> U: sink.next(中间事件)
                end
                L ->> L: 保存状态 → postCall → 清理
                L -->> S: 最终 Msg
                S -->> U: sink.next(AgentResultEvent)
                L -->> S: onComplete → doFinally
                S -->> U: sink.next(AgentEndEvent) → sink.complete()
            end
        end
    end
```
