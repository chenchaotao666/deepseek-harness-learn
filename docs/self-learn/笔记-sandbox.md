# Sandbox 学习笔记：策略、命令执行与系统限制

## 概览

DeepSeek Harness 通过共享沙箱策略约束文件工具和 Shell 命令。模型生成工具调用请求，Harness 决定如何执行；文件工具在修改前检查目标路径，Shell 执行器在创建进程前调用平台沙箱后端。默认组合使用 `workspace-write` 和 `ask` 审批策略，部署配置和会话权限可以覆盖默认值。

本笔记面向了解文件路径、命令行和子进程的读者，作为沙箱实现的学习参考。代码、参数和目录示例用于解释调用关系，不是已经执行的安全验证；精确接口与平台保证以链接的源码和包文档为准。

## 目录

- [1. 沙箱模式与限制范围](#1-沙箱模式与限制范围)
- [2. sandbox 包组与 fs-sandbox 的关系](#2-sandbox-包组与-fs-sandbox-的关系)
- [3. 模型命令如何进入沙箱](#3-模型命令如何进入沙箱)
- [4. sandbox-local 如何包装命令](#4-sandbox-local-如何包装命令)
- [5. Linux 的 bwrap 与 Landlock](#5-linux-的-bwrap-与-landlock)
- [6. macOS 的 Seatbelt](#6-macos-的-seatbelt)
- [7. Windows 的受限令牌与 ACL](#7-windows-的受限令牌与-acl)
- [8. 权限拒绝与审批重试](#8-权限拒绝与审批重试)
- [9. 接入范围与使用限制](#9-接入范围与使用限制)
- [10. 阅读资料](#10-阅读资料)

## 1. 沙箱模式与限制范围

沙箱模式主要约束文件修改效果，包括写入、编辑、删除和移动；模式名称不承诺统一的网络隔离或读取隔离。

| 模式 | 含义 |
|---|---|
| `read-only` | 文件工具拒绝修改；进程后端按平台规则限制写入，保留必要的特殊目标，例如 POSIX 的 `/dev/null` |
| `workspace-write` | 允许工作区及后端支持的额外授权目录、临时区域写入 |
| `danger-full-access` | 绕过 Harness 沙箱限制，仍受操作系统本身权限约束 |

默认 base 组合的模式来自 `DSH_PERMISSION_MODE` 或 `workspace-write`，权限预设将受限模式与 `ask` 审批策略关联。配置见 [base bundle](../../packages/bundle/base/cordis.patch.yml)。正常工具调用的工作区授权根来自 Session 创建时的工作目录；切换命令执行目录不会自动扩大授权范围。

例如，在工作区 `E:\projects\demo` 下，文件工具可以修改 `src\index.ts`；修改 `E:\other-project\important.txt` 需要相应的更宽授权。文件读取仍按本地文件系统权限执行，不能从“工作区可写”推断“只能读取工作区”。

这套本地沙箱共享宿主机内核和文件系统，不默认启动 Docker 容器或虚拟机。容器或远程执行环境需要替换相应的文件系统、子进程等能力；具体远程组合由其提供方负责。

## 2. sandbox 包组与 fs-sandbox 的关系

`packages/sandbox` 是包组，`packages/fs/fs-sandbox` 是一个具体包。两条执行路径共享策略，但采用不同的执行限制机制。

| 包 | 职责 |
|---|---|
| `sandbox/sandbox-policy` | 解析每次调用的模式、工作区和显式额外写入授权，提供 `ctx.sandboxPolicy` |
| `sandbox/sandbox` | 定义进程沙箱服务 `ctx.sandbox`、策略和执行能力等类型 |
| `sandbox/sandbox-local` | 选择本地系统后端，将策略转换为启动器参数 |
| `sandbox/sandbox-windows-acl` | 提供 Windows 令牌、ACL 授权和 runner 实现 |
| `fs/fs-sandbox` | 提供受策略约束的 `ctx.fs`，检查文件修改目标 |
| `shell/bash-sandbox`、`shell/pwsh-sandbox` | 将 Shell 执行接到 `ctx.sandbox`，启动包装后的命令并解释结果 |

```text
                       sandbox-policy
                   解析本次调用的权限
                            |
              +-------------+-------------+
              |                           |
           文件工具                    Shell 工具
              |                           |
          fs-sandbox              bash/pwsh-sandbox
              |                           |
        检查实际目标路径              ctx.sandbox
              |                           |
        执行文件修改                sandbox-local
                                          |
                                 系统限制下的子进程
```

`fs-sandbox` 检查 `writeText`、`editText`、`patchText`、`deleteFile` 和 `moveFile`；移动必须同时通过源路径和目标路径检查。它会在操作前重新规范化路径，判断目标是否位于允许写入的目录中。拒绝时返回 `FS_SANDBOX_DENIED`。

文件工具的检查发生在可信代码里，不会把每次文件操作交给 `sandbox-local`。任意 Shell 命令内部的文件访问不经过 `ctx.fs`，所以需要进程沙箱限制。反过来，文件工具可直接在 Harness 进程中操作文件，也不能只依靠子进程沙箱。路径检查的并发替换限制见 [fs-sandbox 的威胁模型](../../packages/fs/fs-sandbox/README.md#threat-model)。

## 3. 模型命令如何进入沙箱

模型只输出工具调用数据，真正创建进程的是 Harness。以“运行 Python 生成文件”为例，模型可能产生如下语义上的请求；这里省略协议包装，实际工具参数可能以 JSON 字符串传递：

```json
{
  "name": "bash",
  "arguments": {
    "command": "python task.py"
  }
}
```

此时 `python task.py` 只是字符串，没有任何 Python 进程被启动。Harness 控制执行入口，调用链如下：

```text
模型返回 tool-call
  → agent-loop 收集工具调用
  → ctx.tools.execute 分发到已注册工具
  → tool-bash 校验参数、解析 Session 策略和审批
  → ctx.shell.resolve 生成执行规格
  → ctx.shell.execute
  → bash-sandbox 调用 ctx.sandbox.confine
  → sandbox-local 返回包装后的 argv
  → subprocess 启动沙箱程序
  → 沙箱程序在限制生效后执行原命令
```

Windows 默认接入 PowerShell 工具及其沙箱执行器，遵循相同的分层关系。Bash 工具的执行入口见 [tool-bash](../../packages/shell/tool-bash/src/index.ts)，Loop 分发见 [agent-loop](../../packages/core/agent-loop/src/agent.ts)。

工具层调用 Shell 服务的过程可以概括为下面的示意代码：

```javascript
const spec = ctx.shell.resolve(request);
const execution = await ctx.shell.execute(spec);
const result = await execution.result();
```

`ctx.shell` 的实际实现由插件组合决定。默认组合加载受沙箱约束的执行器，因此这次服务调用进入命令包装路径。模型在参数里写“忽略沙箱”，不会修改服务实现或系统规则。

## 4. sandbox-local 如何包装命令

`confine(argv, policy, signal)` 接收准备执行的参数数组和已经解析的策略。它规范化工作区路径，选择后端，并返回包装后的参数；实际进程由调用方启动。Windows 的准备阶段还会物化目录授权和私有临时目录。

输入示意：

```javascript
const argv = ["bash", "-c", "python task.py"];
const policy = {
  mode: "workspace-write",
  workspaceRoot: "/home/alice/project",
  sessionId: "session-001"
};
```

返回结果包含四类信息：

| 字段 | 用途 |
|---|---|
| `argv` | 启动器、权限规则、分隔符和原始命令参数 |
| `enforcement` | 后端报告的 `full` 或 `partial` 执行能力 |
| `denialSignatures` | 识别权限拒绝的诊断特征 |
| `runnerFailureRules` | 区分启动器失败与目标命令失败的规则 |

后端选择由平台决定：Linux 依次功能探测 bwrap 和 Landlock，缓存首个可用结果；macOS 和 Windows 各只有一个候选，直接选择，在实际启动失败时仍阻止命令执行。未知平台或所有候选不可用时返回 `SANDBOX_UNAVAILABLE`。

操作员可配置 `runnerCommand` 覆盖默认选择，但该配置跳过内置探测，并假定自定义程序确实实现 bwrap 兼容规则；不能把配置存在当作安全验证。核心实现见 [sandbox-local](../../packages/sandbox/sandbox-local/src/index.ts)。

## 5. Linux 的 bwrap 与 Landlock

Linux 的两个后端分别通过挂载视图和进程文件访问规则施加限制。两者提供相同的策略入口，但临时目录和内核支持能力不完全相同。

### bwrap：只读文件树与可写挂载

`workspace-write` 下生成的命令参数大致如下。这是实现示意，不是本笔记要求执行的操作：

```text
bwrap
  --ro-bind / /
  --dev /dev
  --unshare-pid
  --proc /proc
  --die-with-parent
  --tmpfs /tmp
  --bind /home/alice/project /home/alice/project
  --
  bash -c "python task.py"
```

| 参数 | 效果 |
|---|---|
| `--ro-bind / /` | 将宿主文件树提供为只读挂载 |
| `--bind 工作区 工作区` | 为工作区建立可写挂载，支持的额外目录也采用这一方式 |
| `--tmpfs /tmp` | 提供临时文件系统 |
| `--dev /dev` | 提供新的设备目录 |
| `--unshare-pid`、`--proc /proc` | 使用私有 PID 命名空间及对应的 procfs |
| `--die-with-parent` | 父进程退出时终止受控进程 |

`read-only` 不添加工作区可写挂载和上述 `/tmp` tmpfs。具体参数见 [profiles.ts](../../packages/sandbox/sandbox-local/src/profiles.ts)。

假设 Python 试图写入 `/home/alice/other/result.txt`，操作系统会按只读挂载规则拒绝。Harness 不需要解析 Python 源码，也不需要预先猜出它计算出的文件路径。Shell 再调用脚本或启动后代进程，仍在这条受限执行路径中运行。

### Landlock：给进程安装访问规则

bwrap 不可用时，Harness 尝试原生 Landlock 启动器。启动器创建 ruleset，添加路径授权，设置 `PR_SET_NO_NEW_PRIVS`，调用 `landlock_restrict_self`，随后执行目标程序。限制随执行和后代进程延续。

| 模式 | Harness 生成的主要路径授权 |
|---|---|
| `read-only` | `/` 可读，`/dev/null` 保留必要写入 |
| `workspace-write` | 进一步允许 `/tmp`、工作区和显式额外目录写入 |

Landlock 对 `/tmp` 的授权不等于创建独立 tmpfs。旧内核 ABI 只能覆盖部分文件操作时，探测会报告 `partial`。规则映射由 `profiles.ts` 维护，原生限制过程见 [Landlock 启动器](../../native/system/packages/entry/src/main.c)。

## 6. macOS 的 Seatbelt

macOS 使用 `sandbox-exec` 加载生成的 Seatbelt 策略，再执行原始命令。策略原则是默认允许其他操作、拒绝文件写入，然后为允许的路径增加例外。

简化策略示例：

```text
(version 1)
(allow default)
(deny file-write*)
(allow file-write* (literal "/dev/null"))
(allow file-write* (subpath "/Users/alice/project"))
```

实际 `workspace-write` 策略还根据共享辅助函数加入允许的临时目录和额外写入目录。路径先规范化，因为 Seatbelt 匹配实际路径，例如 `/tmp` 对应的 `/private/tmp`。

生成的调用形式是 `sandbox-exec -p <策略> -- <原始参数>`。这份策略主要限制文件写入；`allow default` 不会建立统一网络隔离。

## 7. Windows 的受限令牌与 ACL

Windows 使用 `sandbox-windows-acl` runner。它准备目录授权，复制当前调用者的访问令牌并创建 `WRITE_RESTRICTED` 令牌，将其降到 Low integrity，再使用该令牌启动目标子进程。runner 将子进程加入 `KILL_ON_JOB_CLOSE` Job，并转发输出和退出码。

| 机制 | 职责 |
|---|---|
| SID | 标识工作区和私有临时目录对应的授权身份 |
| ACL / ACE | 在目录安全描述符中表达允许和拒绝规则 |
| `WRITE_RESTRICTED` | 写权限同时接受普通身份与限制身份的访问检查 |
| Low integrity | 按完整性等级进一步约束可写目标 |
| Job Object | 管理子进程树，Job 关闭时终止所属进程 |

`workspace-write` 的限制身份包含工作区 SID 和私有临时目录 SID；`read-only` 不携带这些写入身份。为了防止通过父目录删除权绕开目标自身的权限，授权还拒绝 `FILE_DELETE_CHILD`，让删除操作依赖目标对象上的删除权限。详细的保留身份和系统调用规则见 [Windows 后端机制](../../packages/sandbox/sandbox-windows-acl/README.md#mechanism)。

工作区 SID 由规范化后的路径确定，同一工作区可以复用授权。每个活跃 Session/工作区组合另有随机私有临时目录和不同的 SID；runner 将 `TMP`、`TEMP` 指向它，避免共享工作区权限同时授权其他会话的临时目录。

### Windows 的特有限制与持久影响

Windows 后端明确报告 `partial`，不能将它理解为完整容器隔离：

- 读取、网络和进程可见性不由这套写限制完整隔离。
- NTFS 硬链接让多个路径指向同一个文件对象，目录授权可能影响工作区外的同一文件别名。
- 某些被其他 AppContainer 工具设置 ACL 的目录可能对受限进程不可读。
- 当前后端拒绝 `additionalWritableRoots`，不能假定所有平台都支持额外可写目录。
- 工作区 ACE、拒绝项和 Low 标签是持久目录变更，正常释放不会撤销；同一用户的其他 Low-integrity 进程也可能因此获得原先被完整性等级阻止的工作区写入能力。
- 私有临时目录授权会在清理时撤销，工作区授权和临时授权具有不同生命周期。

任何关键 Win32 操作失败都会阻止目标程序按无沙箱方式启动。源码入口为 [AclSandbox](../../packages/sandbox/sandbox-windows-acl/src/index.ts)、[令牌处理](../../packages/sandbox/sandbox-windows-acl/src/token.ts)和 [runner](../../packages/sandbox/sandbox-windows-acl/src/runner.ts)。

## 8. 权限拒绝与审批重试

审批决定某次操作是否获得更宽授权，沙箱执行器只执行已解析的策略。模型请求更高权限时，工具层先校验参数并进入审批流程；只有获得授权后，才将更宽的模式传给本次执行。请求参数本身不等于批准。

```text
模型请求执行命令
  → 按现有策略运行
  → 操作被拒绝，工具结果带回拒绝信息
  → 在审批策略允许时申请更宽权限
  → 获批后，以新的本次策略重试
```

工具也可以在执行前提出权限申请。一次获批的更宽重试不要求永久放宽整个会话；`danger-full-access` 是明确授权或配置后的路径，此时消费者绕过 `ctx.sandbox`，直接运行原始参数。

| 失败类型 | 例子 | 含义 |
|---|---|---|
| 文件工具拒绝 | `FS_SANDBOX_DENIED` | 目标路径未通过当前策略 |
| 受限命令拒绝 | 只读文件系统、权限拒绝 | 系统后端限制了目标操作 |
| 沙箱不可用 | `SANDBOX_UNAVAILABLE` | 找不到可用后端，不能开始受限执行 |
| runner 自身失败 | 规则安装失败或启动失败 | 不能假定目标命令已经运行 |

Shell 执行器结合退出状态和后端诊断规则区分 runner 失败与权限拒绝；runner 失败优先分类。受限模式失败不会自动切换为无限制执行。后台执行同样经过沙箱准备，并不会因成为 job 就获得更宽权限。

Session 权限变更通过 `sandbox/mode` 等持久事件保存，运行时上下文向模型说明当前策略。Session 日志保存权限事实，不替代每次执行时的策略解析和系统限制。

## 9. 接入范围与使用限制

`sandbox-local` 不全局替换 Node 的进程创建 API，也不拦截模型以外的所有宿主行为。保证来自默认工具、服务组合和平台后端共同参与的执行路径。

| 操作入口 | 约束路径 |
|---|---|
| 默认 Bash / PowerShell 工具 | 受限 Shell 执行器调用 `ctx.sandbox` |
| 默认文件修改工具 | `ctx.fs` 的 `fs-sandbox` 实现检查路径 |
| 自定义插件直接调用 `child_process.spawn()` | 不会仅因加载了 `sandbox-local` 就自动走沙箱 |
| 外部 MCP 服务执行操作 | 由服务自身部署和执行机制决定，本地 Shell 沙箱不会自动覆盖 |
| 使用 `danger-full-access` 的调用 | 绕过 Harness 沙箱包装 |

插件实现如果需要受限执行，应通过对应能力服务和已解析策略执行，不能把模型参数直接交给未经约束的宿主 API。`full` 表示对该模式承诺的文件效果完整执行，不表示读取、网络、所有进程行为均已隔离；`partial` 则要求调用方正视具体平台缺口。

## 10. 阅读资料

- [沙箱包组](../../packages/sandbox/README.md)：策略、接口与后端的包关系。
- [进程沙箱参考](../subsystems/sandbox.md)：模式、每次调用策略和失败类型。
- [共享策略](../../packages/sandbox/sandbox-policy/README.md)：会话模式与工作区授权来源。
- [本地沙箱后端](../../packages/sandbox/sandbox-local/README.md)：后端选择、执行能力与平台差异。
- [文件沙箱](../../packages/fs/fs-sandbox/README.md)：直接文件修改的路径检查。
- [Bash 沙箱执行器](../../packages/shell/bash-sandbox/README.md)及 [PowerShell 沙箱执行器](../../packages/shell/pwsh-sandbox/README.md)：命令准备、执行与结果分类。
- [Windows ACL 后端](../../packages/sandbox/sandbox-windows-acl/README.md)：令牌、授权生命周期与限制。

## Dev Note

本文件是个人学习参考。内容依据当前源码和包文档整理，没有实际运行各平台沙箱命令或验证权限拒绝；平台安全保证由对应后端及其测试维护。
