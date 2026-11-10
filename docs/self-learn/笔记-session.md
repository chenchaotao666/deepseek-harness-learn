# Session 学习笔记：上下文、持久化与事件投影

## 概览

本笔记面向了解基本 JSON 和数据库概念的读者，说明一次提问如何成为 Session 事件、模型上下文如何派生，以及日志、搜索索引和投影缓存分别保存什么。示例数据用于解释字段，不是实际用户记录或可直接导入的完整 Session；实现细节以文末链接的代码和包文档为准。

## 目录

- [1. 完整历史和模型当前看到的历史分开管理](#1-完整历史和模型当前看到的历史分开管理)
- [2. 每次请求都重新组装并记录上下文](#2-每次请求都重新组装并记录上下文)
- [3. 用统一 token 计量服务判断压力](#3-用统一-token-计量服务判断压力)
- [4. 接近上限时先裁剪再摘要](#4-接近上限时先裁剪再摘要)
- [5. 压缩本身也可恢复可重放](#5-压缩本身也可恢复可重放)
- [6. JSONL 与 SQLite 的分工](#6-jsonl-与-sqlite-的分工)
- [7. 事件投影与缓存](#7-事件投影与缓存)
- [8. JSON 与 JSONL 的区别](#8-json-与-jsonl-的区别)
- [9. 日志保存路径与保留规则](#9-日志保存路径与保留规则)
- [10. 一次提问的持久化与恢复](#10-一次提问的持久化与恢复)
- [11. SQLite 表结构与示例数据](#11-sqlite-表结构与示例数据)
- [12. 阅读资料](#12-阅读资料)

## 1. 完整历史和模型当前看到的历史分开管理

Session 有两个层次：

- **事件日志（log）**：保存用户消息、模型回复、工具结果、请求配置、压缩记录等事实，只追加。
- **当前消息表层（surface）**：记录哪些消息节点仍进入模型上下文，通过 `surfaceOp` 的 `append`、`replace` 更新。

`session.deriveMessages()` 从 surface 派生本次模型消息。因此，压缩替换的是**模型可见历史**，原始事件仍留在日志中，可以审计和重放。

这个派生过程有增量缓存：普通追加只处理新增节点；发生替换或消息投影变化时重建。返回的消息对象被冻结，防止调用方意外修改历史。见 [deriveMessages 实现](../../packages/core/session/src/index.ts)。

## 2. 每次请求都重新组装并记录上下文

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
- `AGENTS.md`、`CLAUDE.md` 等工作区指令作为带来源的 `user/message` 注入。
- 文件指令按文件活动刷新，内容摘要未变化时不重复注入。
- 时间、其他 Session 引用等由独立插件提供；并非全部默认启用。

见[架构中的请求流程](../architecture.md)和[工作区指令实现说明](../../packages/context/agent-instructions/README.md)。

## 3. 用统一 token 计量服务判断压力

`ctx.tokenMeter.measure(session)` 根据当前历史和请求配置估算上下文压力。

它不是始终调用精确 tokenizer：

- 文本采用启发式估算，基础约为每 4 个字符一个 token，并计入消息开销。
- 图片优先采用模型适配器声明的计价方式。
- 满足请求配置一致等条件时，使用提供方实际返回的 usage 作为基准，再计算历史变化量。

因此，UI 占用率、估算值和实际账单 token 不能直接等同，中文与 JSON 的启发式误差尤其需要注意。见 [Token meter](../../packages/llm/token-meter/README.md)。

## 4. 接近上限时先裁剪再摘要

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

摘要请求尽可能复用原来的系统提示词、工具定义和历史前缀，以利用提供方的前缀缓存。见[压缩策略与实现](../../packages/compaction/compaction-basic/README.md)。

## 5. 压缩本身也可恢复可重放

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

## 6. JSONL 与 SQLite 的分工

Session 原始事件以 JSONL 形式持久化，默认使用 Zstandard 压缩。SQLite 是从日志建立的可重建全文搜索索引，不负责完整会话恢复。默认 base 组合启用 JSONL 后端，但搜索配置为 `path: ':memory:'`、`openAt: never`，因此默认不会打开 SQLite。

内容如：

> 帮我读取 `README.md` 并总结。

**JSONL 中会记录完整过程**，下面只展示事件概要：

```
turn/start
step/start
system/message       系统提示词
user/message         “帮我读取 README.md 并总结”
request/header       使用哪个模型、有哪些工具
assistant/message    模型生成读取文件的工具调用
tool/call            工具名、调用 ID、原始参数
tool/result          读取到的内容
step/end
step/start
assistant/message    总结结果，以及 stream、usage 等信息
step/end
turn/end

```

**SQLite 则提取其中适合搜索的部分**：

| session\_id | seq | type              | text                                   |
| ----------- | ---: | ----------------- | -------------------------------------- |
| s001        | 3   | user/message      | 帮我读取 `README.md` 并总结 |
| s001        | 6   | tool/call         | 工具名及读取参数                               |
| s001        | 7   | tool/result       | `README.md` 的文本内容    |
| s001        | 10  | assistant/message | 总结结果                                   |

所以两种操作走不同路径：

```
“继续上次对话”
  → 读取 JSONL → 重建 Session → 继续执行

“查找提到 README.md 的历史会话”
  → 查询 SQLite FTS5 → 返回 Session ID、事件位置、文本片段
  → 打开完整会话时，再读取 JSONL

```

这里的分工是 **Harness 的实现选择**。SQLite 本身完全可以保存完整事件日志，只是当前 Harness 将它用于可重建的搜索索引，原始 Session 持久化由 JSONL 后端负责。

## 7. 事件投影与缓存

**事件投影，就是按顺序读取事件历史，计算出某种“当前状态”或“查询结果”。**

在 DeepSeek Harness 中，事件日志记录“发生过什么”；投影回答“根据这些记录，现在是什么状态”。

例如，会话中依次保存了这些事件：

```
seq 0：turn/start          第 1 轮开始
seq 1：user/message        用户问：“读取 README 并总结”
seq 2：assistant/message   模型要求调用读取工具
seq 3：tool/result         返回 README 内容
seq 4：assistant/message   模型给出总结，usage 为输入 800、输出 100
seq 5：turn/end            第 1 轮完成

```

同一份日志可以产生不同投影：

| 投影用途     | 读取哪些信息                  | 得到的结果                    |
| -------- | ----------------------- | ------------------------ |
| 轮次状态     | `turn/start`、`turn/end` | 第 1 轮已完成                 |
| Token 用量 | 模型回复中的 `usage` 等记录      | 输入 800、输出 100            |
| 模型消息历史   | 当前有效的消息节点               | 用户问题、工具调用、工具结果、模型总结      |
| 全文搜索文档   | 事件中的可搜索文本               | 文本与 Session ID、事件序号的对应关系 |

这些是不同的派生视图；具体实现可以是注册的状态投影、消息派生函数或搜索文档提取器。

**它是怎么计算的？**

最常见的方式是逐条更新状态，也叫 *fold*：

```
初始状态
   ↓ 处理事件 0
新状态
   ↓ 处理事件 1
新状态
   ↓ ……
最终状态

```

例如，轮次状态可以这样理解。下面是示意代码，不是仓库原样实现：

```
function applyEvent(state, event) {
  switch (event.type) {
    case "turn/start":
      return { ...state, turn: event.data.turn, running: true };

    case "turn/end":
      return { ...state, running: false, reason: event.data.reason };

    default:
      return state;
  }
}

```

假设投影已经处理到 `seq = 5`，新事件到来时通常只需继续处理 `seq = 6、7……`，不用每次从头计算。

**为什么还要保存“投影缓存”？**

因为历史很长时，每次启动都从第一条事件算起会慢。因此可以保存：

```
{
  "ver": 1,
  "seq": 5,
  "val": {
    "turn": 1,
    "running": false,
    "reason": {"kind": "completed"}
  }
}

```

这也是示意数据，含义是：

- `ver`：这份投影状态的数据格式版本。
- `seq`：已经处理到哪条事件。
- `val`：处理到这里得到的状态。

恢复时，先加载兼容的缓存，再处理后面的事件。缓存失效或丢失，则可以从原始日志重新计算。

**投影和上下文压缩有什么关系？**

假设日志里原来有：

```
用户问题 A → 回答 A → 用户问题 B → 回答 B

```

压缩后，日志追加一条摘要消息及替换记录。此时：

```
原始日志：仍保留 A、B 的完整事件，以及后来的压缩记录
模型消息视图：旧历史摘要 → 后续消息

```

因此，投影不仅是统计，也可以解释日志中的替换操作，决定哪些内容目前有效。**原始事件是依据，投影是针对某个用途从这些依据计算出的结果。**

## 8. JSON 与 JSONL 的区别

JSON 文件包含一个完整 JSON 值，可以用数组保存多条记录；JSONL 文件中每个物理行都是一个独立 JSON 值，记录之间没有逗号，外层也没有数组括号。

JSON 示例：

```json
[
  {"type": "user/message", "text": "你好"},
  {"type": "assistant/message", "text": "你好，有什么可以帮你？"}
]
```

JSONL 示例：

```jsonl
{"type":"user/message","text":"你好"}
{"type":"assistant/message","text":"你好，有什么可以帮你？"}
```

这些记录只展示格式差异，省略了 Session 事件的公共字段。消息内容中的换行写成转义字符 `\n`，不能把一条 JSONL 记录排成多个物理行。

| 对比 | JSON 文件 | JSONL 文件 |
|---|---|---|
| 整体结构 | 一个完整 JSON 值 | 每行一个独立 JSON 值 |
| 排版 | 可以缩进并跨行 | 每条记录占一行 |
| 新增数组记录 | 需要维护数组括号和分隔逗号 | 可在末尾追加一行 |
| 读取 | 通常整体解析，也可流式解析 | 便于逐行解析 |
| 常见用途 | 配置、API 响应 | 事件日志、批量数据 |

Session 持续产生事件，因此适合追加式 JSONL。`.jsonl.zstd` 是这些 JSONL 记录的 Zstandard 压缩文件，不是另一套事件数据格式。

## 9. 日志保存路径与保留规则

默认 base 组合将日志根目录设置为 `dshHomePath('sessions')`。Harness home 未另行配置时读取 `DSH_HOME`，未设置时使用用户目录下的 `.dsh`；持久化插件也可以通过 `root` 使用其他目录。实际位置取决于启动配置，不能只根据当前项目目录判断。

```text
$DSH_HOME/
  sessions/
    --规范化后的项目路径--/
      <编码后的 Session ID>/
        session.v4.jsonl.zstd
```

以 Windows 用户目录 `C:\Users\Administrator`、项目目录 `E:\projects\demo` 为例，默认路径为：

```text
C:\Users\Administrator\.dsh\sessions\
  --E-projects-demo--\
    <Session ID>\
      session.v4.jsonl.zstd
```

这里的 `v4` 对应本示例使用的 Session 格式，实际当前版本由 [SESSION_FORMAT_VERSION](../../packages/core/session/src/types.ts) 定义。没有工作目录的 Session 归入 `_no-cwd`；关闭压缩时文件后缀为 `.jsonl`。Session header 和事件编码规则见 [JSONL 后端](../../packages/session/session-persistence-jsonl/README.md)。

| 情况 | 对原始日志的影响 |
|---|---|
| 一轮问答结束、关闭程序、释放内存 Session | 保留已经持久化的日志 |
| 上下文压缩 | 追加摘要与替换记录，保留原始事件 |
| Session 格式升级 | 发布新版本文件，保留已提交的历史版本 |
| 超过一定天数或磁盘容量 | 当前没有自动过期或容量淘汰机制 |
| 用户或外部清理工具删除文件 | 原始日志可能丢失，搜索索引不能完整还原它 |

持久化接口没有删除 API，日志会在根目录中持续积累，直到被外部删除。上下文压缩减少的是模型请求中的历史，不会回收已有日志占用的磁盘空间。后端清理临时文件或修复崩溃造成的未完成尾部，不等于按保留期限删除已提交事件。

## 10. 一次提问的持久化与恢复

以“搜索 DeepSeek Harness 的资料，然后给我一句介绍”为例：一个 Session 可以包含多轮提问；处理这次提问是一个 Turn；第一次模型调用要求搜索，第二次根据结果回答，因此本轮有两个 Step。

### 输入如何进入日志

用户问题先通过 `agent/inbox/spliced` 入队。Loop 写入 `turn/start`，再领取输入；只有输入通过请求准备并写为 `user/message` 后，才成为模型历史。排队记录与模型已看到的消息是两件事。

下面列出主要事件顺序，省略权限、标题等插件记录，不为省略后的列表分配虚假的连续事件编号：

```text
agent/inbox/spliced   用户问题入队
turn/start           开始本轮
agent/inbox/spliced   领取问题
step/start           第 1 步
system/message       系统提示词
user/message         用户问题及适用的上下文消息
request/header       模型配置和工具定义
request/context      路由容量等信息
                     flush 成功后开始模型请求
assistant/message    模型要求搜索，内嵌本次 stream 和可用的 usage
tool/call            搜索调用及原始参数
                     flush 成功后执行顶层工具
tool/result          搜索结果
step/end
step/start           第 2 步，之前的回复和结果先经过持久化屏障
assistant/message    根据搜索结果给出最终回答
step/end
turn/end             本轮完成
```

`request/header` 和 `request/context` 按变化及恢复等规则记录，不一定在每个 Step 重复。完整的 V4 结构可参考 [搜索会话快照](../../snapshots/web/web-search-round/session.v4.jsonl)；快照使用占位符并归一化部分字段，不是未经处理的生产日志。

### 文件里的记录

文件第一行是 Session header，不参与事件 `seq` 编号。下面的 header 和事件采用 V4 教学数据；事件片段前后的记录未展示，不能单独作为可恢复的完整日志。

```json
{
  "type": "session",
  "version": 4,
  "id": "s-demo-001",
  "createdAt": 1800000000000,
  "cwd": "E:\\projects\\demo",
  "isSeeded": false,
  "delegationDepth": 0,
  "agentPreset": "standard"
}
```

一条已经接受的用户消息展开后如下。实际 JSONL 会把它写成一行：

```json
{
  "type": "user/message",
  "seq": 5,
  "time": 1800000000180,
  "data": {
    "id": "m-user-001",
    "role": "user",
    "source": {"kind": "user"},
    "content": [
      {"type": "text", "text": "搜索 DeepSeek Harness 的资料，然后给我一句介绍。"}
    ]
  },
  "surfaceOp": "append"
}
```

`seq` 是 Session 内连续的事件编号，`time` 是 Unix 毫秒时间，`data` 保存事件专属内容。`surfaceOp` 控制消息加入或替换当前模型历史；结构事件如 `turn/start` 没有该字段。`source` 区分直接用户输入、运行时上下文、模型和工具等来源。

工具调用和结果通过 `callId`、`toolCallId` 关联。调用参数保存为模型产生的原始 JSON 字符串：

```jsonl
{"type":"tool/call","seq":10,"time":1800000001000,"data":{"turn":1,"step":1,"callId":"call-search-001","name":"web_search","arguments":"{\"queries\":[\"DeepSeek Harness\"]}"}}
{"type":"tool/result","seq":11,"time":1800000001500,"data":{"turn":1,"step":1,"message":{"id":"m-tool-001","role":"tool","source":{"kind":"tool","callId":"call-search-001"},"toolCallId":"call-search-001","content":[{"type":"text","text":"搜索结果：DeepSeek Harness 是基于 Cordis 的插件化智能体框架。"}],"isError":false}},"surfaceOp":"append"}
```

`assistant/message` 除组装后的消息外，还保存本次模型输出的紧凑 `stream`，以及提供方返回时才有的 `usage`；`tool/result` 可以附带工具私有的 `meta`，供界面重建结果卡片。不是所有运行时通知都对应独立日志行：实时流片段先用于界面更新，结算后进入 assistant 事件。

### 写入时机与崩溃恢复

JSONL 后端延迟创建实际文件：首次 append 写入 header 和事件，显式 flush 也能物化仅含 header 的文件。写入按批次追加并 `fsync`。默认 checkpoint 插件在模型调用前、顶层工具执行前和下一步请求派生前等待持久化完成；写入失败时不进入依赖它的模型调用或工具执行。

再次打开会话时，后端选择最高版本的规范日志文件，校验和解码事件，必要时做受支持的历史格式迁移。Session 从日志重建消息视图和队列状态，兼容的投影缓存可以加速状态折叠；SQLite 搜索结果不参与完整会话恢复。

| 崩溃时已经持久化的证据 | 恢复处理 |
|---|---|
| assistant 请求调用工具，但没有 `tool/call` | 补 `TOOL_NOT_STARTED` |
| 有 `tool/call`，没有 `tool/result` | 补 `TOOL_OUTCOME_UNKNOWN` |
| 已有工具结果 | 重放已保存结果 |
| 模型仍在流式输出，尚未结算 assistant 事件 | 硬崩溃可能丢失尚未持久化的流片段 |

未知结果不代表工具没有执行成功。例如外部写入可能已经完成，只是结果尚未落盘，因此恢复后需要核实副作用，不能盲目重复执行。读操作可在内存中补齐中断记录；实际续写通过写句柄持久化恢复记录。

## 11. SQLite 表结构与示例数据

全文检索启用后，搜索服务从事件中提取文本，建立可重建的索引。以下是核心持久表的学习摘录，准确 DDL 和临时表定义由 [schema.ts](../../packages/session-query/session-query-sqlite/src/schema.ts) 维护；本笔记不提供手动建库或修改应用数据库的操作步骤。

```sql
CREATE TABLE search_state (
  singleton INTEGER PRIMARY KEY CHECK (singleton = 1),
  global_generation INTEGER NOT NULL
) STRICT;

CREATE TABLE persisted_sessions (
  id TEXT PRIMARY KEY,
  version INTEGER NOT NULL,
  created_at INTEGER NOT NULL,
  cwd TEXT,
  parent_session TEXT,
  seed_length INTEGER,
  delegation_depth INTEGER,
  agent_preset TEXT,
  revision TEXT NOT NULL,
  generation INTEGER NOT NULL
) STRICT;

CREATE VIRTUAL TABLE persisted_docs USING fts5(
  text,
  session_id UNINDEXED,
  seq UNINDEXED,
  type UNINDEXED,
  time UNINDEXED,
  surface UNINDEXED,
  codepoint_length UNINDEXED,
  tokenize = 'unicode61'
);
```

`persisted_sessions` 保存会话元数据和源修订标识；`persisted_docs` 保存可搜索文本及其事件位置。FTS5 的 `UNINDEXED` 表示字段不参与全文分词，不表示不存储。`search_state` 的代次用于识别跨会话分页游标是否过期。Session 格式 `version` 和数据库自身的 `PRAGMA user_version` 是不同的版本号。

假设 `s-demo-001` 已持久化且完成索引同步，元数据行的部分字段如下。revision 由后端生成，不能使用下面的文字占位符代替实际值：

| id | version | cwd | parent_session | agent_preset | revision |
|---|---|---|---|---|---|
| s-demo-001 | 4 | E:\projects\demo | NULL | standard | 源日志修订标识 |

对应的搜索文档可以包含下列行，事件编号引用上一节的示例片段：

| session_id | seq | type | surface | text |
|---|---|---|---|---|
| s-demo-001 | 5 | user/message | current | 搜索 DeepSeek Harness 的资料，然后给我一句介绍。 |
| s-demo-001 | 10 | tool/call | log-only | web_search、换行以及原始参数 JSON |
| s-demo-001 | 11 | tool/result | current | 搜索结果：DeepSeek Harness 是基于 Cordis 的插件化智能体框架。 |

这不是完整索引行集。assistant 工具调用和最终回答也可能产生文档；结构事件、请求配置、内嵌 stream、reasoning 和当前提取器未支持的类型不产生搜索文本。`surface` 区分 `current`、`shadowed`、`log-only`，因此被压缩遮蔽的旧消息仍可以搜索。

运行中的 Session 使用连接级临时表 `live_sessions` 和 `live_docs`，同一 Session 的实时视图优先于持久索引。`live_sessions` 使用 `fingerprint`、`persisted` 等字段描述实时状态；临时表不会跨重启保存。每次搜索前，服务比较持久日志 revision，在事务中同步新增或变化的文本，再执行查询。

投影缓存则位于 `$DSH_HOME/storages/session_projcache/sessions/<id>.json`，不在这些 SQLite 表中。其逻辑记录包含 `identity` 和 `rows`：前者标识日志格式、创建时间、工作目录和继承关系，后者按投影名称保存 `{ver, seq, val}`。写缓存前先 flush 日志，所以缓存可能落后，但不能领先于已持久化历史；具体状态字段由各投影插件定义。

## 12. 阅读资料

- [架构中的请求流程与 Session 日志](../architecture.md)：模型请求、消息与事件的关系。
- [Session 定义](../../packages/core/session/README.md)：追加事件和派生消息。
- [JSONL 持久化后端](../../packages/session/session-persistence-jsonl/README.md)：文件布局、写入和恢复。
- [持久化屏障](../../packages/session/session-checkpoint-policy/README.md)：模型与工具执行前的 flush。
- [SQLite 搜索服务](../../packages/session-query/session-query-sqlite/README.md)：索引同步、实时表与搜索规则。
- [事件投影服务](../../packages/session/session-projection/README.md)及[投影缓存](../../packages/session/session-projection-cache/README.md)：状态折叠与缓存有效性。

## Dev Note

本文件是个人学习参考，示例数据按源码结构编写；没有执行真实模型请求、故障恢复或持久化集成测试。配置、版本和恢复保证由上面的实现文档及其测试维护。
