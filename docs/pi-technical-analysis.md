# Pi 技术特点与原理分析

> 分析基线：当前 `main` 工作区，提交 `90d863f7c`。
>
> 配套文档：[Pi 项目详细学习计划](./pi-learning-plan.md)。
>
> 阅读约定：本文以当前源码为准，引用以文件路径和导出符号定位；行号会随后续提交漂移。官方文档与源码不一致时，以源码为最终依据，并在文中标注。

## 1. 结论先行

Pi 是一个**最小、可扩展的 Agent harness**。它不训练或提供模型，而是协调以下资源完成一次任务：

```text
模型 Provider
  ↓ 统一流式消息与事件
pi-ai
  ↓ 模型请求 / thinking / tool call / usage
pi-agent-core
  ↓ Agent 状态、工具执行、steering/follow-up、turn/run 事件
pi-coding-agent
  ↓ Session、上下文投影、compaction、扩展、资源、安全边界
交互 TUI / print / JSON / RPC / SDK
  ↓
终端用户或宿主应用
```

最重要的四个技术特点是：

1. **统一而不抹平差异的模型层**：Provider、API 协议、模型元数据和统一事件流分离；通用调用可以跨模型，Provider 特有行为仍通过元数据和适配器保留。
2. **事件驱动的 Agent 循环**：模型流式响应、工具执行、队列和中断都以明确事件推进；`turn`、`run`、`agent_end`、`agent_settled` 是不同粒度。
3. **原始历史与模型上下文分离**：Session 是追加式 JSONL 树；branch、compaction 和 `context_edit` 只改变当前投影，不改写原始记录。
4. **同进程高权限扩展系统**：扩展可注册工具、命令、Provider、事件和 UI，组合能力强；代价是扩展与 Pi 拥有相同进程权限，信任机制不是沙箱。

还需要先澄清一个边界：仓库同时包含两套 Agent 运行时：

- **当前 CLI 主链路**：`coding-agent → pi-agent-core → pi-ai`，Session 使用 coding-agent 自己的 JSONL `SessionManager` 和 compaction。
- **独立实验运行时**：`pi-durable`，提供原子提交、可恢复任务、文档和多种 Storage。它调用 `pi-ai` 和 Chord 基础设施，但当前 `coding-agent`、`pi-agent-core`、`pi-ai` 都不依赖它。

因此，不能把 durable 的 `Conversation`、`Task`、`Document` 直接套到当前 CLI 主链路；两者的 Session 和 compaction 概念相似，实现边界不同。

## 2. Monorepo 架构与主链路

### 2.1 包职责

| 包 | 定位 | 是否在当前 CLI 主链路 |
|---|---|---|
| `@earendil-works/pi-ai` | 多 Provider 模型目录、认证、请求、协议适配、统一流式消息 | 是，模型访问层 |
| `@earendil-works/pi-agent-core` | 有状态 Agent 循环、工具执行、队列和事件 | 是，低层 Agent 运行时 |
| `@earendil-works/pi-coding-agent` | CLI、SDK、Session、上下文、扩展、内置工具、资源和模式 | 是，产品主入口 |
| `@earendil-works/pi-tui` | 终端组件、布局、输入、焦点、能力和差量渲染 | 是，交互模式 UI |
| `@earendil-works/pi-codemode` | QuickJS JavaScript 沙箱和宿主工具桥接 | 可选；内置扩展默认不激活 |
| `@earendil-works/pi-mcp` | MCP 客户端：stdio、Streamable HTTP、工具/资源、OAuth | 可选；由 MCP 配置启用 |
| `@earendil-works/pi-durable` | 实验性、提交优先、可恢复的 Conversation/Task/Document runtime | 否，独立运行时 |
| `@earendil-works/chord` | 契约对象、facet、热重载、RPC、复制状态、delta 基础设施 | 完整 Host 不在 CLI 主链；基础设施被 durable 使用 |
| `@earendil-works/pi-env` | Durable `ExecutionEnv` 的远程/SSH/daemon/PTY/listFiles 实现 | 否，durable 宿主可选后端 |
| `@earendil-works/pi-telemetry` | 后端中立 span 合约、noop/内存实现、一致性测试 | 可选；当前主链没有默认 exporter |
| `@earendil-works/pi-evals` | 私有开发评测工作区，直接驱动真实 coding-agent | 否，开发/CI 能力 |

构建脚本按 `chord → tui → telemetry → codemode → mcp → ai → durable → env → agent → coding-agent` 显式执行。这个顺序反映维护中的构建契约，不等于运行时调用图；例如构建顺序中有 durable，并不能证明 CLI 使用 durable。

### 2.2 当前 CLI 的真实依赖方向

```text
┌────────────────────────────────────────────────────────────┐
│ Interfaces                                                  │
│ Interactive TUI / print text / print JSON / RPC / SDK       │
└──────────────────────────────┬─────────────────────────────┘
                               │ 共用 AgentSession / Runtime
┌──────────────────────────────▼─────────────────────────────┐
│ pi-coding-agent                                             │
│ createAgentSession / AgentSessionRuntime                    │
│ ├─ AgentSession：run 调度、retry、compaction、事件、扩展      │
│ ├─ SessionManager：JSONL entry tree、leaf、projection       │
│ ├─ SettingsManager / ModelRuntime / ResourceLoader          │
│ ├─ ExtensionRunner：事件、工具、命令、Provider、UI            │
│ └─ 内置工具 / MCP / codemode / tool_search                  │
└──────────────────────────────┬─────────────────────────────┘
                               │ prompt / continue / events
┌──────────────────────────────▼─────────────────────────────┐
│ pi-agent-core                                               │
│ Agent：state、agentLoop、工具 preflight/execute、队列         │
└──────────────────────────────┬─────────────────────────────┘
                               │ StreamFn
┌──────────────────────────────▼─────────────────────────────┐
│ pi-ai                                                       │
│ Models / ModelRuntime → Provider → API adapter              │
│ 统一 Transcript、AssistantMessageEvent、usage、thinking       │
└──────────────────────────────┬─────────────────────────────┘
                               │
                    Anthropic / OpenAI / Google / ...
```

SDK 不是绕过 CLI 的另一套 Agent；它直接创建同一 `AgentSession`。RPC、JSON 和 print 也不是另一套 Session 逻辑，而是给同一核心增加进程 I/O 适配。

### 2.3 三层上下文必须分开

理解 Pi 时，始终区分：

1. **原始记录层**：Session JSONL 中所有 entry，包括其他分支、被省略的失败尝试、usage、label、扩展状态。
2. **当前投影层**：当前 leaf 到根的路径，经最新 compaction、branch 选择和 `context_edit` 得到的 entry/message 集合。
3. **Provider 请求层**：再经 `transformContext`、`convertToLlm`、图片开关、hidden declarations、forced prompt 和 Provider 自己的消息转换后，实际发给模型的消息。

一个具体例子：Session 文件有 100 条 entry，当前分支只经过其中 40 条；最新 compaction 又省略前 25 条；一次失败重试用 `context_edit` 省略 1 条；最终模型可能只看到 15 条左右的消息，而不是文件中的 100 条。

## 3. 统一模型层：`pi-ai`

### 3.1 Provider、API、Model 是三个维度

**问题**：同一个模型可能由多个服务提供；同一个服务也可能使用多种线协议。若把“服务”“模型”“协议”写成一个概念，兼容端点和动态目录会很难表达。

Pi 将其拆开：

- `provider`：认证、目录、运行时归属，例如某个上游服务或本地代理。
- `api`：消息如何编码、请求如何发送、响应如何解析，例如 Anthropic Messages 或 OpenAI Completions。
- `Model`：纯元数据，包含 `id/name/provider/api/baseUrl/input/cost/reasoning/contextWindow/maxTokens/compat/thinkingLevelMap` 等。

核心定义位于 `packages/ai/src/types.ts` 的 `BaseModel`、`Model`，Provider 组装位于 `packages/ai/src/models.ts` 的 `createProvider()` 和 `ModelsImpl`。

因此：

- 多个 Provider 可复用同一个 API adapter。
- 一个 Provider 可按模型选择不同 API。
- `models.json` 可以给兼容端点添加模型，而不必写 Provider extension。
- Provider extension 可只改认证、base URL、动态发现或流式行为。

`ModelsImpl.streamSimple()` 的顺序是：规范化上下文 → 创建 lazy stream → 按 `model.provider` 找 Provider → 异步应用认证和请求参数 → 按模型/API 调用具体 stream。

### 3.2 Transcript system message 表达动态 prompt 和工具

**问题**：工具或指令可能在会话中途变化。如果 Provider 请求只接受一个静态 `systemPrompt + tools`，就无法保留“什么时候添加/移除了哪个工具”的历史，也难以跨 Provider 重放。

Pi 把 prompt 和工具声明放进 transcript 的 system messages：

- 第一条 system message 声明初始 prompt 和工具。
- 后续 system message 可通过 `sections` 添加、替换或删除命名 prompt 段。
- `toolsAdded`/`toolsRemoved` 表达工具声明变化。
- 顺序重放这些 system messages 得到当前完整状态。

实现位于 `packages/ai/src/utils/transcript.ts`。

如果 Provider 不支持会话中途 system message，adapter 会将其折叠成开头的一条完整 system checkpoint。这样做的代价是可能使缓存前缀失效；换来的是统一 transcript 和动态工具能力。

### 3.3 统一流式事件协议

Provider 原生事件被归一化为：

```text
start
text_start / text_delta / text_end
thinking_start / thinking_delta / thinking_end
toolcall_start / toolcall_delta / toolcall_end
done
error
```

定义位于 `packages/ai/src/types.ts` 的 `AssistantMessageEvent`。

两个重要规则：

1. `contentIndex` 标识内容块。不同 text、thinking、tool call 块的事件可能交错，不能假设一个块的 start/delta/end 连续出现。
2. `partial` 是共享可变对象，不是历史快照。需要保存历史时，应在事件处理时复制需要的字段，不能收集 `partial` 引用。

`complete()`/`completeSimple()` 没有另一套非流式协议实现，而是消费同一事件流并等待最终 `AssistantMessage`。这使流式解析、工具参数处理和错误形状保持单一实现。

### 3.4 工具参数的多阶段验证

**问题**：模型会流式输出 JSON 参数。流中间结果通常是不完整 JSON，但 UI 又想尽早显示部分参数；最终执行时又必须防止截断、类型错误或非法字段造成真实副作用。

Pi 分阶段处理：

1. **流式阶段**：尽力解析不完整 JSON，仅用于展示和增量状态；解析失败不等于最终失败。
2. **toolcall_end 阶段**：要求参数文本是完整 JSON object；非法时记录 `argumentsError`，参数退化为 `{}`。
3. **Agent preflight 阶段**：用 TypeBox schema 转换和校验参数。
4. **扩展 `tool_call` 阶段**：可做权限、路径、命令等业务拦截。
5. **长度截断保护**：Assistant `stopReason: "length"` 时，即使参数碰巧可解析，也不执行整个工具批次，而是返回错误结果要求模型重新发出完整调用。

相关实现：

- `packages/ai/src/utils/json-parse.ts`：流式和最终 JSON 解析。
- `packages/ai/src/utils/validation.ts`：`validateToolArguments()`。
- `packages/agent/src/agent-loop.ts`：`prepareToolCall()`、`executeToolCalls()`。

所以“参数能显示”不等于“参数完整”；“JSON 能解析”也不等于“通过 schema 或被允许执行”。

### 3.5 Thinking/reasoning 的跨模型处理

统一 thinking level 是 `off/minimal/low/medium/high/xhigh/max`。Provider 可将它映射为 effort、budget tokens 或忽略，映射规则来自模型元数据和 adapter。

历史消息中的 thinking 具有 Provider 特征：

- 可读 thinking 文本。
- `thinkingSignature` 或 redacted payload。
- 文本和 tool call 的 Provider 签名。

跨模型或跨 API 重放时，Provider 签名通常不能再安全使用。`packages/ai/src/api/transform-messages.ts` 会按目标 `provider + api + model` 判断：

- 完全相同：保留有效 thinking 和签名。
- 不同：删除 thinking 和 Provider 签名，保留普通文本与工具调用。

错误或被 abort 的 Assistant 也不会无条件重放；转换层会跳过 `error/aborted` Assistant，并为缺少结果的 tool call 合成错误 ToolResult，避免把不完整结构发给下一次请求。

### 3.6 静态模型目录与动态刷新

Pi 同时存在两套模型来源：

- **静态生成目录**：`packages/ai/scripts/generate-models.ts` 汇总上游模型、人工修正、能力、价格、缓存和兼容信息，生成 `models.generated.ts`、各 Provider JSON 和 wrapper。生成过程有 manifest、文件 hash、结构 hash 和 staging/rollback。
- **运行时动态刷新**：`Models.refresh()` 处理本地服务或实时目录，支持持久化恢复、离线启动、并发刷新和 generation counter，防止旧刷新覆盖新状态。

静态目录解决“发布时可类型检查、可离线使用”的问题；动态刷新解决“本地模型和实时服务变化”的问题。两者不是重复设计。

根入口刻意轻量，Provider/API adapter 通过 lazy wrapper 延迟加载；`providers/all.ts` 是显式重型入口。这让 SDK 或小型宿主能控制包体和启动成本。

## 4. 低层 Agent 循环：`pi-agent-core`

### 4.1 Agent 状态与事件

`AgentState` 包含：

- 当前 transcript 重放出的 system prompt。
- 当前模型和 thinking level。
- 可执行工具。
- 消息数组。
- `isStreaming`、`streamingMessage`、`pendingToolCalls`、`errorMessage`。

定义位于 `packages/agent/src/types.ts`，状态封装位于 `packages/agent/src/agent.ts`。

事件包括：

- Agent 层：`agent_start`、`agent_end`。
- Turn 层：`turn_start`、`turn_end`。
- Message 层：`message_start/update/end`。
- Tool 层：`tool_execution_start/update/end`。

`Agent` 的监听器是 await 的。内部状态先更新，再调用监听器；因此高层 `AgentSession` 可以把某些监听器当作持久化或扩展处理 barrier。

### 4.2 一次正常 tool-call 循环

以“读取一个文件，再根据内容回答”为例：

```text
agent_start
turn_start
message_start/end(user)
message_start(assistant)
message_update(text/toolcall deltas...)
message_end(assistant: read(path))
tool_execution_start(read)
tool_execution_update(read)       // 可选
tool_execution_end(read)
message_start/end(toolResult)
turn_end
turn_start                      // tool result 触发续问
message_start(assistant)
message_update(text deltas...)
message_end(assistant)
turn_end
agent_end
```

源码路径是：

1. `Agent.prompt()` 创建 UserMessage 和当前状态快照。
2. `runAgentLoop()` 写入 prompt，发出 `agent_start/turn_start/message_*`。
3. `prepareRequest()` 执行 transform 和 LLM 消息转换。
4. `streamAssistantResponse()` 消费 pi-ai 事件并发出 Agent 事件。
5. Assistant 完成后过滤 toolCall。
6. `executeToolCalls()` 校验、拦截并执行工具。
7. ToolResult 按模型调用顺序写入消息。
8. `finishTurn()` 和 `turn_end` 后决定下一 turn 或结束。

关键实现位于 `packages/agent/src/agent-loop.ts`。

### 4.3 Turn、run 与队列语义

- **turn**：一次 Assistant 响应，加上该响应产生的工具调用和工具结果。
- **低层 run**：`Agent.prompt()` 或 `Agent.continue()` 驱动的一轮循环，可包含多个 turn。
- **steering**：当前 Assistant turn 及其工具完成后注入，用于改变当前任务方向。
- **follow-up**：没有更多工具、没有 steering、原本准备停止时才注入。

活动 run 中不能直接再次 `prompt()`；调用方必须明确选择 `steer()` 或 `followUp()`。这避免系统猜测用户想“打断当前方向”还是“追加后续工作”。

每种队列支持：

- `one-at-a-time`：每次 drain 最老一条。
- `all`：一次 drain 全部。

### 4.4 工具并发、结果顺序与提前结束

工具默认并行，但如果：

- Agent 配置为 sequential，或
- 任一目标工具声明 `executionMode: "sequential"`，

整个批次顺序执行。

并行路径的实际顺序是：

1. 按模型输出顺序发出 `tool_execution_start`。
2. 按同一顺序做工具查找、参数准备、schema 校验和 `beforeToolCall`。
3. preflight 完成后并发执行允许运行的工具。
4. `tool_execution_end` 按实际完成顺序发出。
5. 所有工具结束后，ToolResult 消息按模型原始 tool call 顺序持久化。

这同时保留了真实执行进度和可重放的 transcript 顺序。

`terminate: true` 是批次级提示：只有当前批次所有最终工具结果都同意 terminate，才抑制由该工具批次触发的自动下一请求。它不是“某个工具立即杀死 Agent”。

### 4.5 动态工具声明

Agent 同时维护：

- 运行时实际可执行的 `AgentTool` 集合。
- transcript 中声明给模型的工具状态。

每次请求前，`declareToolChanges()` 比较二者，将差异写成 system message 的 `toolsAdded/toolsRemoved`。同名但定义变化会表示为“删除旧定义 + 添加新定义”。

这就是 `tool_search`、MCP deferred 工具和扩展动态工具能跨 resume/branch 恢复的基础：工具变化本身也记录在会话历史中。

### 4.6 Abort 是协作式取消

`Agent.abort()` 只 abort 当前 run 的 `AbortController`。该 signal 会传给 Provider、hooks 和工具执行，但没有强制杀死工具 Promise 的机制。

因此：

- 遵守 signal 的网络请求和 backoff 可较快停止。
- 忽略 signal 的工具可能继续运行。
- 需要硬超时时，应由工具实现、外层执行环境或进程/容器边界提供。

## 5. 会话级运行时：`pi-coding-agent`

### 5.1 `createAgentSession()` 如何组装依赖

`packages/coding-agent/src/core/sdk.ts` 的 `createAgentSession()` 按以下方向创建或接收：

```text
cwd / agentDir
  → ModelRuntime
  → SettingsManager
  → SessionManager
  → ResourceLoader
  → Agent(streamFn, tools, model, thinking, queues, transforms)
  → AgentSession
```

可注入边界包括：

- `modelRuntime`、`model`、`thinkingLevel`、`scopedModels`。
- `settingsManager`。
- `sessionManager`。
- `resourceLoader`。
- `tools/noTools/excludeTools/customTools`。

默认工具选择是 `read/bash/edit/write`。`tools` 普通名称是 allowlist；全是 `+name/-name` 时是默认集 modifier，两者不能混用。

构造 `AgentSession` 后，前端或宿主还要调用 `bindExtensions()` 绑定 UI、mode、命令动作、错误处理和 shutdown；`session_start` 在该绑定中发出，而不是 loader 或 `createAgentSession()` 自动保证。

多会话宿主可使用 `AgentSessionRuntime`：切换、resume、fork 或 import 时先 abort/settle 旧会话，发出 shutdown，dispose，再创建新 runtime 并重新绑定订阅。

### 5.2 AgentSession 是比低层 Agent 更宽的生命周期

低层 `Agent.prompt()` 返回只表示一个低层 run 结束。`AgentSession` 还负责：

- 输入命令、prompt template、skill 展开。
- provider auth 预检。
- 自动 turn retry。
- overflow compaction 和重试。
- 队列消息。
- before-settle 扩展边界。
- 最终 `agent_settled`。

所以：

- `agent_end` 后可能还有自动重试、压缩恢复或 follow-up。
- `session.isStreaming` 可能比 `agent.state.isStreaming` 更久。
- 需要“不会再自动继续”的信号时，应使用 `agent_settled` 或 `waitForIdle()`。

### 5.3 会话层 prompt 处理顺序

普通 `session.prompt()` 的核心顺序是：

1. 若以 `/` 开头，先尝试扩展命令；命中则立即执行，不进入模型。
2. 执行扩展 `input` handlers，可 transform 或 handled。
3. 展开 `/skill:name` 和 prompt templates。
4. 若正在 streaming，必须显式选择 `steer` 或 `followUp`。
5. 非 streaming 时检查模型和认证。
6. 设置会话级 busy。
7. 发出 `before_agent_start`，允许扩展改 prompt、工具和追加 custom messages。
8. 应用 prompt/tool loadout。
9. 调用低层 `agent.prompt()`。
10. 低层 run 结束后，处理 retry、compaction、queued work。
11. 运行 `agent_before_settle`。
12. 发出 `agent_settled`。

busy 标记在 `before_agent_start` 前设置，因此扩展处理期间提交的消息会进入当前 run 队列，而不是启动第二个 run。

### 5.4 自动重试有三套，不能混为一谈

| 层级 | 位置 | 默认行为 | 作用 |
|---|---|---|---|
| Provider 请求级 | `pi-ai` / request options | 通常 `maxRetries: 0` | 对 408/429/5xx、网络错误等按 HTTP 语义等待重试 |
| Assistant turn 级 | `AgentSession` | 默认启用，最多 3 次 | 对可重试 Assistant error 做指数退避，并用 `context_edit` 省略失败响应 |
| 摘要级 | compaction/branch summary | 使用 summarization retry | 对摘要请求失败做有限重试 |

上下文溢出不走普通 turn retry，而是走“省略失败尝试 → compaction → compact-and-retry”。该恢复通常限制为一次，避免无限压缩和重试。

### 5.5 持久化与事件的顺序边界

`Agent` 在调用监听器前先更新 `agent.state.messages`。`AgentSession` 的处理顺序大致是：

1. 底层事件到达。
2. 扩展事件处理和替换。
3. 公共 `AgentSessionEvent` listener。
4. `message_end` 后写 SessionManager。

因此公共 `message_end` listener 被调用时，Agent 状态已包含该消息，但普通消息的 SessionManager append 可能尚未完成。自定义消息某些立即 append 路径顺序相反：先写 SessionManager、刷新投影，再合成公共事件。扩展或宿主不能假设所有消息路径的持久化时点完全一致。

`SessionManager` 是持久化上下文的权威来源。每次 Provider 请求前，`AgentSession` 会用 `buildSessionProjection()` 重建 canonical context；compaction、tree、custom append 后也会刷新投影并替换 `agent.state.messages`。

## 6. Session 树与上下文投影

### 6.1 问题：既要可回退，又不能丢历史

**问题**：线性聊天记录很难同时支持“回到旧消息重新提问”“保留被放弃方案”“压缩上下文”和“审计失败尝试”。

例如用户在 `U1 → A1 → U2a → A2a` 后回到 `A1`，改问 `U2b`：

```text
U1 ─ A1 ─ U2a ─ A2a   被放弃的探索
     └─ U2b ─ A2b     当前活动分支
```

如果只保存线性记录，两个 `U2` 无法共存；如果直接删除 `U2a`，又丢失探索和审计信息。

### 6.2 方案：追加式 JSONL 树

Session 文件是 JSONL：

- 第一条有效记录是 `session` header，包含版本、session ID、timestamp、cwd、可选 parentSession。
- 后续每条 entry 有 `id/parentId/timestamp`。
- `parentId: null` 是根。
- 新 entry 追加到文件末尾，并把内存 leaf 移到该 entry。
- branch 通常只移动 leaf，不改写旧 entry。

当前格式版本是 3。旧版本会迁移到树结构和 `custom` 消息角色。

核心实现位于 `packages/coding-agent/src/core/session-manager.ts`。

### 6.3 从当前 leaf 到模型上下文

`buildSessionProjection()` 的步骤是：

1. 从当前 leaf 沿 `parentId` 回溯到根，得到当前原始路径。
2. 从完整路径恢复模型和 thinking level。
3. 找路径中最新 compaction。
4. 没有 compaction：完整路径是候选上下文。
5. 有 compaction：候选上下文为“compaction + firstKeptEntryId 起的保留尾部 + compaction 后的路径”。
6. 去掉保留尾部中的旧 system message，避免与 compaction checkpoint 重复。
7. 应用最新有效 `context_edit`。
8. 将 entry 转为消息并保留 source entry 映射。

因此 `getBranch()` 返回原始路径，`buildSessionProjection()` 返回模型可见投影，二者不是同一 API。

### 6.4 `context_edit` 是追加式覆盖

**问题**：重试前的失败 Assistant、溢出的工具结果或过期内容不应继续影响模型，但直接删除原始消息会破坏审计和导出。

`context_edit` 追加一条新 entry：

- `targetId` 指向当前活动分支上的 user/assistant/toolResult/custom message。
- `replacement: null`：未来上下文省略目标。
- 非空 replacement：只替换 content，保留原 role 和元数据。

同一目标的最新 edit 获胜；切到 edit 之前的分支，原内容重新可见。原始 entry、UI、导出和 usage 统计不被改写。

### 6.5 System checkpoint 解决跨 compaction 恢复

如果 compaction 只保存摘要，后续恢复可能不知道当时有哪些 prompt sections 和工具声明。追加 compaction 时，Pi 会把压缩前的当前 system message 快照存入 compaction entry。重建时先重放该 checkpoint，再读取保留尾部和后续消息。

这是用少量冗余换取恢复确定性，避免依赖已经被摘要裁掉的早期 system entry。

### 6.6 当前实现的持久化限制

当前 `SessionManager` 更接近单写者同步追加模型：

- 普通追加用同步 `appendFileSync()` 写完整一行。
- 新文件用排他创建防止覆盖。
- 没有 session 文件锁、跨进程事务或临时文件 + rename。
- 多 entry 操作不是磁盘事务。
- 坏 JSONL 行会跳过；末行写入中断通常可恢复，但中间坏行也可能被静默忽略。
- 只有出现第一条 user/assistant message 时才延迟创建 session 文件，避免生成空文件。
- 当前 leaf 不单独持久化；重新打开时以文件最后一条物理记录所在分支为起点。纯导航而不追加新 entry 的位置不保证跨重启恢复。

这些限制是格式简单、易导出、易审计与强并发/事务保证之间的取舍。需要原子任务恢复时，应关注独立的 `pi-durable`，而不是把 JSONL Session 当成多写者数据库。

## 7. Compaction 与 branch summary

### 7.1 问题：上下文会超窗，但原始历史仍有价值

**问题**：模型上下文有限，长任务中的工具输出尤其庞大；直接截断旧消息会丢失目标、决策和文件状态。

例如一个 128k context 模型，`reserveTokens = 16384`，当估算上下文超过 `128k - 16k = 112k` 时，需要腾出空间；但最近 20k token 内的进行中工作又应尽量保留原文。

### 7.2 触发条件与位置

默认参数：

```text
enabled = true
reserveTokens = 16384
keepRecentTokens = 20000
```

触发公式是严格大于：

```text
contextTokens > contextWindow - reserveTokens
```

自动检查分布在多个边界：

- 新 prompt 前。
- 低层 run 结束后。
- 下一 Assistant response 前。
- 请求投影时，尤其虚拟模型路由到不同 context window 的物理模型后。

Provider context overflow 和可恢复的 `stopReason: "length"` 会走 compact-and-retry；已成功但 usage 显示超窗的响应只压缩，不会重放已经完成的响应。

### 7.3 切点算法

`prepareCompaction()` 基于当前 canonical projection，而不是原始文件全量历史：

1. 从最新 entry 向前累计估算 token，达到 `keepRecentTokens`。
2. 在附近找合法 cut point。
3. 可以在 user、assistant、bash/custom/branch/compaction summary 等位置切。
4. 不在 toolResult 处切，避免保留一个没有 tool call 的孤立结果。
5. 若切在带 tool calls 的 Assistant，其后续 tool results 一起保留。
6. 之前的 compaction 摘要作为 `previousSummary` 参与增量更新，不作为普通历史重复总结。

如果切点落在一个超长用户任务 span 内部，Pi 会生成两份摘要：

- 旧历史摘要。
- 当前 span 前缀摘要。

然后把前缀摘要与保留的当前 span 后续原始消息合并，形成 split-turn checkpoint。

### 7.4 摘要是有损但带保护

摘要请求不会把原始 transcript 当对话继续，而是：

1. 转成 LLM 消息。
2. 序列化为 `[User]`、`[Assistant tool calls]`、`[Tool result]` 等标签文本。
3. 包进独立 summarization prompt。
4. 要求结构化输出。

保护规则：

- thinking 被省略。
- 单个 tool result 最多保留约 2000 字符。
- 摘要请求禁止 prompt cache 写入。
- `length`、tool call、空文本或 error 都不会保存为有效 checkpoint。
- 临时错误走摘要重试。

内建 compaction 还提取 `read/write/edit` 及嵌套工具的文件路径，累积生成：

- `<read-files>`
- `<modified-files>`

这是在“摘要必有损”的前提下，对最关键文件状态的额外防丢机制。

### 7.5 Compaction 不等于删除

Compaction 追加 entry，包含：

- summary
- `firstKeptEntryId`
- `tokensBefore`
- system checkpoint
- usage
- details

旧 entries 不删除，Session 文件不会因为 compaction 变小。它改变的是后续请求的投影，不是物理历史。

### 7.6 Branch summary 是另一种摘要

使用 `/tree` 离开一个分支时，Pi 可总结从公共祖先到旧 leaf 的被放弃路径，并把 `branch_summary` 附到目标分支：

```text
A ─ B ─ C ─ D     旧 leaf，被放弃
└─ E ─ F ─ S(B,C,D)  新分支获得摘要
```

它与 compaction 的差别：

- compaction 在当前活动路径中腾出上下文。
- branch summary 把“离开的分支”压缩给“进入的分支”。
- branch summary 默认跳过 toolResult，直接读取 abandoned raw entries；当前实现不会应用这些 raw entries 上的 `context_edit`。
- 摘要失败时通常尚未完成导航切换。

## 8. 扩展系统与资源机制

### 8.1 问题：不同需求需要不同最小能力

Pi 没有把所有自定义都做成代码插件，而是按能力分级：

| 需求 | 最小机制 | 原因 |
|---|---|---|
| 给项目长期指令 | `AGENTS.md`/上下文文件 | 不需要可执行代码 |
| 复用一段输入 | Prompt template | 只需要文本展开和参数 |
| 按需加载专业说明和文件 | Skill | 避免全文常驻上下文 |
| 增加工具、命令、事件或 Provider | Extension | 需要执行行为 |
| 修改终端交互 | TUI 扩展点 | 需要组件、焦点和渲染 |
| 分发一组资源 | Pi package | 需要安装、依赖和过滤 |
| 接入外部工具协议 | MCP | 复用标准客户端/服务器生态 |

这体现“最小权限原则”的产品设计：不要为了一段可复用文本加载可执行代码。

### 8.2 Extension 加载模型

扩展是 TypeScript/JavaScript 模块，默认导出 factory：

```ts
export default function (pi: ExtensionAPI) {
  // 注册工具、命令、事件...
}
```

加载特点：

- 通过 jiti 加载，本地 TS 不需要手动编译。
- 扩展路径逐个串行加载，顺序确定。
- factory 可 async，Pi 会等待它完成。
- factory 主要用于注册；不应启动 watcher、server、timer 等长期资源。
- 加载失败只丢弃该扩展注册项，但无法回滚 factory 已产生的外部副作用。
- 模块缓存的是 factory；reload 会重新执行 factory，但不是新进程。

核心实现位于：

- `packages/coding-agent/src/core/extensions/loader.ts`
- `packages/coding-agent/src/core/extensions/runner.ts`
- `packages/coding-agent/src/core/extensions/types.ts`

虚拟模块把 Pi API 和 TypeBox 映射到宿主已加载的同一份模块，避免扩展和宿主各有一份类、registry 或初始化状态。这也意味着扩展不是独立依赖沙箱。

### 8.3 Project trust 是两阶段编排

**问题**：项目目录可以包含 `.pi/extensions`、`.pi/mcp.json` 等可执行配置。若启动时直接加载，克隆一个恶意仓库就可能执行代码。

Pi 使用两阶段：

1. pre-trust：只加载用户级、CLI 和 inline extensions，排除项目资源和内置扩展。
2. 解析 project trust：CLI override、扩展 `project_trust`、保存的决定、默认策略。
3. 根据结果重新读取 settings，加载最终资源集。

关键边界：

- 直接调用底层 extension discovery/loader 不会自动获得信任保护；安全来自 `DefaultResourceLoader + project-trust` 编排。
- `AGENTS.md`/`CLAUDE.md` 上下文文件不受项目 trust 控制，会作为不可信内容进入 prompt。
- 项目 `sessionDir` 是早期例外：为定位 session，会在完整信任决定前读取。
- 信任通过后，扩展与 Pi 同进程同权限；它不是运行时 sandbox。

### 8.4 Extension API 的主要扩展点

- 生命周期和消息事件：`pi.on()`。
- 模型可调用工具：`pi.registerTool()`。
- `/` 命令：`pi.registerCommand()`。
- 快捷键和 CLI flag：`registerShortcut()`、`registerFlag()`。
- Provider：`registerProvider()`。
- MCP server：`registerMcpServer()`。
- 每请求模型路由：`registerVirtualModel()`。
- 渲染：message、entry、tool renderer。
- 扩展间通信：共享 `pi.events`。
- 状态：tool result `details`、custom entry、custom message、外部存储。

工具结果分三层：

- `content`：发给模型的文本/图片。
- `details`：UI 渲染和 Session 恢复用，不自动发给模型。
- `structuredContent`：配合 `outputSchema`，供 codemode 等程序调用方使用。

嵌套工具调用通过 `ctx.executeTool()` 重新进入同一参数校验和 `tool_call/tool_result` 管线；结果不形成独立 transcript tool call，只以有界 `nestedCalls` 元数据记录到父结果。

### 8.5 事件顺序和错误语义

事件 handler 按扩展加载顺序和注册顺序执行。每次分发先快照 handler 列表，因此本次分发中的订阅变化不影响当前列表。

错误语义不是统一 fail-open：

- 普通观察事件：单 handler 抛错通常报告后继续。
- `context`、`input`、`message_end`、`tool_result`：保留此前有效结果，后续 handler 继续。
- `tool_call`：handler 异常会阻止工具执行，属于权限场景的 fail-closed。
- `session_before_*`：显式 `cancel` 才短路；普通错误不取消操作。
- `user_bash`：报告后重新抛出，不静默忽略。

冲突规则也不统一：

- 工具：跨扩展第一个注册者获胜。
- 快捷键：后加载扩展获胜，保留键不可覆盖。
- 重名命令：改为 `/name:1`、`/name:2`。
- `replaceable` 内置扩展可被同名工具/命令/flag 的第三方扩展替换，但其 factory 仍会先执行，外部副作用不能自动回滚。

### 8.6 Skills、prompt templates、themes、packages

**Skills** 使用 progressive disclosure：启动时只把名称、描述和路径注入系统提示；模型判断相关后再读取 `SKILL.md` 全文。这样避免所有专业说明常驻上下文。`disable-model-invocation` 只关闭自动路由，不禁止 `/skill:name`。

**Prompt templates** 在输入成为 user message 前展开，支持 `$1`、`$@`、默认值和参数切片。它们不注册工具或事件。

**Themes** 是终端颜色资源。`system` 主题根据终端前景、背景、ANSI palette 和对比规则动态生成；普通自定义主题经 schema 校验并支持变量、OKHSL/OKLCH、终端默认色和 HTML export 覆盖。

**Pi packages** 把 extensions、skills、prompts、themes 作为 npm/git/local 包分发。资源解析综合 settings、自动发现、包 manifest、过滤规则和优先级。同名 skills/prompts/themes 通常第一个胜出；把路径放到后面不能覆盖已有同名资源。

## 9. MCP、codemode 与动态工具发现

### 9.1 问题：工具数量和输出都可能超过上下文

**问题**：外部 MCP server 可能提供几十上百个工具。若全部声明给模型，工具描述会挤占上下文；若每个大结果都直接进入 transcript，也会迅速耗尽窗口。

Pi 提供三种互补机制：

1. `direct`：少量高频工具直接声明。
2. `deferred + tool_search`：需要时搜索并激活。
3. `codemode`：模型写一段 JavaScript，在沙箱中调用工具、过滤结果，只把摘要返回给模型。

### 9.2 MCP 客户端和工具适配

`pi-mcp` 是 MCP 客户端，不是 MCP server 框架。支持：

- stdio。
- Streamable HTTP；旧式独立 SSE transport 不支持。
- tools/list、tools/call、分页、annotations、structuredContent、进度、取消。
- resources/list/templates/read。
- OAuth discovery、PKCE、动态注册、refresh、loopback callback。

coding-agent 将工具注册为：

```text
mcp__<server>__<tool>
```

非法字符转为 `_`，超长或冲突时追加稳定 hash。项目 `.pi/mcp.json` 只在信任后读取；项目 override 可以启停或改 exposure，但不能夹带全局服务器的底层 command/url/auth。

MCP 工具仍走标准工具管线：参数校验、`tool_call/tool_result` hooks、权限扩展、nested call 记录都适用。annotations 是服务器声明的提示，不是 Pi 已验证的安全事实。

### 9.3 MCP exposure

| Exposure | 行为 | 适用 |
|---|---|---|
| `direct` | 立即声明给模型，也可被脚本调用 | 少量、高频工具 |
| `codemode`（默认） | 不直接声明；脚本通过 `searchTools()/ALL_TOOLS/describeTool()` 发现和调用 | 大型工具集、需要组合/过滤 |
| `deferred` | 不声明；`tool_search` 找到后激活，之后直接调用 | 大工具集但希望直接调用 |
| `hidden` | 注册但不可达 | 禁用或只允许特定覆盖 |

`toolExposure` 可按具体工具覆盖服务器级 exposure。

### 9.4 Codemode 执行原理

**问题**：若模型需要读取 20 个文件再求总数，直接工具调用会产生 20 个 transcript 结果；其中大部分内容对最终决策无用。

codemode 让模型返回一段 JavaScript：

1. coding-agent 注册一个 `codemode` 工具，输入为 `{ code: string }`。
2. 脚本作为 async 函数体在 QuickJS WASM 中执行，支持顶层 `await/return`。
3. Node/Bun 下每次执行通常使用 worker thread 和新的 QuickJS 实例。
4. VM 不直接提供 Node、文件系统、网络或 timers。
5. 脚本只能通过 `tools.<name>(args)` 请求宿主工具。
6. 宿主把调用重新送入 `ctx.executeTool()` 和标准工具管线。
7. 只有脚本的 `return/text()/image()/console` 汇总结果进入外层 tool result。

因此它减少的是**模型可见中间上下文**，不是工具副作用。

### 9.5 Codemode 边界

- QuickJS heap 默认 256 MiB。
- 全进程最多两个 codemode worker。
- coding-agent 对普通脚本默认不设置总 timeout；源码首行可声明 `timeout_ms`。
- 默认输出预算约 10000 tokens，超出后中部截断并保存全文到临时文件。
- 脚本失败不回滚已经完成的工具调用。
- `store()/load()` 只在脚本成功时写入当前 Session 分支的 custom entry，分支间状态隔离。
- 模型 API 信息会去除敏感 headers，调用按 provider/id 从 ModelRegistry 重新解析，避免脚本伪造端点带走凭据。

所以 QuickJS 是脚本运行隔离，不是操作系统权限沙箱。被授予的工具仍可在宿主侧读写文件、执行命令或访问 MCP 服务。

### 9.6 `tool_search`

`tool_search` 搜索尚未 active 的 codemode/deferred 工具，索引名称、描述、参数说明、namespace 和 instructions，用 BM25 排名。匹配工具被加入 active set，下一次模型请求开始声明；该变化写入 transcript，因此可随 resume、fork、tree 恢复。

它是“先发现，再直接调用”；codemode 是“脚本中间接调用并过滤结果”。二者可独立使用，也常与 MCP 默认 exposure 组合。

## 10. 四种 CLI 接口与 SDK

### 10.1 模式选择

CLI 入口解析后按以下优先级选择：

1. `--mode rpc`
2. `--mode json`
3. `--print` 或 stdin/stdout 任一不是 TTY → print
4. 否则 interactive

`--mode text` 不强制一次性执行；明确的一次性文本输出使用 `--print`。RPC 拒绝 `@file`，因为 stdin 保留给 JSONL 命令。

### 10.2 Print text 与 print JSON

Print text：

- 顺序执行所有初始 prompt。
- 只输出最后一条 Assistant 的文本。
- error/aborted/length 会写 stderr 并产生非零退出状态。

Print JSON：

- 先输出 Session header，再逐行输出事件。
- Assistant error 出现在事件流中，不单独按 text 模式规则映射为退出码。
- stdout 只放协议记录；诊断写 stderr。

非交互模式会接管普通 `process.stdout.write()`，把普通日志重定向到 stderr，协议通过 raw stdout 有序写出并处理背压。

### 10.3 JSON 事件是 delta-only

SDK 的 `message_update` 带累计消息快照，但 JSON/RPC wire 事件去掉累计快照和 `partial`，只保留：

- 当前 usage。
- `contentIndex`。
- delta。
- toolcall_start 的稳定 `id/toolName`。

客户端应：

1. 用 `message_start` 初始化。
2. 按 `contentIndex` 应用 delta。
3. 用 `text_end/thinking_end/toolcall_end` 校正块。
4. 最终以 `message_end.message` 替换整个消息。

这显著减少流式传输量，但把正确重组的责任交给客户端。

### 10.4 RPC 是“接受响应 + 事件完成”

RPC 使用严格 LF 分隔的 JSONL。命令可带 `id`，匹配 response 重复该 ID。

`prompt` 的 success response 只说明输入被接受、排队或由扩展 handled：

```json
{"id":"1","type":"response","command":"prompt","success":true,"data":{"disposition":"started"}}
```

它不代表模型工作完成。客户端必须继续读事件，并通常等待 `agent_settled`。

命令处理是异步的，没有全局串行队列：

- abort 可在长命令执行时进入。
- 响应顺序不保证等于输入顺序。
- 顺序敏感的状态修改需要客户端自行串行化。

### 10.5 SDK 的注入边界和生命周期

SDK 直接创建 `AgentSession`，适合 Node.js/Bun 宿主。关键规则：

- 先 `subscribe()`，再 `prompt()`，否则可能错过快速事件。
- `prompt()` 在活动 run 中必须明确 steering/follow-up。
- `dispose()` 会 abort、断开订阅、使扩展 context 失效。
- runtime 替换 session 后，旧订阅和旧 extension context 不能继续用，必须在新 session 上重新绑定。
- `sessionManager` 可换为 in-memory；`resourceLoader` 可完全自定义资源和发现。

SDK 默认不自动加载 CLI 内置的 codemode/tool_search/MCP 工厂；需要这些能力时，显式把对应 extension factories 加入 `DefaultResourceLoader`。

## 11. TUI：组件、布局与差量渲染

### 11.1 保留式组件树

`pi-tui` 的组件不是虚拟 DOM，而是保留实例：

```ts
render(width): string[]
handleInput?(data)
handleMouse?(event)
invalidate()
```

每次渲染返回带 ANSI/终端控制序列的行数组。组件实例和状态跨帧保留；`invalidate()` 清理缓存并触发后续重算。

内置组件包括 Text、Markdown、Image、Container、VStack/HStack、ScrollView、Input、Editor、SelectList、Loader、MouseRegion 等。

### 11.2 主屏与备用屏是两种布局模型

- **主屏 `TuiMainScreen`**：终端拥有 scrollback，TUI 渲染纵向文档并跟踪当前位置。
- **备用屏 `TuiAltScreen`**：应用拥有固定 viewport、区域布局、滚动、选择、Overlay 和鼠标命中。

coding-agent 的 regular 模式使用主屏；fullscreen 使用备用屏。fullscreen 布局大致是：

```text
VStack
├─ transcript ScrollView：basis=0, grow=1, follow=end
└─ dock
   ├─ pending queue
   ├─ status
   ├─ widgets
   ├─ editor
   └─ footer
```

VStack/HStack 使用 basis/grow/shrink/min/max 分配空间；ScrollView 维护 scrollTop、viewport、内容高度和 follow-end 状态；Overlay 独立栈合成在底层内容之上。

### 11.3 输入管线

完整链路是：

```text
process.stdin
  → StdinBuffer：重组 CSI/OSC/paste/鼠标/Kitty 序列
  → ProcessTerminal：协议协商和原始输入
  → TUI 输入监听器
  → viewport 行为：鼠标、滚动、搜索
  → debug handler
  → focusedComponent.handleInput()
  → 立即调度渲染
```

这解决了“字节流分片不等于按键事件”的问题。焦点组件通过零宽 `CURSOR_MARKER` 定位硬件光标，并可渲染假光标，保证 IME 候选窗口位置正确。

### 11.4 Differential rendering

渲染请求约按 16ms 合并；终端更新包在 synchronized output 中，减少撕裂。

两种渲染器的 diff 策略不同：

- **主屏**：生成完整文档，找出首末变化行，通常重画这段连续区间，并小心避免意外滚屏。
- **备用屏**：生成固定高度屏幕帧，逐行比较并跳过未变化行，用绝对行坐标更新变化行。

例如四行 `[A,B,C,D]` 变为 `[A,x,C,y]`：

- 主屏通常重画下标 1 到 3。
- 备用屏纯文本普通帧只更新下标 1 和 3。

组件缓存减少重复计算，行 diff 减少终端写入；它不是“只遍历脏组件”的虚拟 DOM。

### 11.5 终端能力、Unicode 与原生模块

TUI 处理：

- Kitty keyboard、bracketed paste、鼠标、OSC 8、OSC 52、进度和程序状态。
- Kitty/iTerm2 图像能力及降级。
- 终端前景/背景/16 色 palette 查询和主题变化。
- grapheme、宽字符、emoji、组合字符、ANSI 状态恢复和可见列计算。

基础文本 TUI 的 npm 运行时依赖很少；剪贴板、Windows 输入和平台修饰键使用随包发布的预编译 `.node` helper，加载失败会退化，而不是要求安装期编译。

## 12. 可靠性设计

### 12.1 恢复不是单一机制

Pi 的可靠性由多层组成：

1. Provider request 层：可配置的 HTTP/网络重试。
2. Assistant turn 层：AgentSession 对可重试错误做有限指数退避。
3. 上下文溢出层：省略失败尝试，compaction，再 retry。
4. 摘要层：summarization retry，拒绝保存不完整摘要。
5. 工具层：异常转为模型可见 ToolResult，而不是直接崩溃。
6. 取消层：AbortSignal 贯穿请求、hooks、工具和 backoff。
7. 输出层：stdout 写队列、背压和 JSONL framing。
8. Session 层：追加式 JSONL、坏行跳过、投影重建。

这些机制的错误类别和重试上限不同，不能统称为“Pi 自动重试”。

### 12.2 Overflow recovery 的顺序

一次典型 context overflow 恢复是：

```text
保存失败的 Assistant 和结束事件
→ 追加 context_edit，省略失败尝试对未来上下文的影响
→ 触发 compaction
→ 成功后将原请求作为新 run 重试
→ 失败或取消则不再自动继续
```

这不是一个磁盘事务：omission 已写入后，若 compaction 失败，omission 仍保留；原始历史仍可审计。

### 12.3 大输出控制

常见策略是“模型看到截断结果，完整结果落盘”：

- Bash/read/MCP/codemode 的大文本按限制中部截断。
- 完整输出写临时文件，并在结果中告诉模型路径。
- codemode 脚本可获得更完整结果，先过滤/聚合，再把摘要交给模型。
- 摘要对 tool result 再限制到约 2000 字符，避免压缩请求自身过大。

这把“持久取证”和“模型上下文预算”分开。

## 13. 安全模型与边界

### 13.1 Pi 没有内置完整权限系统

默认情况下，Pi 的工具、`!` 命令、扩展、MCP 子进程和 Provider extension 都拥有启动 Pi 的用户权限。项目 trust、工具 hooks、codemode QuickJS 都不是完整操作系统边界。

需要限制文件、进程、网络或凭据时，应使用：

- 整进程容器/VM/sandbox。
- 独立低权限用户。
- Docker Sandboxes/OpenShell 等外部边界。
- 仅工具委托方案，例如 Gondolin；注意其他扩展工具仍可能在宿主运行。

### 13.2 Project trust 控制资源加载，不控制工具权限

Project trust 主要防止项目目录在未确认时加载：

- `.pi/settings.json`
- `.pi/mcp.json`
- `.pi/extensions/skills/prompts/themes`
- `.pi/SYSTEM.md`、`APPEND_SYSTEM.md`
- 项目 packages

它不限制 bash、read/write/edit 或已加载扩展的权限，也不阻止 `AGENTS.md` 内容影响模型。

### 13.3 扩展是高权限代码

Extension 可以读取文件、环境变量、凭据、session 历史，也可发起网络请求和执行命令。Pi package 的 npm/git 安装还可能执行第三方生命周期脚本。只应从可信来源安装扩展和包。

### 13.4 Prompt injection 是内容风险

文件、issue、网页、命令输出和 MCP 返回内容都可能包含“执行某操作”的文本。模型看到的内容不是可信指令边界。降低风险依赖：

- 限制可用工具。
- 对危险工具调用加确认 hook。
- 外部 OS sandbox。
- 最小化凭据和网络暴露。
- 审查 diff 和导出内容。

### 13.5 Session 本身可能含敏感信息

Session、HTML export、debug log 和分享链接可能包含 prompt、文件内容、命令输出、工具参数、MCP 结果和意外暴露的凭据。分享前应审查。

## 14. 工程化与供应链

### 14.1 TypeScript/ESM 约束

共享 TypeScript 配置使用：

- ESM / Node 模块解析。
- strict。
- `erasableSyntaxOnly`。
- 源码相对 `.ts` 导入，构建时改写为 `.js`。
- declaration、declarationMap、sourceMap。

这同时支持 Node 类型擦除运行、`tsc` npm 包构建和测试工具解析。因此不允许需要 JS emit 的 TS 语法，也不允许随意混用相对 `.js` 导入。

### 14.2 检查与测试分层

根 `npm run check` 包括：

- Biome，带 `--write`，因此不是只读检查。
- 精确依赖版本检查。
- runtime dependency/import 图检查。
- TS 相对导入检查。
- public entry import graph 检查。
- coding-agent install lock 检查。
- `tsc --noEmit`。
- browser smoke。

测试按层次分布：

- 纯函数和协议测试。
- TUI xterm headless 测试。
- 包级离线 Vitest、faux provider。
- 平台/原生能力测试。
- 凭据和网络 E2E，需显式环境启用。
- MCP conformance 和安装后 consumer smoke。

`./test.sh` 会隔离 HOME、XDG、npm cache、git 配置和环境变量，避免意外触发真实 Provider e2e。

### 14.3 模型数据可复现

模型生成器把上游目录和人工修正固化为源码/JSON，并有：

- schema version。
- provider/type/api 分组校验。
- 文件 SHA-256。
- 结构 hash。
- staging 验证和失败恢复。
- Nix 不可变 catalog pin。

`models.generated.ts` 是生成文件，不应手改；应修改生成脚本并重新生成。

### 14.4 安装锁与发布

供应链设计区分：

1. 根 `package-lock.json`：开发依赖ground truth。
2. `packages/coding-agent/install-lock/`：安装器使用的生产依赖闭包，固定内部包 tarball 和传递依赖。
3. 发布 tarball：SHA-512 命名、artifact manifest、integrity 校验、npm provenance。

直接外部依赖要求精确版本；安装和 CI 使用 `--ignore-scripts`。install-lock 对生命周期脚本有显式 allowlist，新增带脚本的依赖会失败，必须人工审查。

## 15. Durable、Chord 与其他支撑包

### 15.1 `pi-durable`：提交优先的可恢复运行时

Durable 解决的是另一个更强的问题：进程在模型请求、工具执行或 compaction 中途退出后，如何从持久 checkpoint 恢复。

核心机制：

- `Session.commit()` 是原子持久化边界；Storage 成功后才 adopt 内存文档并发布 commit。
- `Entry` 保存模型贡献、head 和 edits。
- `Task` 是持久状态机，有 pending/running/waiting/completing/terminal。
- `Document` 是 session/conversation/task 作用域的 typed JSON 状态，支持 base + delta、rewindable 和 fork。
- Generation、Tool、Compaction 都是内置任务。
- 支持 SQLite、JSONL、Memory Storage。

它通过 `pi-ai` 发模型请求，通过 Chord delta/state 管理文档和观察。当前 CLI 不使用它；它是独立的实验性 Harness。

### 15.2 Chord：完整插件宿主与基础设施分离

Chord 有两层：

- 完整 Host/Facet/contract/RPC/热重载/复制对象系统。
- 可独立使用的 Context、immutable delta、replicatedState、OutputBuffer。

当前 coding-agent 主链不使用完整 Chord Host；Chord 的 delta/state/context/output 被 durable 作为主实现基础设施。不要把它与 coding-agent extensions 或 CLI RPC 混为一谈。

### 15.3 `pi-env`

`pi-env` 是 durable `ExecutionEnv` 的远程/SSH/daemon 实现，提供文件、命令、watch、PTY 和 `listFiles`。它支持分块大文件、远程 daemon、SSH host key 管理和跨平台语义。当前 CLI、agent、ai 不依赖它；只有 durable 宿主显式配置时进入链路。

### 15.4 `pi-telemetry`

Telemetry 提供后端中立的 `startSpan()` 合约、noop/内存实现和 adapter conformance。当前 `pi-ai` 请求选项可以透传 `telemetryContext`，但内置 Provider、AgentSession 和 coding-agent 主链没有默认创建 span 或 exporter。它也不等于 coding-agent 的 install/update 上报。

### 15.5 `pi-evals`

Evals 是私有开发工作区，使用真实 `ModelRuntime`、session services 和 `AgentSession` 评测行为。文档评测会构建 with/without docs 的 Docker 双变体，比较通过率、token、工具调用和成本。它不是生产运行时模块。

## 16. 关键设计取舍

| 设计 | 解决的问题 | 代价/限制 |
|---|---|---|
| 多接口共用 AgentSession | 避免 TUI、JSON、RPC、SDK 各自实现一套 Agent | 接口语义复杂；必须理解事件和 settle 边界 |
| Provider/API/Model 分离 | 支持兼容端点、动态目录和复用协议适配 | 概念更多；自定义 Provider 需要理解契约 |
| Transcript system messages | 可重放动态 prompt/tool 变化 | 不支持中途 system 的 Provider 需折叠，可能失缓存 |
| 统一流式事件 | 一套 Agent/UI/JSON 逻辑适配多模型 | `partial` 可变、事件交错，客户端必须正确使用 `contentIndex` |
| 并行工具 + 源顺序结果 | 执行快且 transcript 可重放 | 共享状态工具必须声明 sequential 或自行加锁 |
| 追加式 JSONL 树 | 完整历史、分支、审计、简单导出 | 文件不随 compaction 缩小；无多写者锁/事务 |
| context_edit/compaction | 修复未来上下文而不篡改历史 | 模型看到的内容与原始 UI/导出不同，需理解投影 |
| 同进程扩展 | API 强、状态共享、类型一致、组合容易 | 无隔离；恶意扩展拥有宿主权限 |
| Skill 按需加载 | 减少常驻上下文 | 模型可能漏加载；关键流程需显式 `/skill:name` |
| Codemode | 减少工具声明和中间结果占用 | JS 沙箱、worker、输出格式和调试更复杂 |
| JSON delta-only | 流式协议线性大小 | 客户端重组成本高 |
| TUI 行差量渲染 | 减少闪烁和终端写入 | Unicode、图像、滚动和 viewport 边界复杂 |
| 静态目录 + 动态刷新 | 发布可复现且支持实时模型 | 需要维护生成、hash、离线数据两套机制 |

## 17. 端到端 Trace：一次“读文件并总结”

下面把一个具体任务串起来：

```text
用户：读取 README，给出三点总结
```

1. **接口层**  
   TUI/RPC/SDK 将输入交给 `AgentSession.prompt()`。

2. **输入处理**  
   扩展命令未命中；`input` handlers 可转换；skill/prompt template 展开。

3. **会话准备**  
   SessionManager 重建当前投影；AgentSession 检查模型、auth 和 compaction；`before_agent_start` 可改 prompt/tools。

4. **写入用户消息**  
   低层 Agent 发出 `agent_start/turn_start/message_start/end(user)`；AgentSession 持久化该消息。

5. **请求模型**  
   Agent 执行 `transformContext → convertToLlm → normalizeContext`；ModelRuntime/Models 解析认证，选择 Provider/API adapter；Provider 构造请求并开始流式响应。

6. **归一化流**  
   Provider 事件变成 `text_delta` 和 `toolcall_delta`；Agent 转发为 `message_update`；TUI 增量渲染，JSON 客户端按 `contentIndex` 重组。

7. **工具调用**  
   `toolcall_end` 得到 read 参数；最终 JSON 和 TypeBox 校验；扩展 `tool_call` 可拦截；工具执行读取文件；长输出截断并给完整路径。

8. **工具结果**  
   `tool_execution_end` 发出；ToolResult 按源顺序写入 Session；`turn_end` 包含 Assistant 和 toolResults。

9. **自动续问**  
   工具结果触发下一 turn；模型拿到文件内容，生成三点总结。

10. **结束**  
    无更多工具或队列消息，低层 `agent_end`；AgentSession 检查 retry/compaction/queue，执行 `agent_before_settle`，最后发 `agent_settled`。

11. **后续恢复**  
    Session 文件保留 user、Assistant、tool call、tool result、usage 和 model metadata。重新打开时从当前 leaf 建投影，而不是简单读取最后一行文本。

## 18. 源码阅读地图

| 主题 | 首要文件 | 重点符号/问题 |
|---|---|---|
| 统一消息和事件 | `packages/ai/src/types.ts` | `Message`、`AssistantMessageEvent`、`Model`、`Tool` |
| Transcript 动态 prompt/tools | `packages/ai/src/utils/transcript.ts` | `normalizeContext()`、system replay/collapse |
| Provider 集合与请求门面 | `packages/ai/src/models.ts` | `ModelsImpl.streamSimple()`、`createProvider()` |
| 工具参数 | `packages/ai/src/utils/validation.ts` | `validateToolArguments()` |
| Agent 状态和公共 API | `packages/agent/src/agent.ts` | `Agent.prompt()`、`steer()`、`followUp()`、`abort()` |
| 低层循环 | `packages/agent/src/agent-loop.ts` | `runAgentLoop()`、`executeToolCalls()`、`declareToolChanges()` |
| Session 组合入口 | `packages/coding-agent/src/core/sdk.ts` | `createAgentSession()` |
| 会话级生命周期 | `packages/coding-agent/src/core/agent-session.ts` | `prompt()`、`_startRun()`、`_handlePostAgentRun()`、`bindExtensions()` |
| JSONL 树和投影 | `packages/coding-agent/src/core/session-manager.ts` | `buildSessionProjection()`、`appendContextEdit()`、`branchWithSummary()` |
| Compaction | `packages/coding-agent/src/core/compaction/compaction.ts` | `prepareCompaction()`、`compact()`、cut point |
| 扩展类型 | `packages/coding-agent/src/core/extensions/types.ts` | `ExtensionAPI`、事件联合、tool exposure |
| 扩展加载/运行 | `packages/coding-agent/src/core/extensions/loader.ts`、`runner.ts` | factory、事件合并、错误边界 |
| 资源发现 | `packages/coding-agent/src/core/resource-loader.ts`、`package-manager.ts` | trust、优先级、包过滤 |
| JSON wire | `packages/coding-agent/src/modes/json-event.ts` | `toJsonEvent()` |
| RPC | `packages/coding-agent/src/modes/rpc/rpc-mode.ts`、`rpc-types.ts` | command/response/event |
| TUI 核心 | `packages/tui/src/tui.ts`、`tui-main-screen.ts`、`tui-alt-screen.ts` | `Component`、focus、diff render |
| Codemode | `packages/coding-agent/src/extensions/codemode/`、`packages/codemode/src/runtime/` | sandbox、nested tools、store |
| MCP | `packages/mcp/src/client.ts`、`packages/coding-agent/src/extensions/mcp/` | transport、tool adapter、exposure |
| Durable | `packages/durable/src/harness/`、`session/`、`storage/` | commit、Task、Document、recovery |
| Chord | `packages/chord/src/host.ts`、`delta/`、`state.ts` | Host/facet、delta、replicated state |

## 19. 最适合继续深入的 10 个问题

1. Provider adapter 如何处理各家 tool call ID、orphan tool call 和跨模型 handoff？
2. `before_agent_start`、`context`、`context_with_system` 的推荐边界是什么？
3. 如何让一个扩展的状态严格跟随 Session branch？
4. 为什么普通公共 `message_end` 与 SessionManager append 的时点对宿主重要？
5. 如何设计一个安全的 `tool_call` 权限门，同时不制造不可用的非交互模式？
6. Codemode 与 tool_search 分别如何影响 token、缓存和模型行为？
7. Compaction 的 token 估算在哪些输入上最容易失真？
8. JSON/RPC 客户端如何正确实现取消、超时和背压？
9. TUI 的宽度、Overlay 和图像 diff 在哪些终端上会退化？
10. 如果未来把 coding-agent 迁移到 durable，JSONL Session 的哪些语义最难保留？

## 20. 最终心智模型

可以用一句话概括 Pi：

> Pi 把“一次模型调用”扩展成一个可观察、可恢复、可分支、可扩展的任务运行时；它通过统一 Provider 事件驱动 Agent 循环，通过追加式 Session 树保存原始事实，通过投影决定模型当前看到什么，通过同进程扩展把产品行为开放给工作流。

学习时不要试图一次掌握所有包。先掌握主链：

```text
pi-ai → pi-agent-core → coding-agent AgentSession → SessionManager
```

再向外扩展到：

```text
Extensions → MCP/codemode → Interfaces → TUI → durable/chord
```

这条顺序与系统实际依赖方向一致，也能最快解释大多数源码行为。
