# 批量工具调用学习笔记

## 概览

DSH 可以接收模型一次返回的多个工具调用，对允许并发的调用使用有上限的滚动执行池，对独占调用设置执行屏障，并按模型调用顺序处理结果。核心实现是 `packages/core/tools/src/batch.ts`。

本笔记围绕批量工具调用（batch tool execution）解释源码，不讨论文件补丁工具。调用链和数据示例用于源码阅读，不是独立可运行应用。具体接口以链接的源码为准。

## 目录

- [一、实现位置与调用链](#一实现位置与调用链)
- [二、输入输出与整体结构](#二输入输出与整体结构)
- [三、输入规划与分组](#三输入规划与分组)
- [四、并发安全与独占屏障](#四并发安全与独占屏障)
- [五、runGroup 的状态变量](#五rungroup-的状态变量)
- [六、启动调用与滚动执行池](#六启动调用与滚动执行池)
- [七、乱序完成与顺序提交](#七乱序完成与顺序提交)
- [八、结果收集与会话记录](#八结果收集与会话记录)
- [九、取消与内部异常](#九取消与内部异常)
- [十、完整示例与关键限制](#十完整示例与关键限制)
- [十一、源码阅读顺序](#十一源码阅读顺序)

-----

## 一、实现位置与调用链

Agent loop 收集模型响应中的工具调用，再交给当前执行器。原生执行器把整个批次交给共享调度器，单次调用的准备、派发与完成处理由执行流水线负责。

| 位置 | 职责 |
|---|---|
| [agent-loop/src/agent.ts](../../packages/core/agent-loop/src/agent.ts) | 提取全部 `tool-call`，调用 `ctx.tools.execute()` |
| [tools/src/index.ts](../../packages/core/tools/src/index.ts) | 选择 agent 使用的执行器并分发批次 |
| [tool-executor-native/src/index.ts](../../packages/core/tool-executor-native/src/index.ts) | 原生执行入口，调用 `executeToolBatch()` |
| [tools/src/batch.ts](../../packages/core/tools/src/batch.ts) | 批量调度、顺序提交、取消与异常收尾 |
| [tools/src/pipeline.ts](../../packages/core/tools/src/pipeline.ts) | 单次调用的 `prepare()`、`dispatch()`、`complete()` |
| [tools/src/executor.ts](../../packages/core/tools/src/executor.ts) | 批次输入输出、执行器接口和批次异常 |

```text
模型返回多个 tool-call
  → Agent 提取 calls[]
  → ctx.tools.execute()
  → NativeToolExecutor.execute()
  → executeToolBatch()
  → runGroup()
  → pipeline.prepare() / dispatch() / complete()
```

PTC 程序内通过 `Promise.all()` 提交的子调用另有调度逻辑，见 [tool-executor-ptc/src/ptc.ts](../../packages/core/tool-executor-ptc/src/ptc.ts)。根批次并发上限和程序内部子调用上限是分开的；本页主要解释 `batch.ts`。

## 二、输入输出与整体结构

`ToolBatchInput` 的输入既包括调用本身，也包括 agent 作用域、取消信号和可选日志归属。

| 字段 | 含义 |
|---|---|
| `calls` | 模型按顺序返回的原始调用列表 |
| `signal` | 批次取消信号 |
| `maxParallelToolCalls` | 可选正整数，覆盖执行器的并发上限 |
| `agent` | 提供调用作用域和执行上下文 |
| `toolSet` | 当前模型步骤允许调用的工具集合 |
| `log` | 可选的 turn、step，指定根调用事件归属；提供时要求同时提供 agent |

`agent` 和 `log` 不等价：前者提供作用域，后者决定本调度器是否追加根调用的 `tool/call`、`tool/result` 事件。不提供 `log` 时，这些根事件不记录，但工具自身仍可能使用 agent 的 Session。

返回的 `ToolBatchResult` 包含按模型调用顺序排列的 `results`、工具产生的 `additionalContexts`，以及表示是否结束当前 turn 的 `concluded`。

文件中的四个主要部分如下：

| 部分 | 作用 |
|---|---|
| `executeToolBatch()` | 规划输入并依次运行并行组或独占调用 |
| `parseArguments()` | 解析模型参数字符串 |
| `runGroup()` | 维护执行池与提交游标，处理取消和异常 |
| `BatchSettlement` | 收集结果和上下文，按需记录 Session 事件 |

## 三、输入规划与分组

`executeToolBatch()` 先验证显式传入的并发上限必须是正整数。此检查位于外层 `try` 之前，因此非法上限直接抛出 `RangeError`，不会包装为 `ToolBatchExecutionError`。上限为 1 时，可并发调用也串行执行；没有覆盖值时，调度器读取执行器提供的上限。

每个调用转换为 `PlannedCall`，保留两部分数据：`block` 是模型原始调用，用于日志和结果关联；`exec` 是解析后的执行输入，包含调用 ID、名称、参数、agent、toolSet 和 signal。每个调用创建独立的执行输入对象，因为执行包装器可能替换它的信号。

参数解析遵循以下规则：

```text
空参数字符串 → {}
合法 JSON    → JSON.parse() 的结果
非法 JSON    → 保留原始字符串，交给后续流水线诊断
```

因此不会仅因这里解析一个调用失败，就直接终止整批调用。

外层游标 `next` 指向尚未消费的调用。当前调用独占时，只将它自己交给 `runGroup()`；当前调用允许并行时，将剩余列表交给 `runGroup()`，由内部调度在启动后续调用前检查模式并决定停止位置。

```text
next 指向首个未消费调用
  → 查询 executionMode
  → parallel：传入剩余列表
  → exclusive：只传入当前项
  → runGroup() 返回 consumed
  → next += consumed
```

这不是一次性固定分组：未启动调用的分类可以随着工具注册状态变化而变化。

## 四、并发安全与独占屏障

`pipeline.executionMode()` 根据可见工具的 `isConcurrencySafe(arguments)` 判断执行方式。仅当它返回严格的 `true` 时允许并行；未声明、工具不可见、返回其他值或判断抛错时，均按独占处理。

同一工具可以根据参数区分读取和修改操作。调度器不自动分析文件路径冲突或业务依赖，并发安全声明由工具定义负责。

独占调用构成屏障：前面的并行工作结束后才能执行它，后面的调用也不能越过它提前启动。

```text
输入：读取 A → 读取 B → 修改 C → 读取 D

读取 A ─┐
读取 B ─┴→ 等待结束 → 修改 C → 读取 D
```

屏障作用于这个批次的调度，不是跨 agent、跨进程的全局资源锁。多个批次共享外部资源时，仍需工具或服务自身提供必要的协调。

## 五、runGroup 的状态变量

`runGroup()` 将启动进度、主体执行进度和结果提交进度分开维护。

| 变量 | 含义 |
|---|---|
| `slots[index]` | 已产生、等待有序完成处理的结果 |
| `callSeqs[index]` | 该调用的 `tool/call` 事件序号 |
| `nextToStart` | 下一个准备启动的下标 |
| `started` | 已进入启动流程的调用数量 |
| `committed` | 连续提交的结果数量，也是下一个提交位置 |
| `inFlight` | 已派发、尚未从调度集合移除的 Promise |
| `aborted` | 调度器是否已观察到取消 |
| `schedulerFailure` | 首个内部调度异常 |
| `concluded` | 已提交结果中是否出现结束 turn 的标记 |

`started` 不表示工具主体一定执行过。一个调用可以已经记录 `tool/call`、进入准备流程，然后被策略拒绝；它仍算 started，但只产生错误结果，没有执行主体。

`inFlight` 中也可能短暂保留已经完成、尚未被主循环移除的 Promise，因此其大小是调度集合大小，不是精确的 CPU 或线程运行数量。

## 六、启动调用与滚动执行池

`startCall()` 先通过 `settlement.start()` 记录调用，再增加 started，随后等待 `pipeline.prepare()`。

准备阶段返回三种状态：

| 状态 | 后续处理 |
|---|---|
| `dispatch` | 派发工具主体，把 Promise 加入 inFlight |
| `post-result` | 已有结果，写入 slots，仍需执行后处理 |
| `final-result` | 已有结果，写入 slots，跳过执行后策略路径 |

派发时不会立即等待主体结束，而是注册完成回调：正常返回时写入对应 slot；异常拒绝时记录首个 schedulerFailure。两条回调都返回调用下标，供主循环识别完成项。

```text
prepare(A) → 启动 dispatch(A)
prepare(B) → 启动 dispatch(B)

dispatch(A) 与 dispatch(B) 可以重叠
```

`fillPool()` 在未观察到取消、有剩余调用且 `inFlight.size` 小于并发上限时继续补充。准备阶段按调用顺序等待，主体派发可以重叠；有现成结果的调用不需要创建主体执行 Promise。

主循环使用 `Promise.race(inFlight.values())` 等待任一调用结束，移除对应项，尝试有序提交，再补充空位。它不必等待最初一批全部结束才启动下一项，因此称为有界滚动执行池。

```text
并发上限 2，A、B、C 均允许并行：

启动 A 和 B
B 先完成 → 移除 B → 可以启动 C
A 与 C 继续运行
```

后续调用在启动前重新分类；遇到独占项就停止补充，不跳过它执行更后面的项。prepare 和 complete 也可能等待，因此调度器不保证始终满负载。

## 七、乱序完成与顺序提交

`commitReady()` 只处理 `slots[committed]`。当前下标没有结果时立即停止，即使后面的结果已经准备好，也不会越过它提交。

```text
模型顺序：A → B → C
完成顺序：B → C → A

A 未完成时：
  slots[0] = undefined
  slots[1] = B 的结果
  slots[2] = C 的结果
  committed = 0

A 完成后：
  complete(A) → commit(A)
  complete(B) → commit(B)
  complete(C) → commit(C)
```

这里同时保证了执行后处理、结果收集和附加上下文的模型顺序。B 的完成可以释放执行池位置，使 C 启动，但 B 的结果提交仍可能等待 A；执行吞吐与结果发布是两个不同进度。

顺序提交不等于副作用串行化。并行调用的主体已经发生重叠，结果排序不能修复两个工具同时修改同一资源造成的冲突。

## 八、结果收集与会话记录

`BatchSettlement` 维护 results 与 contexts 两个数组，并通过 start、commit 和 skip 统一记录处理。

| 方法 | 行为 |
|---|---|
| `start()` | 提供 log 时追加 tool/call，返回事件序号；否则返回 undefined |
| `commit()` | 提供 log 时追加 tool/result，再收集结果和 additionalContexts |
| `skip()` | 为取消后未启动调用补充调用记录和明确的错误结果 |
| `result()` | 返回结果、上下文与 concluded |

`tool/call` 保留原始参数字符串。`tool/result` 通过 `sourceEventSeqs` 关联调用事件，并包含模型可见结果、错误信息和可选 metadata。此处的 Session append 不等于本文件已经完成磁盘刷写，物理持久化由对应 Session 组件负责。

`skip()` 生成 `TOOL_ABORTED_BEFORE_DISPATCH` 错误，明确工具主体没有开始执行。它仍调用 start 和 commit，使模型声明的调用具有对应结果，便于会话回放。

`additionalContexts` 按结果提交顺序收集，返回上层后由 agent loop 安排到后续步骤，而不是主体一产生上下文就立即乱序注入。

## 九、取消与内部异常

普通工具错误、用户取消和调度内部异常是不同路径。

| 情况 | 行为 |
|---|---|
| 普通工具执行失败 | 流水线通常转成 isError 结果，有序提交，其他调用可以继续 |
| 用户取消 | 停止补充新调用，等待已派发工作，提交结果，为未启动项补充取消结果 |
| 内部调度异常 | 停止正常调度，等待已派发工作结束，携带已提交的部分结果抛错 |

### 取消收尾

取消后，调度器不会直接丢弃已启动 Promise。流水线将取消信号传给工具，并等待主体结束；当前组未启动项由 runGroup 调用 skip，剩余组外项由外层处理。

```text
观察到取消
  → 停止启动新调用
  → 等待已派发工作结束
  → 按顺序完成并提交已有结果
  → 对未启动调用补充取消结果
  → 返回批次结果
```

取消是协作式的。如果工具不响应取消且其 Promise 一直不结束，批量调度器本身无法强制终止它。这样的等待避免了“批次已经结束，但旧工具仍在后台修改资源”的情况。

### 内部异常收尾

runGroup 捕获内部异常后保留第一个错误，并使用 `Promise.allSettled()` 等待 inFlight，再抛出错误。外层包装为 `ToolBatchExecutionError`，其中 partial 包含已经提交的结果及上下文。

已执行完成但尚未提交的调用不会被列入已提交结果。上层执行器调用方需要停止依赖工作并修复缺失的日志结果；这个异常路径不能承诺每个输入都已经有完整结果。

调度器不会回滚已经发生的副作用，也不会在内部异常分支自动创建取消信号来终止其他调用。等待结束、记录结果与业务事务回滚是不同机制。

## 十、完整示例与关键限制

假设 A、B、C 允许并行，D 独占，E 允许并行，并发上限为 2：

```text
输入：A、B、C、D、E

1. 外层从 A 开始运行并行组。
2. 启动 A、B。
3. B 先结束，结果等待 A；空位允许 C 启动。
4. 下一个 D 是独占调用，停止补充。
5. 等待 A、C 结束，按 A、B、C 顺序提交。
6. 返回 consumed = 3，外层 next 移到 D。
7. 单独执行并提交 D。
8. 执行并提交 E。
```

本实现有以下需要保留的区别：

- 批量调用不等于全部并行；只有明确允许并发的调用可以重叠。
- 执行顺序、完成顺序与提交顺序不同；结果有序不代表副作用有序。
- 并发上限约束当前批次的主体派发，不能代替系统级资源限额或跨批次锁。
- `concluded` 只累计“结束 turn”的标记，不立即停止当前批次；上层在批次完成后决定结束 turn。
- `aborted` 停止新调用，并为跳过调用补齐结果，与 concluded 不同。
- 普通工具错误可以作为结果返回；内部调度异常通过携带 partial 的异常报告。
- 本页描述原生根调用批次；PTC 内部子调用有单独的调度与并发配置。

## 十一、源码阅读顺序

1. [executor.ts](../../packages/core/tools/src/executor.ts)：理解输入、输出和执行器失败保证。
2. [batch.ts](../../packages/core/tools/src/batch.ts)：先看 executeToolBatch，再看 BatchSettlement，最后追踪 runGroup 的三个内部函数。
3. [pipeline.ts](../../packages/core/tools/src/pipeline.ts)：理解 executionMode、prepare、dispatch、complete 的分工。
4. [tool-executor-native](../../packages/core/tool-executor-native/src/index.ts)：确认原生入口如何连接调度器。
5. [agent-loop/src/agent.ts](../../packages/core/agent-loop/src/agent.ts)：查看工具批次如何进入步骤，以及结果上下文和 concluded 如何被消费。
6. [execution-mode 测试](../../packages/core/tools/tests/execution-mode.spec.ts) 与 [工具流水线测试](../../packages/core/tools/tests/tools.spec.ts)：对照并发分类、取消和错误结果的行为。

阅读时同时追踪 nextToStart、inFlight 和 committed：它们分别表示准备启动的位置、已派发集合和顺序提交的位置。

## 开发备注

本页整理源码行为与学习示例，不包含本次执行测试的结果。维护中的接口与行为以源码及其测试为准。
