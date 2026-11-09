**1. 完整历史和模型当前看到的历史分开管理**

Session 有两个层次：

- **事件日志（log）**：保存用户消息、模型回复、工具结果、请求配置、压缩记录等事实，只追加。
- **当前消息表层（surface）**：记录哪些消息节点仍进入模型上下文，通过 `surfaceOp` 的 `append`、`replace` 更新。

`session.deriveMessages()` 从 surface 派生本次模型消息。因此，压缩替换的是**模型可见历史**，原始事件仍留在日志中，可以审计和重放。

这个派生过程有增量缓存：普通追加只处理新增节点；发生替换或消息投影变化时重建。返回的消息对象被冻结，防止调用方意外修改历史。见 deriveMessages 实现 (packages/core/session/src/index.ts:982)。

**2. 每次请求都重新组装并记录上下文**

主要流程是：

```
领取待处理输入
  → 组装系统提示词、工具定义、运行时上下文
  → pre-step 插件处理输入、检查压缩
  → 准备模型调用
  → 将系统提示词和接受的用户输入写入 Session
  → 记录请求配置，派生并冻结消息历史
  → 调用模型
  → 将回复、工具调用及结果写回 Session

```

核心约束是 **“模型可见的信息必须有日志依据”**，恢复会话时才能重建相同输入。

上下文来源也分开处理：

- 系统提示词与工具 schema 由 `system-prompt` 服务组装。
- [`AGENTS.md`](http://AGENTS.md)、[`CLAUDE.md`](http://CLAUDE.md) 等工作区指令作为带来源的 `user/message` 注入。
- 文件指令按文件活动刷新，内容摘要未变化时不重复注入。
- 时间、其他 Session 引用等由独立插件提供；并非全部默认启用。

见 架构中的请求流程 (docs/[architecture.md](http://architecture.md)) 和 工作区指令实现说明 (packages/context/agent-instructions/[README.md](http://README.md))。

**3. 用统一 token 计量服务判断压力**

`ctx.tokenMeter.measure(session)` 根据当前历史和请求配置估算上下文压力。

它不是始终调用精确 tokenizer：

- 文本采用启发式估算，基础约为每 4 个字符一个 token，并计入消息开销。
- 图片优先采用模型适配器声明的计价方式。
- 满足请求配置一致等条件时，使用提供方实际返回的 usage 作为基准，再计算历史变化量。

因此，UI 占用率、估算值和实际账单 token 不能直接等同，中文与 JSON 的启发式误差尤其需要注意。见 Token meter (packages/llm/token-meter/[README.md](http://README.md))。

**4. 接近上限时，先裁剪，再摘要**

默认启用的 `compaction-basic` 在 `agent/pre-step` 检查压力。默认触发阈值为：

```
threshold = floor(min(W × 0.8, W − O − B))

W = 模型上下文窗口
O = 请求预留的最大输出 token
B = 额外余量，默认 65,536 token

```

这些参数支持按 provider/model 配置，小窗口模型需要降低余量。

触发后：

1. 如果挂载了工具结果裁剪插件，先对超长文本保留头尾、裁掉中间，不调用模型。
2. 如果仍超阈值，选取最早的一段完整历史，保证工具调用与结果的配对不被拆散。
3. 发起额外模型请求生成摘要。
4. 用一条包含 `<compacted-summary>` 的用户消息替换旧历史，保留近期消息原文。

默认近期保留预算是 `16% × (W − O)`。例如：

```
压缩前：系统提示词 → 旧对话与工具结果 → 最近对话
压缩后：系统提示词 → 旧历史摘要       → 最近对话

```

摘要请求尽可能复用原来的系统提示词、工具定义和历史前缀，以利用提供方的前缀缓存。见 压缩策略与实现 (packages/compaction/compaction-basic/[README.md](http://README.md))。

**5. 压缩本身也可恢复、可重放**

一次压缩记录为：

```
compaction/start
  → compaction/summary
  → user/message，携带 surface replace
compaction/end

```

标记对同时承担压缩互斥作用；提交摘要前会重新验证选区，且摘要必须确实缩小原内容。

此外还支持：

- **溢出恢复**：提供方明确返回 `CONTEXT_WINDOW_EXCEEDED` 后尝试压缩；确认发生了持久化缩减才允许重试，默认最多重试一次。
- **手动** `/compact`：无需等到自动阈值。
- **切换模型检查**：按照目标模型的容量检查旧历史，必要时先压缩再切换。

这套机制的限制也很明确：摘要会损失细节，压缩不会缩减工具定义和固定请求开销，也无法任意拆开一个不可分割的大消息单元。**完整日志仍在，不代表完整日志仍被发送给模型。**