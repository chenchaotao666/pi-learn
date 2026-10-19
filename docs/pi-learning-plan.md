# Pi 项目详细学习计划

> 适用版本：当前 `main` 工作区，分析基线 `90d863f7c`。
>
> 配套文档：[Pi 技术特点与原理分析](./pi-technical-analysis.md)。
>
> 推荐节奏：6 周主线，每周 5 天，每天 1.5～2 小时，总计约 45～60 小时；另有 2 周进阶内容。时间不足时，优先完成每周的“验收标准”，不要平均用力阅读所有文件。

## 1. 学习目标

完成主线后，应能够：

1. 用一张图说明 CLI、SDK、`pi-coding-agent`、`pi-agent-core`、`pi-ai` 和模型服务之间的关系。
2. 跟踪一次请求从用户输入、模型流式响应、工具调用到下一次模型请求的完整过程。
3. 区分一次 assistant turn、一次低层 agent run，以及直到 `agent_settled` 的整个会话级任务。
4. 解释 Session 为什么是树而不是线性聊天记录，并手动还原当前分支的模型上下文。
5. 解释 compaction、`context_edit`、system message checkpoint 为什么不修改原始历史。
6. 编写一个带参数校验、事件处理、状态持久化和非交互模式行为的扩展。
7. 在 prompt template、skill、extension、MCP、codemode 和 Pi package 之间选择合适机制。
8. 使用 SDK 或 RPC 嵌入 Pi，并正确重组流式消息、等待任务真正结束、处理取消。
9. 说明 project trust、工具权限、QuickJS sandbox 和操作系统 sandbox 各自不能与不能提供什么保证。
10. 按仓库约束完成检查、定向测试和改动说明。

本计划不覆盖模型训练、推理引擎内部实现，也不要求逐个精读所有 Provider。Provider 协议适配应在掌握统一抽象后按需深入。

## 2. 前置知识与准备

### 2.1 必要基础

| 领域 | 最低要求 | 不足时先补什么 |
|---|---|---|
| TypeScript | 类型别名、联合类型、泛型、模块导入导出 | TypeScript 的 `type`、接口和 ESM 基础 |
| Node.js | `async/await`、Promise、事件、流、子进程 | Promise 并发、`AbortSignal`、AsyncIterable |
| 终端 | 环境变量、管道、标准输入输出、退出码 | stdout/stderr、TTY、JSONL |
| LLM 应用 | system/user/assistant、token、上下文窗口、tool call | 工具调用循环和流式响应的基本概念 |
| 数据格式 | JSON、JSON Schema、JSONL | 每行一个完整 JSON 对象的 framing |
| 工程协作 | Git 工作区、diff、定向测试 | 只暂存自己修改的文件 |

TypeBox 用于声明工具参数。看到 `Type.Object(...)` 时，将其理解为“同时供程序校验和模型工具声明使用的 schema”即可，不必先学完整库。

### 2.2 环境

- Node.js `>=22.19.0`。
- 一个终端和 Git。
- Windows 下可用原生 Node.js；涉及 `.sh` 脚本时使用 Git Bash 或 WSL，并阅读[原生 Windows 说明](../packages/coding-agent/docs/windows.md)。
- 静态阅读和不依赖模型的实验不需要 API key。需要真实 CLI 请求时，在 Pi 中运行 `/login`，或配置本地/兼容端点。

源码安装：

```bash
npm install --ignore-scripts
```

从源码启动：

```bash
./pi-test.sh
```

改动后的常用检查：

```bash
npm run check
./test.sh
```

不要为了学习而直接运行完整 vitest 命令；仓库的 `test.sh` 会避开需要真实端点的 e2e 测试。修改某个测试后，优先运行该测试文件。不要为了“验证学习成果”消耗真实模型额度，优先使用已有单元测试、内存 Session 和 faux provider。

### 2.3 固定学习方法

每个主题按以下顺序学习：

1. **先写问题**：这个机制解决什么具体问题？
2. **读官方文档**：先获得正确使用方式，不急着看实现。
3. **读核心源码**：找入口、状态、事件和返回位置。
4. **画一次 trace**：用一个具体输入写出调用顺序和关键数据。
5. **做最小实验**：一次只改变一个变量。
6. **写反例**：说明它不是什么，以及在哪一边界会失效。

建议每天保留一份简短记录：

```text
主题：
我原来认为：
源码实际行为：
关键文件和符号：
一次具体调用链：
仍未理解的问题：
```

## 3. 总体路线图

| 阶段 | 时间 | 核心问题 | 主要产出 |
|---|---:|---|---|
| 第 1 周 | 7～9 小时 | Pi 是什么，如何观察一次任务 | 架构图、四种接口对比、事件 trace |
| 第 2 周 | 9～11 小时 | Provider 与 Agent 循环如何协作 | 一次 tool-call 循环时序图 |
| 第 3 周 | 9～11 小时 | Session、分支和上下文如何重建 | 手工上下文投影、compaction 演算 |
| 第 4 周 | 9～11 小时 | 如何用扩展机制改变 Pi | 一个可持久化扩展、机制选型表 |
| 第 5 周 | 9～11 小时 | 如何嵌入 Pi 并理解终端 UI | SDK/RPC 小客户端、渲染流程图 |
| 第 6 周 | 9～11 小时 | 如何处理故障、安全和完整工程流程 | 综合项目、威胁模型、验证报告 |
| 进阶第 7 周 | 6～8 小时 | 支撑包各自解决什么问题 | durable/chord 等包职责图 |
| 进阶第 8 周 | 6～8 小时 | 如何形成可维护贡献 | 一个小改动及定向测试 |

## 4. 每周详细计划

## 第 1 周：建立整体模型，先会观察再读实现

**目标**：不把 Pi 误解为“一个 CLI 加几个模型请求”，先看到统一的 Agent/Session 核心。

### 第 1 天：仓库和产品边界

- 阅读[根 README](../README.md)、根 `package.json` 和 [coding-agent README](../packages/coding-agent/README.md)。
- 列出所有 workspace 包，先只记录“一句话职责”，不深入支撑包。
- 找出 Node 版本、workspaces、构建顺序、检查和测试入口。
- 用一句话回答：Pi 是模型、模型 Provider、Agent harness，还是 TUI？

**产出**：一页仓库地图；标出 `ai → agent → coding-agent` 的学习顺序，以及 TUI、接口和扩展分别位于哪里。

### 第 2 天：交互模式与基本资源

- 阅读[文档索引](../packages/coding-agent/docs/index.md)、[How Pi Works](../packages/coding-agent/docs/how-pi-works.md) 和[交互使用](../packages/coding-agent/docs/usage.md)。
- 在有模型访问条件时，于临时目录执行一个只读任务，例如“说明当前目录结构”。将默认工具限制为 `read,grep,find,ls`。
- 观察 system prompt、上下文文件、工具声明、skill 描述分别何时进入请求。
- 体验 `/model`、`/thinking`、`/session`、`/new`、`/resume`。

**产出**：一张“用户输入之前，Pi 已经装配了什么”的清单。

### 第 3 天：Print、JSON、RPC、SDK 的差异

- 阅读[接口选择](../packages/coding-agent/docs/cli-integration.md)、[CLI](../packages/coding-agent/docs/cli.md) 和 [JSON 事件流](../packages/coding-agent/docs/json.md)。
- 比较交互、print、JSON、RPC 的输入、输出、进程生命周期和适用场景。
- 有条件时保存一次 JSONL 事件流，给每条记录标注 `agent/turn/message/tool/queue` 类别。
- 没有模型时，直接以文档中的事件序列为样本完成同样标注。

**产出**：四种 CLI 接口加 SDK 的对比表；解释“JSON mode”为什么不是“要求模型输出 JSON”。

### 第 4 天：队列与结束条件

- 重读 [How Pi Works 的 Agent loop](../packages/coding-agent/docs/how-pi-works.md#agent-loop) 和 [JSON 事件序列](../packages/coding-agent/docs/json.md#event-sequence)。
- 用一个包含两次工具调用的假设任务，写出 `turn_start`、`message_end`、`tool_execution_*`、`turn_end`、`agent_end` 的顺序。
- 解释 steering 与 follow-up 分别何时进入。
- 专门回答：为什么收到 `agent_end` 后仍不能立即关闭客户端？

**产出**：一个两 turn 的 trace；用红字标出 `agent_settled` 才是无后续自动工作的信号。

### 第 5 天：配置与信任边界

- 阅读[配置](../packages/coding-agent/docs/configuration.md)和[安全](../packages/coding-agent/docs/security.md)。
- 对比用户级 `<agent-dir>`、项目级 `.pi`、上下文文件 `AGENTS.md`/`CLAUDE.md` 的发现规则。
- 列出 project trust 保护和不保护的内容。
- 检查一个测试目录启动时实际加载了哪些资源，不把真实凭据写入笔记。

**产出**：资源加载顺序图；一段“trust 不是 sandbox”的具体解释。

**第 1 周验收**：

- [ ] 能在 3 分钟内向别人说明 Pi 的定位和主链路。
- [ ] 能区分 turn、run、`agent_end`、`agent_settled`。
- [ ] 能解释 stdout 在 JSON/RPC 模式为什么必须只放协议记录。
- [ ] 能指出项目 trust 不限制工具调用权限。

## 第 2 周：掌握 Provider 抽象与 Agent 循环

**目标**：从“模型返回文本”深入到“流式消息、工具调用、状态和事件共同驱动循环”。

### 第 6 天：统一消息模型

- 阅读[消息类型](../packages/coding-agent/docs/message-types.md)和 `packages/ai/src/types.ts`。
- 对比 user、assistant、toolResult、system 的字段。
- 分析 text、image、thinking、toolCall 四类内容块为什么不能用单一字符串替代。
- 记录 `stopReason`、`usage`、provider 签名字段各自服务什么目的。

**产出**：一个包含文本和工具调用的 assistant message 示例；解释 `message_end` 为什么是最终权威值。

### 第 7 天：Provider 适配与流式事件

- 阅读 `packages/ai/README.md` 和[自定义 Provider](../packages/coding-agent/docs/custom-provider.md)。
- 选择一个内置 API 实现，跟踪“统一消息 → Provider 请求 → Provider 事件 → 统一 assistant 事件”。
- 找出 `start`、内容 delta、`done`/`error` 的平衡约束。
- 理解 system prompt 和工具声明为什么放进 transcript system messages。

**产出**：Provider adapter 时序图；列出自定义流实现必须满足的五个条件。

### 第 8 天：Agent 状态与主循环

- 阅读 `packages/agent/README.md`、`packages/agent/src/types.ts` 和 Agent 核心循环。
- 找出消息数组、当前模型、工具集合、streaming 状态和事件订阅的位置。
- 跟踪无工具调用时循环如何结束，有工具调用时为什么开始下一 turn。
- 区分底层 Agent 消息与 coding-agent 扩展出的消息角色。

**产出**：一份 Agent 状态字段表；一张主循环流程图。

### 第 9 天：工具执行、并发与取消

- 阅读工具执行相关实现和测试。
- 跟踪参数校验、`tool_call`、执行、`tool_result`、结果消息的顺序。
- 研究同一 assistant message 中多个工具调用何时可并行，何时需要 sequential。
- 跟踪 `AbortSignal` 如何穿过模型请求和工具执行。

**产出**：两个并行工具加一个失败工具的 trace；说明失败如何成为模型可见结果，而不一定使进程崩溃。

### 第 10 天：用 faux provider 固定理解

- 找到 Agent 和 coding-agent 测试中的 faux provider。
- 阅读一个“文本响应”测试、一个“工具后继续”测试和一个“中断/重试”测试。
- 若修改或新增实验测试，只运行对应测试文件并迭代到通过。
- 不看实现，独立写出一次 prompt 的调用链，再与源码核对。

**产出**：一份经过源码纠正的 end-to-end trace。

**第 2 周验收**：

- [ ] 能说明 Provider adapter 与 Agent loop 各自不负责什么。
- [ ] 能解释流式 delta 与最终完整消息的关系。
- [ ] 能解释工具结果为什么会触发下一次模型请求。
- [ ] 能指出并发工具共享可变状态时的风险和对应执行模式。

## 第 3 周：掌握 Session 树、上下文投影与压缩

**目标**：理解“磁盘原始历史”“当前分支”“实际发给模型的消息”是三个不同层次。

### 第 11 天：AgentSession 与 SessionManager

- 阅读 [SDK Session 生命周期](../packages/coding-agent/docs/sdk.md#session-lifecycle)。
- 阅读 `packages/coding-agent/src/core/session-manager.ts` 的条目类型、活动 leaf 和分支导航。
- 找出 `AgentSession` 拥有的模型、工具、队列、compaction 和扩展运行时。
- 明确 `SessionManager` 才是持久化模型上下文的权威来源。

**产出**：两个核心对象的职责表；解释为什么不能靠直接替换 Agent 的消息数组恢复历史。

### 第 12 天：JSONL 树和分支

- 完整阅读[Session 文件格式](../packages/coding-agent/docs/session-format.md)。
- 解析一个测试 Session：第一行 header，其余条目按 `id/parentId` 建树。
- 对 `/tree`、`/fork`、`/clone` 分别画出“原文件是否改变、是否产生新文件、当前 leaf 如何变化”。
- 验证非当前分支仍保留在文件中，但不会全部进入模型上下文。

**产出**：一个至少含两个分支的手绘树；写出当前分支的 ID 路径。

### 第 13 天：从条目到模型上下文

- 研究 `buildContextEntries()`、`buildSessionProjection()`、`buildSessionContext()` 的输入输出。
- 将 message、custom、custom_message、usage、model_change、branch_summary 分类为“进入模型上下文/只记录状态/只用于展示”。
- 研究 `context_edit` 的 `replacement: null` 和非空替换。
- 分析 system message 的 sections、`toolsAdded`、`toolsRemoved` 如何重放。

**产出**：一张三层表：`raw entries → selected/projected entries → LLM messages`。

### 第 14 天：Compaction 算法

- 完整阅读[Compaction Reference](../packages/coding-agent/docs/compaction.md)。
- 用具体数字演算触发条件：`contextTokens > contextWindow - reserveTokens`。
- 找一个 cut point，说明为什么不能在 tool result 前切断而使其与 tool call 分离。
- 解释 `firstKeptEntryId`、summary、system checkpoint 和 `tokensBefore`。
- 推演第二次 compaction 为什么可能再次包含上次保留的消息。

**产出**： compaction 前后各一张图；回答“为什么 compaction 降低的是模型上下文，不等于删除磁盘历史”。

### 第 15 天：内置工具与输出预算

- 阅读 `read`、`bash`、`edit`、`write` 的实现，再按需看 `grep/find/ls/powershell`。
- 对比模型可见 `content`、渲染/恢复用 `details`、结构化 `structuredContent`。
- 找到长输出截断和完整输出路径的逻辑。
- 研究文件修改为什么需要 mutation queue，嵌套工具为什么要有调用边界。

**产出**：一次“读取大文件 → 截断展示 → 模型按路径继续读取”的具体说明。

**第 3 周验收**：

- [ ] 给一个 session JSONL，能手算 active branch。
- [ ] 能解释 branch、fork、clone 的区别。
- [ ] 能说明 `context_edit` 为什么不会修改原始消息。
- [ ] 能独立推导一次 compaction 后模型实际看到的消息序列。

## 第 4 周：扩展、Skills、MCP 与 Codemode

**目标**：掌握“用最小能力解决问题”的扩展选型，并完成一个真正可恢复的扩展。

### 第 16 天：最小扩展与事件链

- 阅读[Extensions](../packages/coding-agent/docs/extensions.md) 和 [`hello.ts`](../packages/coding-agent/examples/extensions/hello.ts)。
- 写一个注册 `/hello` 命令或 `hello` 工具的最小扩展。
- 用 `pi --extension ./hello.ts` 显式加载。
- 添加一个只记录事件的 handler，观察加载顺序、执行顺序和返回值。

**产出**：最小扩展；说明 jiti 让本地 TypeScript 免于单独编译，但不改变其进程内权限。

### 第 17 天：自定义工具与状态

- 阅读 [`todo.ts`](../packages/coding-agent/examples/extensions/todo.ts) 和扩展示例中的状态恢复模式。
- 为工具定义 TypeBox 参数、模型描述、`content` 和 `details`。
- 在 `session_start` 中只从 `getBranch()` 重建状态。
- 分别测试继续当前分支和切换到旧分支，观察状态是否跟随分支。

**产出**：一个 branch-aware 的计数器或 todo 工具；附状态恢复 trace。

### 第 18 天：Prompt、Skill、Theme、Package

- 阅读[Prompt Templates](../packages/coding-agent/docs/prompt-templates.md)、[Skills](../packages/coding-agent/docs/skills.md)、[Themes](../packages/coding-agent/docs/themes.md) 和 [Pi Packages](../packages/coding-agent/docs/packages.md)。
- 创建一个带参数和默认值的 prompt template。
- 创建一个只有 `SKILL.md` 的 skill，观察启动时只暴露名称、描述和路径。
- 将两者和一个扩展放进同一包目录，写出资源发现规则。

**产出**：机制选型表：固定文本、按需说明、可执行行为、视觉主题、资源分发分别选什么。

### 第 19 天：MCP 与工具暴露

- 阅读[MCP](../packages/coding-agent/docs/mcp.md) 的传输、配置和 exposure 部分。
- 理解 stdio 与 streamable HTTP、工具命名、后台连接、OAuth 的基本路径。
- 对比 `direct`、`codemode`、`deferred`、`hidden`。
- 明确 annotations 是服务器声明的提示，不是 Pi 已验证的安全事实。

**产出**：一个包含大量工具的 MCP server 应如何选择 exposure 的决策记录。

### 第 20 天：Codemode 与权限扩展

- 阅读[Codemode](../packages/coding-agent/docs/codemode.md)。
- 用一个脚本示例说明并行调用、过滤大结果和只返回摘要如何减少模型上下文。
- 列出 QuickJS 中没有的能力，以及工具调用仍然产生的真实副作用。
- 阅读[`permission-gate.ts`](../packages/coding-agent/examples/extensions/permission-gate.ts)，补充非交互模式的默认行为分析。

**产出**：一段“codemode sandbox 不等于工具 sandbox”的解释；一个权限 gate 改进方案。

**第 4 周验收**：

- [ ] 扩展在 resume 和 branch 后不会混入其他分支状态。
- [ ] 工具异常、被 block、返回 `isError` 的区别已写清楚。
- [ ] 能解释 skill 的 progressive disclosure。
- [ ] 能说明 MCP/codemode 的工具调用仍经过 Pi 工具管线和扩展事件。

## 第 5 周：SDK、RPC 与终端 UI

**目标**：把 Pi 当作可嵌入运行时，而不只是交互式命令。

### 第 21 天：最小 SDK 应用

- 阅读[SDK](../packages/coding-agent/docs/sdk.md) 和[`01-minimal.ts`](../packages/coding-agent/examples/sdk/01-minimal.ts)。
- 创建 session，先订阅事件再调用 `prompt()`。
- 只输出 `text_delta`，在 `message_end` 用完整消息校正重组结果。
- 在 `finally` 中调用 `dispose()`。

**产出**：最小 SDK 程序；列出默认创建的 ModelRuntime、SettingsManager、SessionManager 和 ResourceLoader。

### 第 22 天：自定义 SDK 边界

- 阅读[`11-sessions.ts`](../packages/coding-agent/examples/sdk/11-sessions.ts) 和 `12-full-control.ts`。
- 分别使用内存 Session、只读工具集、自定义 system prompt。
- 对比默认资源发现与完全自定义 ResourceLoader。
- 阅读 `13-session-runtime.ts`，说明 runtime 替换 session 后为什么必须重新绑定订阅。

**产出**：一个无磁盘 Session 的只读 SDK 配置；一份资源依赖清单。

### 第 23 天：RPC 客户端

- 阅读[RPC](../packages/coding-agent/docs/rpc.md) 和[`rpc-client.ts`](../packages/coding-agent/examples/rpc-client.ts)。
- 为命令设置唯一 ID，并将 response 与 ID 关联。
- 在发送 prompt 前订阅事件，等待 `agent_settled` 而不是只等 command response。
- 按 LF 分割 JSONL，持续读取 stdout，并单独处理 stderr。

**产出**：RPC 客户端状态机：启动、发送、流式接收、取消、关闭。

### 第 24 天：TUI 组件与差量渲染

- 阅读 `packages/tui/README.md` 和[Terminal UI](../packages/coding-agent/docs/tui.md)。
- 理解组件 `render(width)` 为什么返回终端行，以及宽度为什么要按可见列计算。
- 跟踪状态变化、`invalidate()`、`requestRender()`、合并渲染和终端写入。
- 研究 focus、overlay、硬件光标和 IME 的关系。

**产出**：一次按键到屏幕更新的渲染 trace；一个窄宽度/宽字符测试清单。

### 第 25 天：Provider 扩展与虚拟模型

- 阅读[Custom Providers](../packages/coding-agent/docs/custom-provider.md) 和[Virtual Models](../packages/coding-agent/docs/virtual-models.md)。
- 比较 `models.json`、复用内置 stream、完全自定义 stream 三种集成成本。
- 解释虚拟模型的“用户选择”与“每次请求实际派发”为什么需要分别记录。
- 分析 continuation/retry 保持物理模型稳定对缓存和 Provider 签名的意义。

**产出**：一个路由决策表；明确这是进阶可选能力，不是理解主循环的前置条件。

**第 5 周验收**：

- [ ] SDK/RPC 客户端不会把“prompt 已接受”误判为“工作已完成”。
- [ ] 流式重组最终以 `message_end` 为准。
- [ ] 能解释 TUI 为什么需要缓存失效和可见宽度计算。
- [ ] 能为自定义模型服务选择最小集成方式。

## 第 6 周：可靠性、安全、工程化与综合项目

**目标**：把分散机制组合成可维护系统，并验证边界条件。

### 第 26 天：重试与上下文溢出恢复

- 阅读[网络与重试设置](../packages/coding-agent/docs/settings.md#network-and-retries)及 [JSON Retry Events](../packages/coding-agent/docs/json.md#retry-events)。
- 对比 Provider 级重试和 Agent 级重试的默认值与职责。
- 画出 transient error、context overflow、output length 三类路径。
- 解释恢复为什么可能需要先保留失败响应和结束事件，再修复未来上下文并重试。

**产出**：故障分类表和恢复时序图。

### 第 27 天：威胁模型与隔离

- 重读[安全](../packages/coding-agent/docs/security.md)和[容器化](../packages/coding-agent/docs/containerization.md)。
- 分别分析不可信仓库、不可信扩展、prompt injection、危险命令、凭据泄漏。
- 对比直接运行、权限扩展、整进程容器、仅工具委托到隔离环境。
- 检查 session 导出可能包含的敏感内容。

**产出**：一张“资产—攻击面—现有控制—仍需外部控制”的表格。

### 第 28 天：工程化和供应链

- 阅读根构建/检查脚本及代表性检查脚本。
- 说明构建顺序为什么反映包依赖。
- 了解 erasable TypeScript、ESM、生成模型数据和安装锁的约束。
- 运行 `npm run check`；若只改文档，说明为什么不需要运行测试。

**产出**：一张从源码到 npm 包/独立二进制的工程流程图。

### 第 29 天：综合项目实现

从第 5 节选择一个项目，先写最小验收用例，再实现。限制范围，不在同一天同时加入自定义 Provider、复杂 TUI 和 MCP。

**产出**：可运行实现、架构说明和已知限制。

### 第 30 天：验证与讲解

- 运行与改动相关的定向测试，再按仓库要求检查。
- 验证正常完成、工具失败、取消、恢复、非交互模式和 Session 恢复。
- 用一次具体输入做 10 分钟讲解：架构、调用链、关键取舍、安全边界。
- 记录未覆盖内容，不把未测试路径写成已验证。

**产出**：验证报告和最终架构图。

**第 6 周验收**：

- [ ] 综合项目有明确进程边界和权限边界。
- [ ] 失败与取消路径均有实际验证或明确标注未覆盖。
- [ ] 文档中的行为描述能定位到源码或测试。
- [ ] 没有把 project trust、扩展 hook 或 QuickJS 当成完整安全沙箱。

## 5. 综合项目选题

### 方案 A：分支感知的任务管理扩展

**适合重点**：Extension API、工具、Session 状态。

最低要求：

- 一个 TypeBox 参数定义的 todo 工具。
- `/todos` 命令和简单 renderer。
- 状态保存在 tool result `details` 或 custom entry 中，并说明选择理由。
- `session_start` 只从当前分支重建。
- 验证 resume、`/tree` 和 fork 后状态不串线。
- 非交互模式下不调用不存在的 TUI 能力。

### 方案 B：可嵌入的 RPC 任务客户端

**适合重点**：事件协议、进程生命周期、取消。

最低要求：

- 启动 `pi --mode rpc --no-session`。
- 命令 ID 关联响应；stdout 严格 JSONL，stderr 单独展示。
- 实时重组文本和工具事件，以 `message_end` 校正。
- 等待 `agent_settled`，支持 abort 和有序关闭。
- 处理子进程启动失败、异常退出和客户端超时。

### 方案 C：带权限门的只读代码助手

**适合重点**：工具暴露、扩展事件、安全边界。

最低要求：

- 默认只启用 `read,grep,find,ls`。
- 对额外工具使用 `tool_call` 权限门，非交互模式默认拒绝危险操作。
- 用 SDK 或 RPC 嵌入一个简单宿主。
- 明确权限 hook 不是操作系统安全边界，并给出容器化部署说明。

### 方案 D：自定义 Provider 协议适配器

**适合重点**：pi-ai、流式归一化、模型运行时。

最低要求：

- 优先复用内置 API stream；只有协议无法表达时才完全自定义。
- 支持文本、工具调用、usage、abort 和终止事件。
- 测试空响应、 malformed stream、Unicode、上下文溢出和跨 Provider Session。
- 不在日志中记录凭据或完整敏感响应。

## 6. 进阶两周

### 第 7 周：支撑包

按“问题—核心 API—一个实现文件—是否进入 coding-agent 主链路”分析：

1. `packages/durable`：持久 conversation/task/document 运行时的边界，以及它与 coding-agent JSONL Session 的关系。
2. `packages/chord`：热重载插件、contract、复制状态和 JSON link RPC。
3. `packages/codemode` 与 `packages/mcp`：协议/执行底层和 coding-agent 内置扩展的衔接。
4. `packages/env`：环境差异抽象。
5. `packages/telemetry` 与 `packages/evals`：可观测性契约和评测，不把它们误当成 Agent 主循环必需条件。

**产出**：支撑包依赖图；每个包一个“什么时候应该使用”的具体场景。

### 第 8 周：小型贡献演练

1. 选择一个有明确行为边界的小问题，不以大规模重构作为入门贡献。
2. 完整阅读涉及文件和相关测试。
3. 先添加能复现问题的定向测试，再修改实现。
4. 运行该测试文件并迭代到通过，再运行 `npm run check`。
5. 只暂存本次修改的明确路径，检查 `git status`。
6. 若涉及 GitHub issue 的回归测试，在测试旁写 issue 编号。
7. 不直接修改生成的模型文件，不静默修改锁文件。

**产出**：改动说明，包含问题、最小复现、根因、方案、验证和未覆盖范围。

## 7. 推荐源码阅读顺序

不要按目录字母顺序通读。沿一条调用链阅读：

| 顺序 | 阅读对象 | 阅读时要回答的问题 |
|---:|---|---|
| 1 | `packages/ai/src/types.ts` | 统一消息、内容块、流式事件是什么 |
| 2 | `packages/agent/src/types.ts` | Agent 如何扩展消息和状态 |
| 3 | Agent 核心循环 | 何时请求模型、何时执行工具、何时结束 |
| 4 | `packages/coding-agent/src/core/session-manager.ts` | 条目树、leaf、分支和上下文如何保存 |
| 5 | AgentSession 与 Session 工厂 | 模型、工具、队列、扩展和资源如何组装 |
| 6 | `packages/coding-agent/src/core/compaction/compaction.ts` | 何时压缩、保留哪里、如何恢复 |
| 7 | `packages/coding-agent/src/core/extensions/types.ts` | 哪些事件可观察、转换或取消 |
| 8 | `packages/coding-agent/src/modes/json-event.ts` | SDK 事件如何变成线性 JSONL |
| 9 | `packages/coding-agent/src/modes/rpc/rpc-types.ts` | 命令、响应和事件如何分帧 |
| 10 | `packages/tui/src/index.ts` 及渲染核心 | 组件输出如何变成终端写入 |

每读一个文件，先找公开入口和类型，再找状态写点，最后找事件和错误出口。不要从工具函数列表开始背实现。

## 8. 自测题与评分

总分 100，达到 80 且没有安全边界误判，可认为完成主线。

| 能力 | 分值 | 自测题 |
|---|---:|---|
| 架构 | 15 | 为什么多个接口可以共享同一 Agent/Session 核心？ |
| Agent 循环 | 20 | 一个 assistant 产生两个 tool call 后，事件和下一 turn 如何发生？ |
| Provider | 10 | 自定义 stream 为什么必须平衡 start/content/terminal events？ |
| Session | 20 | 当前文件有 100 个条目，为什么模型可能只看到其中 20 个？ |
| 扩展 | 15 | tool `content`、`details`、custom entry 分别保存什么？ |
| 嵌入 | 10 | RPC prompt 返回成功后，为什么还要继续读事件？ |
| 安全与工程 | 10 | 为什么 trust、QuickJS 和权限 hook 都不能单独替代 OS 隔离？ |

额外的口头自测：

- 不看资料画出 `prompt → model → tools → model → settled`。
- 用具体 ID 演示 branch 和 compaction 后的上下文。
- 说明 `agent_end` 与 `agent_settled` 的差别。
- 说明“Session 持久化”和“模型上下文”为什么不是同一份数据。
- 给一个需求，选择 prompt、skill、extension 或 MCP，并说明不选其他机制的理由。

## 9. 常见误区清单

- **误区：Pi 是一个新的大模型。** 它是协调模型、工具、上下文、会话和界面的 Agent harness。
- **误区：Session JSONL 是线性聊天记录。** 条目通过 `id/parentId` 形成树。
- **误区：compaction 删除旧消息。** 它改变后续模型上下文，原始条目仍保留。
- **误区：直接替换 Agent 的消息数组即可恢复 Session。** 持久化上下文以 SessionManager 为准。
- **误区：skill 全文始终占用上下文。** 启动时主要注入路由信息，完整说明按需读取。
- **误区：扩展像独立 sandbox 插件一样运行。** 扩展在 Pi 进程内执行，拥有相同操作系统权限。
- **误区：project trust 会限制 bash 或文件工具。** 它主要控制项目资源加载。
- **误区：codemode 是无副作用的安全执行环境。** VM 限制脚本自身能力，但它调用的工具仍可真实修改外部状态。
- **误区：JSON mode 要求模型输出 JSON。** 它输出的是 Pi 的结构化事件流。
- **误区：`agent_end` 表示整个任务彻底结束。** 自动重试、恢复和排队消息仍可能继续。
- **误区：plan mode 和 subagent 必须修改核心才能获得。** 仓库提供了扩展示例；核心刻意保持可组合。

## 10. 每周复盘模板

```markdown
# 第 N 周复盘

## 我能独立解释
- 

## 我画出的调用链
- 

## 我完成的最小实验
- 输入：
- 观察：
- 结论：

## 我纠正的误解
- 

## 仍未验证
- 

## 下周唯一最重要的目标
- 
```
