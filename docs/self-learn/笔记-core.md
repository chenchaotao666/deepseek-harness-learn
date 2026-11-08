# packages/core 分层学习笔记

## 概览

`packages/core` 实现 Agent 的公共接口和运行机制：接收任务、组织上下文、调用模型、执行工具、记录过程，再决定是否继续。这份源码阅读教程面向了解 TypeScript 基础、刚接触 Cordis 和 Agent 的读者，以“读取 README 并说明启动方式”为例，串起各模块。示例和伪代码用于理解源码，不是独立可运行的应用。

学习时，建议按一次任务的数据流阅读，而不是按目录顺序从头读。下面按职责分成五层；这是便于理解的阅读分层，并不是仓库强制规定的依赖层级。完整架构以[架构说明](../architecture.md)和各模块的 README 为准。

## 目录

- [一、模块全貌](#一模块全貌)
- [二、第一层：session——理解发生过什么](#二第一层session理解发生过什么)
- [三、第二层：agent 和 scope——理解谁在运行、能用什么](#三第二层agent-和-scope理解谁在运行能用什么)
- [四、第三层：system-prompt 与 tools——准备模型输入](#四第三层system-prompt-与-tools准备模型输入)
- [五、第四层：工具执行器——模型动作如何落地](#五第四层工具执行器模型动作如何落地)
- [六、第五层：agent-loop——串起执行过程](#六第五层agent-loop串起执行过程)
- [七、完整例子](#七完整例子)
- [八、建议阅读顺序](#八建议阅读顺序)

-----

## 一、模块全貌

| 层次 | 模块 | 主要问题 |
|---|---|---|
| 基础隔离 | `scope` | 不同 Agent 的工具、提示词和事件如何隔离？ |
| 数据记录 | `session` | 发生过什么？下一次请求应该包含哪些历史？ |
| 对外接口 | `agent` | 如何创建、使用、取消和恢复一个 Agent？ |
| 模型选择 | `agent-default-model` | 新 Agent 默认用哪个模型？ |
| 请求准备 | `system-prompt` | 模型收到什么提示词、上下文和工具说明？ |
| 工具管理 | `tools` | 有哪些工具？当前 Agent 可以使用哪些？ |
| 工具执行 | `tool-executor-native` | 如何执行模型直接输出的工具调用？ |
| 工具执行 | `tool-executor-ptc` | 如何让模型通过程序调用工具？ |
| 执行驱动 | `agent-loop` | 如何让上述过程反复运行直到结束？ |
| 兼容配置 | `agent-tool-presentation` | 旧配置如何选择工具模式？该模块已弃用，新配置直接挂载执行器。 |

模块职责总览见 [core README](../../packages/core/README.md)，旧工具模式选择器的兼容说明见 [agent-tool-presentation README](../../packages/core/agent-tool-presentation/README.md)。

有两个容易误解的地方：

- LLM 接入不在 core 内，它位于 `packages/llm/`；core 通过 `ctx.llm` 调用。
- 读文件、联网搜索等具体工具不在 core 内；core 提供注册和执行机制，具体工具由其他插件实现。

一次任务的大致关系：

```text
用户输入
   ↓
Agent 接口 → Inbox 待处理消息
   ↓
agent-loop
   ├─ system-prompt：组装提示词
   ├─ tools：确定可用工具
   ├─ session：取得当前对话历史
   ↓
ctx.llm：调用模型
   ↓
模型响应
   ├─ 普通回答 → 记录，可能结束
   └─ 工具调用 → 执行器 → 具体工具
                            ↓
                       结果写入 Session
                            ↓
                       进入下一步
```

-----

## 二、第一层：session——理解发生过什么

阅读入口：

- [session/src/types.ts](../../packages/core/session/src/types.ts)：会话事件类型。
- [session/src/index.ts](../../packages/core/session/src/index.ts)：`Session`、`Session.append()` 和 `SessionStore`。
- [session/src/surface.ts](../../packages/core/session/src/surface.ts)：当前有效对话内容的推导。

核心思想是：**Session 保存按顺序追加的事件；模型看到的对话历史由这些事件推导出来。**

它不只是下面这样的聊天数组：

```json
[
  { "role": "user", "content": "你好" },
  { "role": "assistant", "content": "你好！" }
]
```

它还记录任务运行过程，例如：

```text
turn/start
step/start
user/message
assistant/message
tool/call
tool/result
step/end
turn/end
```

每个事件带有类型、序号、时间和具体数据。上面仅列出部分事件，实际日志还会包含系统消息、请求配置等记录。

例如用户说：“读取 README.md，告诉我项目怎么启动。”会产生用户消息、模型工具调用、工具结果、最终回答等事件。这些事件既支持后续模型请求，也支持界面展示、历史查询和恢复。

### 为什么还要有 surface.ts？

“所有历史事件”与“现在应该发给模型的消息”并不相同：

- `step/start` 是运行记录，不是聊天内容。
- 部分旧消息可能已经被压缩或替换。
- 工具调用和工具结果需要组成正确的消息关系。

因此可以理解为：

```text
完整 Session 事件日志
           ↓
    消息投影与 surface 处理
           ↓
    当前有效的模型对话历史
```

这里的 `surface` 指当前有效的对话内容，不是 UI 页面。

还要区分：`core/session` 的事件和内存管理，不等于具体的磁盘持久化实现。持久化和 SQLite 搜索索引由其他模块配合完成。具体语义见 [Session 子系统](../subsystems/session.md)。

读这一层时，重点回答：一个事件写进去之后，如何变成模型下一次看到的消息？

-----

## 三、第二层：agent 和 scope——理解谁在运行、能用什么

阅读入口：

- [agent/src/index.ts](../../packages/core/agent/src/index.ts)：Agent 注册表和创建入口。
- [agent/src/runtime-types.ts](../../packages/core/agent/src/runtime-types.ts)：Agent 运行时接口。
- [scope README](../../packages/core/scope/README.md)：注册项和事件的作用域。

`agent` 定义公共接口，`agent-loop` 提供默认实现。可以用下面的简化关系理解：

```text
调用方
  ↓
ctx.agents.create(...)
  ↓
已注册的 AgentFactory
  ↓
agent-loop 创建具体 Agent
```

实际的工厂注册在 [agent-loop/src/index.ts](../../packages/core/agent-loop/src/index.ts) 中，搜索 `agentLoop.setFactory()` 即可定位：

```text
ctx.effect(() => ctx.agents.setFactory(this), 'agentLoop.setFactory()')
```

插件通常面向 `Agent` 接口编程，不必直接依赖循环内部实现。上面的 `ctx.effect()` 让工厂注册随插件生命周期清理。

### scope 如何隔离 Agent？

`scope` 控制不同 Agent 的贡献范围。例如：

```text
公共作用域：read、grep
   ├─ Agent A：额外注册 edit
   └─ Agent B：额外注册 web_search
```

如果没有其他限制：

- A 可以继承公共工具，并使用自己的 `edit`。
- B 可以继承公共工具，并使用自己的 `web_search`。
- A 的局部注册不会自动出现在 B 中。

作用域隔离不等于操作系统沙箱。它主要决定注册项和事件的可见范围；文件、进程权限由相应策略和沙箱模块管理。

`agent-default-model` 提供部署默认模型选择，由入口用于尚无自身选择的新 Agent。它不是调用模型 API 的实现，详见 [agent-default-model README](../../packages/core/agent-default-model/README.md)。

-----

## 四、第三层：system-prompt 与 tools——准备模型输入

这一层处理两个问题：告诉模型什么？允许模型调用什么？

### 1. system-prompt：组装提示词

关键入口是 [system-prompt/src/index.ts](../../packages/core/system-prompt/src/index.ts) 中的 `SystemPrompt.assemble()`。

不同插件可以贡献提示词段落、动态上下文和工具相关信息，最后按规则组装。例如概念上：

```text
角色要求：你是编码助手
工作环境：当前工作目录、运行环境
工具指导：什么时候使用 read、grep
工具定义：工具名称、描述、参数结构
```

这里需要区分：提示词文本告诉模型行为要求；工具 schema 告诉模型可调用的函数和参数。两者都进入模型请求，但不是简单地全部拼成一段字符串。详见 [system-prompt README](../../packages/core/system-prompt/README.md)。

### 2. tools：工具注册、可用性与分发

在 [tools/src/index.ts](../../packages/core/tools/src/index.ts) 中重点看三个方法：

- `register()`：注册工具定义。
- `resolve()`：确定当前步骤可用的工具集合。
- `execute()`：把工具调用交给执行器。

一个工具可以概念化为以下几部分。下面是结构示意，不是完整注册代码：

```text
{
  name: 'read',
  description: '读取文件内容',
  parameters: 参数定义,
  execute: 实际读取逻辑,
}
```

模型看到名称、描述和参数定义，不会直接执行这里的 JavaScript 函数。它输出工具调用，由 Harness 解析和执行：

```text
模型输出：调用 read，参数为 README.md
                       ↓
Harness 查找对应工具
                       ↓
执行工具并返回结果
```

当前步骤先确定工具集合，再用这份集合声明和执行工具，让“告诉模型能用什么”和“实际允许调用什么”保持一致。注册和执行的完整语义见 [tools README](../../packages/core/tools/README.md)。

-----

## 五、第四层：工具执行器——模型动作如何落地

### Native 模式：直接调用工具

[tool-executor-native/src/index.ts](../../packages/core/tool-executor-native/src/index.ts) 很适合入门阅读。它的核心职责相当简短：

```text
describe(tools)
  → 将工具定义转换为模型可见的 schema

execute(input)
  → 把整个调用批次交给共享执行机制
```

调用路径：

```text
模型输出工具调用
       ↓
ToolsService.execute()
       ↓
NativeToolExecutor
       ↓
批次调度 + ToolExecutionPipeline
       ↓
具体工具 execute()
```

需要继续深入时，再看：

- [tools/src/batch.ts](../../packages/core/tools/src/batch.ts)：批次调度、并发与调用顺序。
- [tools/src/pipeline.ts](../../packages/core/tools/src/pipeline.ts)：单次调用的策略检查、取消和结果处理。

### 为什么不直接 Promise.all() 执行所有工具？

工具不一定可以并行。例如一个调用修改文件，另一个调用读取刚修改的文件。如果随意并行，读取调用可能读到旧内容。执行机制需要区分可并行调用和需要排他执行的调用。

### PTC 模式：通过程序调用工具

[tool-executor-ptc](../../packages/core/tool-executor-ptc/README.md) 让模型直接调用 `run_code`，再由程序使用生成的工具 SDK。概念上：

```text
Native：
模型 → read 工具
模型 → grep 工具

PTC：
模型 → run_code
           ├─ 调用读取工具
           ├─ 调用搜索工具
           └─ 用程序整理结果
```

两种模式共享部分工具执行机制，但模型看到的调用方式不同。第一次学习建议先读 Native，再读 PTC。

-----

## 六、第五层：agent-loop——串起执行过程

这是理解 Agent 行为最重要的部分，但建议放在前面几层之后读。

| 文件 | 职责 |
|---|---|
| [agent-loop/src/index.ts](../../packages/core/agent-loop/src/index.ts) | 插件配置、Agent 工厂、创建与生命周期管理 |
| [agent-loop/src/agent.ts](../../packages/core/agent-loop/src/agent.ts) | 单个 Agent 的输入、状态、turn 和 step 执行 |

先掌握三个概念：

| 概念 | 含义 |
|---|---|
| Session | 整段会话的事件记录 |
| Turn | 一轮处理过程，可能包含多次模型交互 |
| Step | 一次模型响应及其对应的工具处理；失败重试可能仍在同一步内 |

例如：

```text
Session
  └─ Turn 1：读取 README 并解释启动方式
       ├─ Step 1：模型要求读取 README → 执行 read
       └─ Step 2：模型读取结果 → 输出回答
```

沿着以下方法读最有效：

```text
send()
  ↓
wakeDriver()
  ↓
kick()
  ↓
turn()
  ↓
preStep()
  ↓
step()
```

### 1. send()：输入先进入队列

在 `agent-loop/src/agent.ts` 中找到 `send()`。输入有两个主要目标：

- `next-turn`：下一轮处理。
- `next-step`：当前活动后续步骤可接收的输入。

这解释了为什么 Agent 工作时还能接收用户补充。输入并不意味着立刻启动一个并发模型请求。队列的具体实现见 [agent-loop/src/inbox.ts](../../packages/core/agent-loop/src/inbox.ts)。

### 2. preStep()：准备下一步

`preStep()` 主要完成以下工作：

```text
领取待处理消息
  ↓
读取当前模型选择
  ↓
解析当前可用工具
  ↓
组装系统提示词
  ↓
调用 agent/pre-step 扩展点
```

插件可以在这里参与步骤准备；循环不需要把所有扩展功能都写死在自身内部。

### 3. step()：调用模型并处理响应

核心流程可以简化成下面的伪代码，省略重试、取消、日志和生命周期细节：

```text
async function step() {
  const request = buildRequest()
  const response = await collectModelStream(request)

  recordAssistantMessage(response)

  if (response.toolCalls.length === 0) {
    return 'completed'
  }

  const result = await tools.execute(response.toolCalls)
  return result.concluded ? 'completed' : 'continue'
}
```

源码里的关键分支是提取 `tool-call` 内容块；没有工具调用时，返回 `{ kind: 'completed' }`。有工具调用时，通过 `this.loopCtx.tools.execute(...)` 执行。

通常工具执行完成后继续下一步，让模型读取结果；某些工具结果也可以明确结束处理。流式响应的收集与记录可以进一步阅读 [assistant-stream.ts](../../packages/core/agent-loop/src/assistant-stream.ts)。

### 4. turn()：决定是否继续

`turn()` 管理以下事情：

- 写入 turn、step 的开始和结束事件。
- 反复执行步骤。
- 检查新到达的输入。
- 处理结束前扩展点。
- 处理取消、错误以及未完成的工具结果记录。

生产环境里的循环远比“while → 调模型 → 执行工具”复杂：需要确保中途出错时，日志和工具状态仍能被解释。完整运行约定见 [agent-loop README](../../packages/core/agent-loop/README.md)。

-----

## 七、完整例子

任务：“读取 README.md，告诉我怎么启动项目。”假设 Native 模式下，模型只需要一次文件读取：

| 时刻 | 谁负责 | 发生什么 |
|---|---|---|
| 1 | Agent / Inbox | 接收用户输入，进入待处理队列 |
| 2 | `agent-loop` | 启动 Turn 1，准备 Step 1 |
| 3 | `tools` | 确认 `read` 当前可用 |
| 4 | `system-prompt` | 组装提示词和工具说明 |
| 5 | `session` + loop | 记录输入，推导请求中的对话历史 |
| 6 | `ctx.llm` | 模型返回读取 README 的工具调用 |
| 7 | Native 执行器 | 经工具流水线调用文件工具 |
| 8 | 工具执行机制 | 将文件内容作为工具结果记录 |
| 9 | `agent-loop` | 开始 Step 2，再次调用模型 |
| 10 | 模型 | 根据 README 输出启动方法 |
| 11 | `agent-loop` | 无后续工具调用或其他继续条件，结束本轮 |

这里最值得理解的是第 8、9 步：**工具结果要进入会话，成为下一次模型请求的输入，模型才能据此继续决策。**

阅读核对：在 `step()` 中找到 `buildRequest()`、流式模型调用、`assistant/message` 记录和 `tools.execute()`；再回到 `turn()`，确认工具执行后如何进入下一步。该示例描述源码中的调用路径，不表示每次模型都会选择相同工具或相同步数。

-----

## 八、建议阅读顺序

按下面顺序，每次只回答一个问题：

1. **`tool-executor-native/src/index.ts`**：文件短，先看懂“描述工具”和“执行工具”两个职责。
2. **`agent/src/runtime-types.ts`**：看调用者能对 Agent 做什么，先建立接口概念。
3. **`session/src/types.ts`、`surface.ts`**：理解事件日志与当前对话的区别。
4. **`tools/src/index.ts`**：只追 `register → resolve → execute`，暂时跳过其他细节。
5. **`system-prompt/src/index.ts`**：从 `assemble()` 看最终请求信息怎么组织。
6. **`agent-loop/src/agent.ts`**：先读 `step()` 的成功路径，再往上读 `turn()`、`preStep()`、输入队列。
7. **回头看失败与生命周期**：包括取消、重试、流式响应中断、工具恢复、插件卸载。
8. **最后读 `scope` 和 PTC 的完整实现**：此时你已知道它们要解决哪些实际问题。

阅读时始终带着四个问题：这段代码接收什么、改变什么状态、写入什么事件、下一步交给谁？这比先记住所有类名和事件名，更容易理解整个 core。

## Dev Note

本文件是个人源码阅读笔记。代码示例为结构示意或伪代码；模块的完整配置、接口和运行约定以链接的源码、包 README 和子系统文档为准。
