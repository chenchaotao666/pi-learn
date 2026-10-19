# Pi 核心设计与源码学习指南

> 基于提交 `90d863f7c`。源码行号会随后续提交变化，应以符号名和文件路径为主。
>
> 目标是理解 Pi 的设计，并能够独立追踪、修改一次完整的 Agent 请求，而不是按字母顺序读完所有源码。

## 1. 先确定学习主线

Pi 是 monorepo，但当前编码 Agent 的主链路可以浓缩为四层：

```text
pi-coding-agent   产品编排：CLI 模式、会话、工具、压缩、扩展
        |
pi-agent-core     Agent 状态机：消息、事件、模型轮次、工具调用
        |
pi-ai             模型适配：统一消息/流式事件，转换到各供应商 API
        |
OpenAI / Anthropic / Google / ...
```

`pi-tui` 位于 `pi-coding-agent` 的交互表现层，不在 Agent 决策循环中。`durable`、`chord`、`env` 构成另一条进阶运行时路线，不是当前 CLI 请求的必经层。

推荐顺序：

1. `pi-ai`：先理解模型输入、输出和流式事件的统一契约。
2. `pi-agent-core`：理解 Agent 如何围绕契约实现循环和状态管理。
3. `pi-coding-agent`：理解产品层如何提供工具、会话、压缩和多种前端。
4. `pi-tui`：最后理解终端 UI，避免把渲染逻辑误当成 Agent 逻辑。
5. 按需学习扩展、RPC、MCP、codemode、durable 和 chord。

学习时始终追踪四类数据，而不是只追函数：

- **模型上下文**：发送给 LLM 的消息和工具定义。
- **Agent 运行时状态**：当前消息、是否正在流式输出、待处理输入等。
- **持久化会话**：用于恢复、查看和分支的磁盘记录。
- **UI 展示状态**：终端当前显示的组件和文本。

它们有关联，但不是同一份数据。尤其不要预设“压缩上下文等于删除会话历史”。

## 2. 包边界：先判断“属于哪一层”

| 包 | 核心职责 | 初学优先级 |
|---|---|---|
| `packages/ai` | 统一模型、消息、工具调用和流式响应；适配不同 LLM API | 第一主线 |
| `packages/agent` | 通用 Agent 状态、事件和模型—工具循环 | 第一主线 |
| `packages/coding-agent` | 把通用 Agent 组装成编码产品：内置工具、会话、压缩、CLI、SDK、RPC、扩展 | 第一主线 |
| `packages/tui` | 终端组件、输入和差分渲染 | 第二阶段 |
| `packages/mcp` | 独立的 Model Context Protocol 客户端，为应用接入外部 MCP 工具 | 按需 |
| `packages/codemode` | 在受限 JavaScript 运行时中调用显式注入的工具 | 按需 |
| `packages/durable` | 持久会话、任务和文档运行时 | 进阶，非当前 CLI 必经层 |
| `packages/chord` | 类型化插件宿主、契约、复制状态和 JSON 链路 RPC | 进阶，主要服务 durable 等运行时 |
| `packages/env` | 为 durable 提供本地或 SSH 远程执行环境 | 进阶 |
| `packages/telemetry` | 显式传递的、后端无关的遥测契约；不改变业务状态 | 横切能力，后读 |
| `packages/evals` | 行为评测与文档对照实验，不是运行时主链路 | 用于验证行为，后读 |

这里有两个容易混淆的边界：

- **coding-agent 扩展与 chord 插件不是同一个概念**。前者服务于 Pi 编码产品，后者是通用类型化宿主运行时。
- **会话持久化与 durable 不是同一个概念**。当前 CLI 自己可以保存和恢复会话；使用 durable 是另一条更通用的运行时路线。

从 `packages/coding-agent/package.json` 的直接依赖也能验证主线：它依赖 `agent`、`ai`、`tui`、`mcp` 和 `codemode`，并不依赖 `durable`。

<!-- MAIN_TRACE -->

## 4. `pi-ai`：统一的不是供应商，而是对话与事件契约

### 4.1 两阶段路由：先选 provider，再选 wire API

`Models` 是运行时集合，负责保存 provider、解析鉴权并发起请求。路由分两步：

1. `model.provider` 找到拥有该模型的 provider。
2. `model.api` 在该 provider 内选择具体协议实现，例如 OpenAI Responses 或 Anthropic Messages。

因此，同一个 provider 可以容纳多种 API 协议；`Model` 本身是可序列化的元数据，不携带实现函数。核心入口见 [models.ts](../packages/ai/src/models.ts#L246-L364)、[models.ts](../packages/ai/src/models.ts#L1002-L1089) 和 [types.ts](../packages/ai/src/types.ts#L824-L872)。

这比“每种模型一个类”更容易扩展，也把**模型身份**与**协议实现**解耦了。

### 4.2 `Context` 是 provider 的边界

公共 `Context` 由 system prompt、messages 和 tools 组成。`normalizeContext()` 将其规范化成 transcript，provider 不直接接收任意调用方对象。消息角色为 `system | user | assistant | toolResult`：

- user 和 toolResult 可携带文本、图片；
- assistant 可携带 text、thinking、toolCall，并记录 usage、stopReason、模型身份和供应商回放签名；
- ToolCall 的 `id` 与 ToolResult 的 `toolCallId` 配对。

见 [types.ts](../packages/ai/src/types.ts#L408-L477)、[types.ts](../packages/ai/src/types.ts#L765-L788) 和 [transcript.ts](../packages/ai/src/utils/transcript.ts#L25-L34)。

关键边界：**adapter 只生成和解析工具调用，不执行工具，也不运行 Agent 循环**。工具 schema 校验和执行属于调用方；`pi-ai` 提供 [validation.ts](../packages/ai/src/utils/validation.ts#L295-L353) 作为公共校验工具。

跨模型回放也不是简单透传消息。[transform-messages.ts](../packages/ai/src/api/transform-messages.ts#L86-L177) 会按 provider/API/model 决定 thinking 和签名是否保留，并处理工具 ID、孤立 tool call、错误或中止的 assistant turn。这是统一多供应商对话时很容易被忽略的一层。

### 4.3 `complete` 也是流式实现的最终等待

| API | 实际含义 |
|---|---|
| `stream()` | 使用当前 `model.api` 的完整供应商选项，返回统一事件流 |
| `complete()` | 等待 `stream().result()`，不是另一套非流式 HTTP 路径 |
| `streamSimple()` | 把统一的 reasoning、toolChoice 等选项映射为供应商选项 |
| `completeSimple()` | 等待 `streamSimple().result()` |

统一事件流大体为：

```text
start
  → text / thinking / toolcall 的 start、delta、end（可以交错）
  → done
```

失败以 `error` 结束。`EventStream` 用 FIFO 缓冲事件，`result()` 与迭代器共享同一条流；终止后，迭代器仍会先排空已经缓冲的事件。见 [event-stream.ts](../packages/ai/src/utils/event-stream.ts#L25-L89)。

一个非常重要的细节：`partial` 指向**持续被修改的同一个 AssistantMessage**，不是每个事件附带的历史快照。如果消费者需要保存中间状态，不能不加处理地保留这个引用。工具参数的 delta 只是 best-effort partial JSON，结束时才做完整解析； malformed JSON 不会被当成合法参数执行。

### 4.4 adapter 负责双向转换

以 OpenAI Responses 为例，主链是：

```text
Models.stream()
  → normalizeContext / lazy 初始化 / applyAuth
  → provider.stream
  → 按 model.api 选择 adapter
  → buildParams + 消息和工具转换
  → 供应商流式请求
  → 供应商事件归一化为 AssistantMessageEvent
  → done/error，并得到最终 AssistantMessage
```

请求入口在 [openai-responses.ts](../packages/ai/src/api/openai-responses.ts#L147-L265)，消息和事件转换集中在 [openai-responses-shared.ts](../packages/ai/src/api/openai-responses-shared.ts#L156-L422)。Anthropic 复用同样的公共路由，最终进入 [anthropic-messages.ts](../packages/ai/src/api/anthropic-messages.ts#L575-L937)。

学习时应先掌握“统一事件如何形成”，再研究 `thinkingSignature`、SSE、encrypted reasoning 等供应商细节。后者很重要，但不是理解 Agent 循环的前置条件。

### 4.5 usage、重试和模型目录的三个反直觉点

- `Usage` 统一 token 与 cost 字段，cost 由本地 catalog 单价计算，不等于供应商返回的账单；reasoning token 若存在，是 output 的子集，不应重复相加。见 [types.ts](../packages/ai/src/types.ts#L451-L472) 和 [models.ts](../packages/ai/src/models.ts#L1206-L1225)。
- 包内同时有“请求建立重试”和“完整 assistant turn 重试”两类 helper，但默认请求重试次数是 0，完整 turn 重试也不会自动接入 `Models.stream()`。不要假设调用 SDK 就天然获得所有重试策略。见 [provider-retry.ts](../packages/ai/src/utils/provider-retry.ts#L24-L69) 和 [retry.ts](../packages/ai/src/utils/retry.ts#L181-L265)。
- `models.generated.ts` 和 JSON catalog 是构建期模型元数据，用于路由、能力、限制和价格，不是运行时 provider 实现。初学时应最后读生成器，不要从生成文件倒推架构。

## 5. `pi-agent-core`：事件驱动的有状态循环

### 5.1 三层职责

| 层 | 职责 |
|---|---|
| `Agent` | 持有 transcript、模型、工具、输入队列和运行状态；把事件还原进 `state`，并通知监听器 |
| `agentLoop` / `runAgentLoop` | 无 UI 的编排核心：模型响应、工具执行、结果回填、下一轮 |
| `StreamFn` | 模型传输边界；循环只依赖统一的 `AssistantMessageEventStream` |

公共契约集中在 [types.ts](../packages/agent/src/types.ts#L19-L37)，状态定义见 [types.ts](../packages/agent/src/types.ts#L376-L421)，循环入口见 [agent-loop.ts](../packages/agent/src/agent-loop.ts#L102-L150)。

关键设计：**循环内部使用 `AgentMessage[]`，只在请求模型前转换为 provider 可理解的 `Message[]`**。因此产品层可以在 transcript 中携带附件或自定义消息，而不强迫所有 provider 理解它们。

### 5.2 run、turn、message 是三个不同粒度

- 一个 **run** 通常由一次 `prompt()` 触发，可包含多个模型轮次。
- 一个 **turn** 是一次 assistant 响应及其工具结果。
- message 事件同时覆盖用户输入、assistant 和 toolResult。

简化后的主循环是：

```text
把新输入加入 transcript
for each turn:
    转换本次模型上下文
    消费 StreamFn 的统一事件，得到完整 assistant message
    若 assistant error/aborted：结束 run
    若 content 含 toolCall：执行并追加 toolResult
    发出 turn_end
    有工具后续、steering、follow-up 或显式 continue：继续
    否则结束 run
```

实际实现见 [agent-loop.ts](../packages/agent/src/agent-loop.ts#L163-L321)。是否执行工具取决于 content 中是否真的有 `toolCall`，不是只看 `stopReason === "toolUse"`。

还要区分两个入口的返回方式：

- `Agent.prompt()` 返回 `Promise<void>`；答案从事件或 `agent.state.messages` 获取。
- 低级 `agentLoop()` 返回事件流，`runAgentLoop()` 返回本次新增的 `AgentMessage[]`，不是最终字符串。

### 5.3 事件不仅用于显示，也负责驱动 `Agent.state`

事件包括：

```text
agent_start / agent_end
turn_start / turn_end
message_start / message_update / message_end
tool_execution_start / update / end
```

`Agent.processEvents` 先更新状态，再按订阅顺序逐个 `await` 监听器。`message_end` 才把完整消息追加到 `state.messages`，tool start/end 更新 `pendingToolCalls`。实现见 [agent.ts](../packages/agent/src/agent.ts#L558-L612)。

这里存在一个很有意识的背压差异：

- 使用 `Agent` 时，异步监听器会形成事件屏障；例如 assistant `message_end` 的监听器完成后才进入工具预检。
- 直接使用低级 `agentLoop()` 事件流时，慢消费者不会阻塞生产循环。

所以“需要持久化完成后才执行下一步”与“只观察进度”不是同一种订阅语义。

### 5.4 流式桥接同时暴露累计值和增量值

[streamAssistantResponse](../packages/agent/src/agent-loop.ts#L381-L469) 在请求前依次处理：

```text
transformContext → convertToLlm → normalizeContext → getApiKey → streamFn
```

消费事件时：

- `start` 建立 partial assistant；
- delta 事件更新 partial，并发出 `message_update`；
- `done/error` 通过 `response.result()` 取得最终消息，发出 `message_end`。

`message_update.message` 是累计 partial，`assistantMessageEvent.delta` 是新增片段。UI 若要追加文本应使用 delta，若要重绘当前响应则使用 partial，不能混用。

`transformContext` 默认只是**本次请求的投影**。它可以裁剪或注入上下文，但不会自动把裁剪结果写回 `Agent.state.messages`。这正是“运行时 transcript”和“模型所见上下文”不能混为一谈的原因。

### 5.5 工具管线：失败通常交还给模型，而不是终止 run

正常工具管线为：

```text
tool_execution_start
  → 查找工具
  → 参数兼容转换与 schema 验证
  → beforeToolCall
  → execute
  → afterToolCall
  → tool_execution_end
  → 发出 toolResult message
```

完整实现见 [agent-loop.ts](../packages/agent/src/agent-loop.ts#L694-L777) 和 [agent-loop.ts](../packages/agent/src/agent-loop.ts#L821-L949)。

几个必须掌握的行为：

- 未知工具、参数验证失败、hook 阻断或异常会形成 `isError: true` 的 toolResult；不会执行 `execute`。
- `execute` 抛错也会变成错误 toolResult，并继续经过 `afterToolCall`。
- 工具失败通常进入下一轮模型输入，让模型修复或解释；它不等同于 `state.errorMessage` 所表示的 assistant/provider 失败。
- 若 assistant 因 `length` 被截断，其中的 toolCall 不执行，而是返回错误结果让模型重新生成，避免执行不完整的参数。

工具结果还区分 `content`（面向模型）、`details`（主要面向 UI/日志）和 `structuredContent`（面向程序）。不要把所有结果都当成同一段文本。

### 5.6 并行执行有明确的顺序保证

默认并行模式并不是整条管线都并行：

- 预检和 `beforeToolCall` 按 assistant 中的调用顺序串行执行；
- 通过预检的 `execute()` 才并发运行；
- `tool_execution_end` 按实际完成顺序发出；
- toolResult message 和 `turn_end.toolResults` 按原始 toolCall 顺序发出；
- 批次中只要有工具声明 `executionMode: "sequential"`，整批退化为顺序执行。

见 [agent-loop.ts](../packages/agent/src/agent-loop.ts#L508-L660)。这个设计同时保留了执行效率和可重建的 transcript 顺序。

### 5.7 取消与结束不是同一件事

- `agent.abort()` 触发协作式 `AbortSignal`；StreamFn、工具和 hooks 都要尊重它。
- error/aborted assistant 会硬性结束 run，未处理的 steering/follow-up 仍保留。
- 工具结果的 `terminate: true` 只影响工具后的自动下一轮，不等于立即中止整个 run。
- `finishTurn({ action: "end" })` 是正常结束决策，和 abort 不同。

不要只用 `try/catch` 判断一次 prompt 是否成功。StreamFn 契约倾向于把模型和运行错误编码为流中的 `error` 及最终 assistant 的 `stopReason`，调用方还要检查最终消息的 `stopReason/errorMessage`。

### 5.8 两个值得后读的扩展点

- `streamProxy` 也是一种 `StreamFn`：通过 HTTP 传递统一事件，客户端重建 partial assistant；网络错误和异常 EOF 也归一化为 error result。见 [proxy.ts](../packages/agent/src/proxy.ts#L91-L251)。
- `beforeToolCall`、`afterToolCall`、`prepareNextTurn`、`prepareRequest`、`finishTurn` 分别提供审批、结果处理、轮次编排和请求前修改能力。先理解默认路径，再读这些 hooks，否则容易看不清核心循环。

<!-- CODING_AGENT_CORE -->

<!-- EXTENSIONS_AND_TUI -->

<!-- ADVANCED_PACKAGES -->

## 9. 值得学习的设计取舍

### 9.1 小核心与可替换边界

Pi 把供应商差异放在 `pi-ai`，把通用循环放在 `pi-agent-core`，把编码产品策略放在 `pi-coding-agent`。学习或修改功能时，先判断它属于哪一层：

- 新供应商的消息格式：通常属于 `pi-ai`。
- 工具调用后的通用循环行为：通常属于 `pi-agent-core`。
- 文件工具、会话恢复、上下文压缩：通常属于 `pi-coding-agent`。
- 按键和组件绘制：通常属于 `pi-tui` 或 coding-agent 的交互模式。

这个判断比记住目录结构更重要，它能防止把产品策略下沉到通用 Agent 核心。

### 9.2 事件流同时服务内部状态和外部消费者

流式事件不仅让文字逐步显示，也让 TUI、RPC 等消费者观察同一个执行过程。阅读时要区分：

- 中间事件：可能频繁变化，用于进度和局部展示。
- 最终消息：一轮模型输出或工具执行完成后的稳定结果。
- 持久化记录：是否以及何时写入，由会话层决定，不应从 UI 刷新反推。

### 9.3 默认能力刻意保持精简

根 README 明确说明 Pi 不内置某些工作流功能，而是依靠扩展和包让用户组合。这里的设计重点不是“功能越少越好”，而是核心只稳定承载通用机制，把有争议的工作流留给上层。

### 9.4 扩展能力和安全边界是两件事

Pi 默认继承启动进程的操作系统权限，不把扩展机制误认为安全沙箱。需要隔离文件、进程、网络或凭据时，应使用容器或专门的沙箱方案。codemode 的能力限制也应与整个 CLI 的权限模型分开理解。

## 10. 分阶段学习与练习

每个阶段都应产出一张调用链或状态图。只读代码但没有产出，很容易变成“看过但不理解”。

读每个关键符号时，用同一张表做笔记：

| 记录项 | 要回答的问题 |
|---|---|
| 输入 | 接收的是统一消息、供应商消息、事件，还是 UI 状态？ |
| 输出 | 返回最终值、异步迭代事件，还是只修改内部状态？ |
| 状态所有权 | 谁可以修改这份数据，其他位置拿到的是引用还是快照？ |
| 副作用 | 是否发事件、写会话、执行工具或刷新 UI？ |
| 结束条件 | 正常完成、无工具调用、取消和异常分别在哪里结束？ |

<!-- LEARNING_STAGES -->

## 11. 自检问题

能够回答以下问题，说明已经掌握主线：

1. 一次用户输入在哪里变成统一的 LLM `Context`？
2. 供应商的流式响应在哪里被归一化，最终消息在哪里形成？
3. 没有工具调用时，Agent 循环为什么结束；有工具调用时，下一轮输入是什么？
4. 工具开始、进度、成功、失败和取消分别怎样传播？
5. TUI 或 RPC 如何观察执行过程而不拥有自己的另一套 Agent 循环？
6. 内存消息、磁盘会话和 UI 文本有什么区别？
7. 会话恢复和分支为什么不等同于简单读取一个消息数组？
8. 压缩改变了什么，又保留了什么？
9. 扩展、MCP 工具和内置工具分别在哪一层接入？
10. 如果要新增一种能力，应该修改核心循环、产品会话层，还是只写扩展？

## 12. 阅读原则与常见误区

- 不要先读 `models.generated.ts` 或逐个供应商实现；先读公共契约和主流程。
- 不要从 CLI 参数解析一直顺序读到所有模块；沿一次请求的数据流跳转。
- 不要把 UI 中的“消息”直接等同于发给模型的消息。
- 不要只看正常路径；取消、工具错误和中断最能暴露状态所有权。
- 不要一开始同时学习 durable、chord 和当前 CLI；先确认它们是否在同一条依赖链上。
- 测试是行为规格。优先读 agent-loop、会话、扩展和渲染相关测试中的具体场景，而不是只看类型定义。

## 13. 官方文档入口

- [Pi 工作原理](../packages/coding-agent/docs/how-pi-works.md)
- [交互与使用](../packages/coding-agent/docs/usage.md)
- [会话](../packages/coding-agent/docs/sessions.md) 与 [会话格式](../packages/coding-agent/docs/session-format.md)
- [上下文压缩](../packages/coding-agent/docs/compaction.md)
- [扩展](../packages/coding-agent/docs/extensions.md)、[SDK](../packages/coding-agent/docs/sdk.md) 与 [RPC](../packages/coding-agent/docs/rpc.md)
- [安全](../packages/coding-agent/docs/security.md) 与 [容器化](../packages/coding-agent/docs/containerization.md)
