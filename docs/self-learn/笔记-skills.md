# packages/skill 学习笔记

## 概览

Skill 是可复用的任务专项指令。DeepSeek Harness（DSH）先向模型提供技能摘要，再按需加载完整指令，并提供附属资源的位置。这份源码阅读笔记整理渐进式加载、数据类型、作用域、热更新和技能目录之间的关系，面向了解基本 TypeScript、希望理解 Skill 实现的读者。代码调用链是阅读示意，不是独立应用或部署教程；具体配置以各包 README 为准。

## 目录

- [一、模块职责](#一模块职责)
- [二、渐进式加载](#二渐进式加载)
- [三、元数据与正文分离](#三元数据与正文分离)
- [四、注册与发现](#四注册与发现)
- [五、作用域与同名技能选择](#五作用域与同名技能选择)
- [六、技能如何进入模型上下文](#六技能如何进入模型上下文)
- [七、缓存失效与热更新](#七缓存失效与热更新)
- [八、技能存放目录](#八技能存放目录)
- [九、技术特点与优势](#九技术特点与优势)
- [十、源码阅读顺序](#十源码阅读顺序)

-----

## 一、模块职责

`packages/skill` 将技能来源、注册表和模型入口分开。注册表不直接承担文件扫描或模型工具调用；提供方负责发现与读取，消费者负责展示与调用策略。

| 子包 | 职责 | 主要入口 |
|---|---|---|
| `skill` | 定义技能类型，注册提供方，处理作用域、优先级和目录缓存 | `ctx.skills` |
| `skill-filesystem` | 扫描本地目录，解析 Markdown，监听文件变化 | `FileSystemSkillProvider` |
| `tool-skill` | 发布摘要目录，加载正文，处理用户显式调用 | `skill`、`agent/pre-step` |
| `skill-office` | 提供 Office 路由、创作、渲染等内置技能 | `dsh-office` provider |
| `tool-workspace-dependencies` | 提供执行技能所需的解释器、库目录和版本信息 | `load_workspace_dependencies` |

```text
本地文件 / 内置 Office 技能 / 运行时注册
                  ↓
          ctx.skills 注册表
                  ↓
     tool-skill 摘要目录与正文加载
                  ↓
              模型上下文
                  ↓
      按需读取资源、调用执行工具
```

运行环境查询与 Skill 加载相互独立：加载指令不等于执行脚本或准备 Python 环境。完整语义见 [Skill 子系统参考](../subsystems/skills.zh.md)。

## 二、渐进式加载

渐进式披露（Progressive Disclosure）是按任务需要逐步提供内容，主要优化模型上下文，而非自动学习或升级技能。

| 层次 | 模型获得的内容 | 时机 |
|---|---|---|
| 摘要目录 | 技能名称、描述 | 模型步骤前发布，变化时更新 |
| 完整指令 | 所选技能的 Markdown 正文及资源位置 | 模型调用 `skill`，或用户显式调用 |
| 附属资源 | 引用文档、示例或脚本 | 模型根据指令按需读取或执行 |

例如，处理表格时可以先从摘要选择 `office`，再加载 `office-xlsx`，需要渲染或重算时才加载对应技能。也可以直接加载专项技能；框架没有强制先加载路由技能。

第三层通过 `resourceBase` 和正文中的引用组织。资源位置可以是目录、URL 或提供方描述；注册表不会自动递归展开链接或执行脚本。

需要区分“读取文件”和“进入模型上下文”：文件系统发现阶段调用 `parseSkillFile()`，实际上读取并解析整个 `SKILL.md`，然后构造不含正文的候选。加载阶段重新读取文件并返回正文。因此，按需加载主要节省模型输入中的无关内容，不保证首次扫描只读取文件头。

实现与示例见 [tool-skill README](../../packages/skill/tool-skill/README.zh.md) 和 [Office 渐进加载](../../packages/skill/skill-office/README.zh.md#progressive-loading)。

## 三、元数据与正文分离

同一个技能在展示、选择和使用阶段需要不同数据，因此代码区分三种类型。它们定义在 [skill/src/index.ts](../../packages/skill/skill/src/index.ts)。

| 类型 | 主要内容 | 用途 |
|---|---|---|
| `SkillSummary` | 名称、描述、调用策略、来源、提供方及可选资源位置 | 对外查询摘要 |
| `SkillCandidate` | Summary 加上 `rank`、`locator` 和可选 metadata | 内部选择胜出项，并定位正文 |
| `SkillDefinition` | Summary 加上 `content` 和可选 metadata | 返回完整指令 |

以 `pdf` 技能为例，以下字段仅作数据示意：

```text
Summary
  name: pdf
  description: 读取、创建和修改 PDF

Candidate
  Summary 的字段
  rank: 200
  locator: 技能文件路径及其资源目录

Definition
  Summary 的字段
  content: 完整 PDF 操作说明
```

当项目目录和用户目录都提供 `pdf` 时，注册表先比较两个 Candidate，选出一个，再对外返回 Summary。模型实际请求 `pdf` 时，注册表把胜出候选交给对应 provider，获得 Definition。

`locator` 是提供方自己解释的定位数据，注册表只负责传回；`source` 表示来源类别，`provider` 标识负责读取的提供方。同层排名由 `rank` 决定，不直接根据 `source` 字符串判断。

这里的分离是数据结构与使用阶段分离，不要求把元数据和正文拆成两个文件；它们仍然可以放在同一个 `SKILL.md` 中。

## 四、注册与发现

提供方接口将发现与读取分成 `list()` 和 `get()`。注册表通过 `registerProvider()` 接入动态来源，也允许通过 `register()` 直接注册完整技能。

```text
provider.list(options)
  → 候选列表，或候选列表与 complete 状态

provider.get(candidate, options)
  → 完整定义，或 undefined

ctx.skills.snapshot(options)
  → 合并后的摘要列表与 complete 状态

ctx.skills.get(name, options)
  → 加载胜出候选的正文
```

查询的 `cwd` 决定项目来源，`scope` 决定可见层级，`signal` 用于取消异步工作。提供方获得 `control.invalidate()` 通知目录变化，以及随注册释放而中止的 `control.signal`。注册与插件生命周期关联，卸载时移除相应贡献。

文件系统提供方支持 `<root>/<name>/SKILL.md` 和 `<root>/<name>.md`，不会递归发现任意深度的技能树。frontmatter 必须包含合法的 kebab-case `name` 和非空 `description`；正文是去掉 frontmatter 后的 Markdown。

默认允许模型与用户调用。`disable-model-invocation: true` 禁止模型目录和 `skill` 工具使用该技能；`user-invocable: false` 禁止用户显式调用。注册表本身不按调用方式过滤，消费者分别检查策略。

`whenToUse` 可以保存在摘要中，但当前模型目录只渲染名称与描述；关键适用条件应放在 `description` 中。解析和目录行为的维护入口是 [skill-filesystem README](../../packages/skill/skill-filesystem/README.zh.md)。

## 五、作用域与同名技能选择

分层作用域注册表（Scoped Registry）允许 agent 继承全局技能，同时使用预设或自身作用域中的专属技能。遮蔽（Shadowing）表示近层同名技能替代查询结果中的远层技能，远层定义仍然存在。

```text
全局
├── code-review：通用审查
├── pdf：PDF 处理
│
├── frontend 预设
│   └── code-review：前端组件和可访问性审查
│       └── 使用该预设的 agent
│
└── backend 预设
    └── 使用该预设的 agent
```

前端 agent 查询 `code-review` 得到前端版本；后端 agent 若没有其他覆盖，得到全局版本。两者仍可使用全局 `pdf`。

### 作用域如何定义

技能归属由注册时的 Cordis Context 决定，不是在 `SKILL.md` 中添加 `agent` 字段。预设注册表通过 `createScope()` 创建带内部身份标记的 Context，再在该 Context 中挂载预设的插件。技能插件调用 `ctx.skills.register()` 或 `registerProvider()` 时，贡献进入调用 Context 所在的层。

使用者通常在目标预设的 `plugins` 中挂载 `skill-filesystem`，为它指定专属 `customSkillDirs`。`includeDefaultRoots: false` 可以让该提供方只扫描明确配置的来源。预设配置入口见 [agent-preset-registry README](../../packages/preset/agent-preset-registry/README.zh.md)，文件系统字段见 [skill-filesystem README](../../packages/skill/skill-filesystem/README.zh.md)。

框架通过 `bindScopeParent(agentKey, presetKey)` 将 agent 关联到预设作用域；查询使用 `scope: agent`，因此 agent 能继承预设层。同一预设可被多个 agent 共同继承，不必为每个 agent 复制技能。

### 同名技能选择规则

跨层合并顺序是“全局 → 远层祖先 → 近层 → 当前作用域”，后面的同名条目覆盖前面的条目。同一层内才依次比较 `rank`、提供方注册顺序和提供方列表中的候选顺序。

| 同层来源 | rank |
|---|---:|
| 项目 `.dsh/skills` | 100 |
| 项目 `.agents/skills` | 200 |
| 运行时注册 | 250 |
| 自定义目录 | 300 |
| 用户 DSH 目录 | 400 |
| 用户 Agents 目录 | 500 |
| 内置技能 | 600 |

数值越小，同层优先级越高。但当前作用域的 `rank = 600` 仍可以遮蔽全局的同名 `rank = 100`。作用域控制注册表查询可见性，不等同于文件访问权限或安全沙箱。

## 六、技能如何进入模型上下文

模型主动调用和用户显式调用共享正文渲染格式，但进入会话的路径不同。入口位于 [tool-skill/src/index.ts](../../packages/skill/tool-skill/src/index.ts)。

### 模型主动调用

```text
agent/pre-step
  → 检查当前 agent 是否能看到本插件注册的 skill 工具
  → 查询目录并过滤 modelInvocable
  → 发布名称与描述
  → 模型根据任务选择技能
  → skill({ name: "pdf" })
  → 校验名称及调用策略
  → ctx.skills.get() → provider.get()
  → 再次检查加载后的调用策略
  → 渲染正文，作为工具结果进入会话
```

目录描述默认最多 500 个字符，并折叠空白。目录要求模型在任务匹配时先加载指令；这里的任务匹配由模型完成，不是框架关键词分类器自动选择技能。

完整正文由 `renderSkillContent()` 渲染成 `<skill_content>`，内含 `<skill_resources>` 与 `<skill_instructions>`。一次调用返回该技能完整正文，不会自动按章节切片。

### 用户显式调用

用户输入中的独立 `/pdf` token 可以直接触发加载。提取函数只扫描直接用户消息，名称前后需要是空白或文本边界，因此 `/usr/bin` 不会作为技能调用。重复名称会去重；未知名称或不允许用户调用的名称保留为普通文本。

```text
直接用户输入中的 /pdf
  → agent/pre-step 提取名称
  → ctx.skills.get()
  → 检查 userInvocable
  → 注入完整正文
```

注入消息携带 `skill-invocation` 来源，摘要目录携带 `skill-catalog` 来源；模型主动加载则保存在工具历史中。加载不会把技能复制到另一个安装目录。目录变化通过追加消息表达，已记录的历史正文不会被原地改写。

## 七、缓存失效与热更新

事件驱动失效表示文件变化通知注册表“旧目录可能过时”，注册表清除缓存，后续查询重新发现技能。热更新表示运行中的进程可以感知变化，无需为了刷新目录而重启。

```text
新增、修改或删除 SKILL.md
  → 文件监听器，或第一方 write/edit 的 fs/observed 事件
  → control.invalidate()
  → revision 增加，目录缓存清空
  → 下一次 agent/pre-step 查询目录
  → 如果摘要或完整性状态变化，追加替换目录
```

默认启用的文件监听负责感知外部编辑；第一方文件操作还可以直接通知失效。附属资源子树变化不作为目录变化处理。监听的配置与限制见 [skill-filesystem README](../../packages/skill/skill-filesystem/README.zh.md)。

注册表缓存键包含 `cwd`、作用域链身份和 `revision`。只缓存完整且稳定的发现结果；收集期间修订变化会触发有限重试，仍不稳定则返回不完整结果。默认最多保留 128 个目录缓存项，超限移除最早插入的项，不是命中后更新顺序的严格 LRU。

模型目录使用名称、规范化描述和发现完整性计算 digest，内容未变时不重复发布。发现不完整时发布已确认条目和警告，后续查询重试；全部技能消失时发布空替换目录。

| 修改 | 对模型的影响 |
|---|---|
| 新增技能 | 后续目录包含新技能 |
| 删除技能 | 替换目录告知旧名称不可继续使用 |
| 修改描述 | 后续目录展示新摘要 |
| 只修改正文 | 摘要可能不变，再次加载时读取新正文 |

热更新不打断正在进行的模型请求，也不会替换模型已经读过的历史指令。注册表缓存的是候选目录；它没有统一的正文缓存，文件系统提供方在 `get()` 时重新读取正文。

## 八、技能存放目录

技能文件所在目录与加载后的会话内容是两回事。用户编写的技能保留在原目录中，加载操作只读取指令并将其加入会话。

| 范围 | 默认位置 | 当前 Windows 用户的默认路径示例 |
|---|---|---|
| 用户 DSH | `~/.dsh/skills` | `C:\Users\11017\.dsh\skills` |
| 用户 Agents | `~/.agents/skills` | `C:\Users\11017\.agents\skills` |
| 项目 DSH | `<项目根>/.dsh/skills` | 当前项目下的 `.dsh\skills` |
| 项目 Agents | `<项目根>/.agents/skills` | 当前项目下的 `.agents\skills` |

这些是默认解析规则，不表示当前部署一定启用了默认根或对应目录已经存在。`dshHome`、`agentsHome` 或 `DSH_HOME`、`DSH_AGENTS_HOME` 可以调整用户根，`customSkillDirs` 可以添加其他扫描目录。项目根按 `cwd` 向上寻找最近的 `.git`，找不到时使用原始 `cwd`。

目录式技能可以组织为：

```text
<技能根目录>/my-skill/
├── SKILL.md
├── references/
│   └── details.md
└── scripts/
    └── helper.py
```

目录式技能以自身目录作为相对资源基目录；平铺 `<name>.md` 以扫描根作为资源基目录。自动发现不会递归把 `references/` 或 `scripts/` 当作独立技能。

## 九、技术特点与优势

这些机制分别改善上下文使用、来源扩展、定制和运行管理；它们不保证模型一定选对技能，也不把普通指令变成强制执行流程。

| 特点 | 机制 | 优势 |
|---|---|---|
| 渐进式披露 | 摘要、正文、附属资源分阶段进入上下文 | 减少无关指令的输入开销 |
| 元数据与正文分离 | Summary、Candidate、Definition 分别承担展示、选择和加载 | 明确数据职责，避免目录携带全部正文 |
| 提供方抽象 | 统一 `list()` / `get()` | 扩展来源无需改写模型工具 |
| 分层作用域与遮蔽 | 近层覆盖远层，同层确定性排序 | 共享公共技能并支持预设或 agent 定制 |
| 修订感知缓存 | 以工作目录、作用域链和修订号区分缓存 | 减少重复扫描，不长期缓存不完整发现 |
| 事件驱动热更新 | 文件变化触发失效，后续步骤发布目录 | 无需重启即可感知技能变化 |
| 双入口与调用策略 | 模型工具和用户显式调用分别检查权限 | 支持自动调用与仅用户触发的技能 |
| 会话持久化 | 目录、注入和工具结果进入会话记录 | 可追踪模型获得的内容，支持恢复和分叉 |
| 生命周期管理 | Cordis effect、disposer 和 AbortSignal | 清理注册与监听资源，取消等待中的工作 |

## 十、源码阅读顺序

先看数据类型与来源，再追踪模型入口，最后阅读缓存和生命周期，可避免一开始陷入监听器细节。

1. [skill/src/index.ts](../../packages/skill/skill/src/index.ts)：阅读三个数据类型、`SkillProvider`、`list()` / `snapshot()` / `get()`、`renderSkillContent()`。
2. [skill-filesystem/src/index.ts](../../packages/skill/skill-filesystem/src/index.ts)：阅读 `roots()`、`discoverRoot()`、`parseSkillFile()` 和提供方 `get()`。
3. [tool-skill/src/index.ts](../../packages/skill/tool-skill/src/index.ts)：阅读工具 `execute()`、两个 `agent/pre-step` 监听器、目录 digest 和调用名称提取。
4. [scope/src/index.ts](../../packages/core/scope/src/index.ts) 与 [agent-preset-registry/src/index.ts](../../packages/preset/agent-preset-registry/src/index.ts)：理解 `createScope()`、`bindScopeParent()` 和预设插件挂载。
5. 回到 `SkillRegistry.collect()` 与 `SkillWatchManager`，理解目录缓存、变化通知和资源释放。
6. [skill-office/src/index.ts](../../packages/skill/skill-office/src/index.ts) 与 [渐进加载测试](../../packages/skill/skill-office/tests/progressive-loading.spec.ts)：对照具体技能的按需加载，以及恢复、分叉和用户显式调用场景。

## 开发备注

本页用于源码学习与概念检索；维护中的接口和配置以链接的源码、包 README 及子系统参考为准。




完善的skills管理策略： 渐进式披露、分层作用域与遮蔽、rank冲突裁决、skill热更新、双入口调用（模型调用和 /name 调用）