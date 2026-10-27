# 知名 Agent Harness 调研

调研目的：为插件系统设计和 M1 的内核边界提供参照（见 [STATUS.md](../STATUS.md) 待决问题 1）。

范围：

- 完整拆解：DeepSeek Harness、Pi、Hermes Agent。
- 只看特色：Letta、Graphiti、mem0、OpenHands、Codex CLI。

方法：克隆各仓库最新提交阅读源码和文档，结论尽量对应到具体包或文件。调研时间 2026-10-27。

> 状态：初读完成。dsh、Pi 读到了包级文档和部分源码；Hermes 只读了各目录的设计说明和用户文档；其余五个只看了特色。各节结论还需要在讨论中核实。
>
> 调研中发现两个仓库已迁移：Letta 的现行源码在 `letta-ai/letta-code`，OpenHands 的 agent 内核在 `OpenHands/software-agent-sdk`，本文按迁移后的仓库阅读。

## 拆解维度

完整拆解的项目按同一组维度记录，便于横向对比：

1. 分层结构：内核包含什么，哪些是插件。
2. Agent loop：一轮怎么跑，在哪些节点发事件，中断、排队、转向怎么处理。
3. 上下文与记忆：提示词怎么组装，压缩何时触发、由谁执行，长期记忆放在哪里。
4. 会话与日志：存储格式，是否支持分支、回放、迁移。
5. 插件机制：扩展单元、注册点、插件间通信、隔离与分发。
6. 后台与主动能力：定时任务、子 agent、渠道网关。
7. 对本项目的启示。

---

## 1. DeepSeek Harness（dsh）

仓库：`deepseek-ai/deepseek-harness`，提交 `22ddee3`（2026-10-23）。TypeScript monorepo，约 1.9 万个文件，`packages/` 下 50 多个包分组。官方定位为 "Everything is a Plugin"。

### 1.1 分层结构

- 底层是 vendor 进来的插件框架 **Cordis**（源自 Koishi 机器人框架生态）。dsh 没有"特权内核"：agent loop、模型适配器、工具注册表、会话日志都是插件，都能通过配置替换。
- 主干插件（`packages/core/` 与 `packages/llm/llm`）：

  | 包 | 职责 | 服务键 |
  | --- | --- | --- |
  | session | 只追加的会话事件日志、内存存储 | `ctx.sessions` |
  | system-prompt | 提示词片段与工具 schema 的组装 | `ctx.systemPrompt` |
  | tools | 作用域化的工具注册表、带拦截的执行流水线 | `ctx.tools` |
  | agent | Agent 接口、活跃 agent 注册表、`agent/*` 事件 | `ctx.agents` |
  | agent-loop | 默认驱动器（约 2100 行） | `ctx.agentLoop` |
  | scope | 按 agent 划分作用域的注册原语 | 库 |
  | llm | 消息与流式词汇表、适配器 seam | `ctx.llm` |

- 其余全部是能力包：fs、shell、terminal、sandbox、lsp、web、mcp、skill、subagent、jobs、schedule、goal、compaction、context、plan、todo、interaction（审批、询问用户）、credentials、settings、storage 等。

### 1.2 Cordis 插件模型

五个核心概念（`docs/cordis-primer.md`）：

1. **插件**：一个带 `apply(ctx, config)` 的函数或模块，或一个 `Service` 子类；配置用 schema 校验。
2. **ctx 是服务容器**：服务占据固定键，其他插件按键查找，不 import 具体实现。
3. **`inject` 声明依赖**：依赖的服务就绪后插件才启动，没有手工编排的启动顺序。
4. **类型化事件**：事件名通过 TypeScript 声明合并注册，有五种分发模式，模式是事件契约的一部分：

   | 模式 | 是否 await | 语义 |
   | --- | --- | --- |
   | `emit` | 否 | 只通知 |
   | `waterfall` | 否 | 中间件式，监听器收到 `next()`，可修改、包装或短路 |
   | `parallel` | 是 | 并行扇出 |
   | `serial` | 是 | 按序执行 |
   | `bail` | 否 | 遇到第一个 bail 值即停止 |

5. **注册是可撤销的副作用**：工具、提示词片段、监听器都通过 `ctx.effect()` 或 `ctx.on()` 安装，插件卸载或热重载时自动撤销。

### 1.3 事件分三个域

- **会话事件**：持久事实，追加到日志并广播（`turn/*`、`step/*`、`user/message`、`assistant/message`、`tool/*` 等）。重启后仍需存在的事实用这一类。
- **Agent 事件**（`agent/*`）：观察或拦截进行中的工作。重要的有：
  - `agent/pre-step`（waterfall）：每一步开始前接纳、改写或拒绝输入。时间上下文、指令文件、压缩、plan mode、技能、Claude Code hooks 兼容都挂在这里。
  - `agent/request-error`（waterfall）：重试、溢出后压缩都挂在这里。
  - `agent/turn-stopping`（serial）。
  - `agent/inbox/*`：收件箱插入、领取、丢弃。
- **能力事件**：`tools/pre-execute`、`tools/execute`、`tools/post-execute`、`fs/write-intent`、`llm/stream` 等，用于挂策略和适配器。

仓库用脚本从源码生成"事件生产方与消费方矩阵"（`docs/event-producer-consumer.md`），CI 校验它是否最新。

### 1.4 轮次流程

一个**步骤**是一次模型请求加上它调用的工具；一个**轮次**包含零个或多个步骤。

```text
turn/start
  从收件箱领取输入；按当前状态组装提示词片段和工具 schema
  -> agent/pre-step（可拒绝、改写）
     step/start -> 准备调用 -> 提交 system/user 消息到日志
     从日志派生模型历史 -> 流式请求
     assistant/message -> tool/call* -> tools/pre|execute|post -> tool/result*
     step/end
     工具结果待回复，或收件箱有新输入 -> 下一步
  -> agent/turn-stopping
turn/end
```

- 所有输入（用户消息、注入的上下文、定时提醒）都进同一个**收件箱**。`agent.inject()` 注入的上下文要等到下一条唤醒消息才生效。
- 取消时保留已流式交付的文本；失败的步骤补记缺失的工具结果。

### 1.5 会话日志

- 核心规则：**模型可见即已记录**。每一次模型请求都必须能从会话日志完整重建，新增模型可见输入必须对应会话事件。
- 模型历史由 `deriveMessages()` 从日志投影；fork、恢复、文本记录、遥测都从日志派生。
- 投影 seam `ctx.sessionProjections`：插件注册的投影单元增量折叠已提交的事件。
- 存储：JSONL（可选 zstd 压缩），文件名带格式版本号；格式升级通过逐版本的迁移链完成，已提交的文件不改名、不覆盖。

### 1.6 能力 seam

每项可替换能力拆成三个角色：Service Definition（接口）、Service Provider（实现）、Consumer（通常是面向模型的工具）。规则是"插件依赖接口，绝不依赖具体 provider"。例如把 fs 与 subprocess 的 provider 换成远程沙箱，Bash、PTY、LSP 就一起切换到远端。

### 1.7 组装与分发

- 运行中的 dsh 是一棵插件树，由分层的 YAML patch 叠成：bundle（组合包）→ profile → 用户 patch → `--patch`。每一层按条目 id 整条替换 config，或插入新条目。
- 基础组合包 `dsh-base` 约 97 个条目；`web`、`headless`、`sdk`、`acp` 等 profile 在其上叠加。`dsh --dump-config` 打印最终插件树。
- 插件以 npm 包分发，有插件管理器和插件市场；支持热重载（HMR）。
- Python SDK 只是启动 `dsh` 可执行文件并走 JSON-RPC，不是 Python 实现。

### 1.8 插件示例：repeat-tool-reminder

`packages/guard/repeat-tool-reminder`，约 240 行，可以看出插件的典型粒度：

- 在 `tools/post-execute` 上统计每个 agent 连续重复的相同调用，到达阈值（默认 3、5、8）时往下一次请求附加一条提醒。它先调用 `next()` 让下游决定，再把提醒合并到结果上，只增补、不否决。
- 在 `agent/pre-step` 上发现用户插话就清零计数。
- 状态存在 `WeakMap<Agent, Chain>` 里，随 agent 释放。

### 1.9 和本项目相关的现成插件

- **schedule**：持久提醒（一次性、固定间隔、每日、每周、cron，显式时区）。宿主重启后每个重复任务只补发最近一次错过的时点；到期时恢复冷会话，并把消息投递进原会话收件箱。更新用完整记录做 compare-and-set。README 写明了已知边界：会话 flush 和任务写入是两次独立写入，崩溃时可能重复投递。可作为 M3"承诺"的参考。
- **goal**：跨多轮、跨重启持续存在的目标，有 create、pause、resume、complete、block 等状态，自动续行有轮数上限。
- **compaction-basic**：token 压力上升时把最旧的历史压缩成摘要，保留近期消息；上下文溢出报错后压缩并重试；也支持 `/compact` 手动触发。
- **jobs / subagent / webhook**（webhook 仍在实验）：后台任务、子 agent、由外部事件启动会话。
- **hooks-claude-code / hooks-codex**（实验）：兼容 Claude Code、Codex 的 hook 协议。

### 1.10 收件箱与 agent 管理

- agent 有两种启动方式：在配置里声明（插件加载时自动启动），或通过 `ctx.agents.create()` / `ctx.agents.resume()` 以编程方式创建、恢复。两种方式都返回带 `dispose()` 的句柄。
- 收件箱的每次变化（插入、编辑、移除、领取、取消）都提交为一条规范化的 `agent/inbox/spliced` 会话事件，所以收件箱本身也能从日志回放。没有活跃 agent 时，待处理输入仍然可读。
- 取消是协作式的：`agent.cancel()` 中止当前活动，默认同时清空待处理工作，可以用 `keepInbox` 保留。
- 工具调用默认最多并行 10 个，执行器按模型给出的顺序返回结果。

### 1.11 上下文组装

- **提示词片段**：插件向 `ctx.systemPrompt` 注册 `PromptSection`（名称、顺序号、静态文本或按组装上下文动态生成的函数），组装时按顺序号拼接，再经过 `system-prompt/assemble` waterfall。片段可按 agent 作用域注册。
- **系统提示词也是历史中的一条消息**：渲染出的提示词作为 `system/message` 节点提交到日志。第一步追加为第 0 号节点，之后文本变化时原地替换；模型支持时，也可以把更新追加到已缓存历史的后面。
- **上下文注入是持久的**：context 组的插件（指令文件、时间、引用其他会话、tmux 位置）把内容作为带来源标记的用户角色消息追加进会话历史，和普通对话一样持久、可回放、可压缩。例如 time-context 默认最多每 10 分钟注入一次当前时间和距上一条消息的时长。
- **这和本项目的决定不同**：本项目定的是"上下文每一轮按当前状态重新组装"；dsh 的做法是增量追加，并尽量保持前缀稳定。推测原因是为了利用 prompt cache 并保证可回放。这一点需要专门讨论。
- **控制工具结果大小**：spill-policy 把超大的工具结果截成首尾两段，完整内容写入文件，并告诉模型文件路径；compaction-tool-result-pruner 在压力大时先修剪旧的工具输出（不调用模型），修剪不够才做摘要压缩。原始结果始终保留在日志里。

### 1.12 作用域、存储与热重载

- **作用域**（`dsh-scope`）：每个 agent 或分组有一套隔离的注册集合。子作用域继承祖先的注册，就近覆盖；释放作用域时移除它拥有的全部内容。subagent 和 preset 就靠这个获得不同的工具集。
- **两种持久化分开**：会话事件进会话日志；不属于对话的应用数据（工作区、定时任务等）进 `ctx.storageDomain`，每个领域有版本号和 schema，数据损坏时拒绝启动，而不是静默丢弃。
- **后台任务**（jobs）：长时间运行的工具得到一个任务 id，归属于它的 agent；完成时以会话内通知送达，不需要轮询。
- **热重载与插件管理**：HMR 把模块替换、配置变更放进同一个队列；插件管理器可以启停条目、安装或删除组合包，启用 HMR 时立即生效。
- **agent 自己写插件**：extensions 组提供只读的运行时 API 查询工具，"创造模式"下 agent 能为自己编写并挂载插件。

### 1.13 初步启示

可借鉴（与语言无关）：

- 用"服务容器 + 依赖等待 + 可撤销注册"定义插件。
- 事件分持久和实时两个域；用 waterfall 做拦截。
- 能力 seam 三角色。
- "模型可见即已记录"，与本项目"原始日志是唯一事实来源"一致。
- 所有输入统一进收件箱，定时提醒也只是一条投递进来的消息。

需警惕：

- 规模和复杂度来自一个团队、数千个 PR，以及大量 CI 校验脚本，不能照搬到 MVP。
- dsh 连 agent loop 都可以替换；本项目暂定内核包含 loop，比它保守。
- Python 没有 Cordis 的对应物。`pluggy` 只有 hook 调用，没有服务容器、依赖等待和可撤销副作用。
- 上下文增量追加与本项目"每轮重新组装"的决定冲突，需要权衡 prompt cache 命中率和上下文的可控性。

---

## 2. Pi

仓库：`badlogic/pi-mono`（现归属 earendil-works 组织），2026-10-27 的提交。TypeScript monorepo，约 2000 个文件。定位是"极简、可扩展、你能改成自己想要的样子"：默认内置的东西尽量少，刻意不做子 agent 和 plan mode，交给扩展或用户自己让 Pi 去写。OpenClaw 就建立在 Pi 之上。

### 2.1 分层结构

按依赖从下到上（行数为 `src/` 下非测试代码）：

| 包 | 职责 | 规模 |
| --- | --- | --- |
| `pi-ai` | 统一的多厂商模型 API：流式、工具调用、模型目录 | 约 2.9 万行（大部分是各厂商适配） |
| `pi-agent-core` | 有状态的 agent：loop、工具执行、事件流、消息队列 | 约 2500 行 |
| `pi-coding-agent` | CLI 产品：会话树、扩展、技能、压缩、TUI、RPC、SDK | 约 7.6 万行 |
| `pi-tui` | 差量渲染的终端 UI 库 | 约 2 万行 |
| `chord` | 新：跨进程对象、可热重载插件、状态复制的宿主运行时 | 约 1.5 万行 |
| `pi-durable` | 新（实验）：持久化 agent harness，崩溃后可恢复 | 约 2.8 万行 |
| `pi-codemode` | 在 QuickJS 沙箱里运行模型写的 JS，脚本只能调用注入的工具 | 约 3000 行 |
| `pi-mcp` | MCP 支持 | 约 3300 行 |

"内核"就是 `pi-agent-core`，约 2500 行，不含会话存储、压缩、扩展加载。这些都在上层产品包里。内置工具只有 read、write、edit、bash（Windows 上是 powershell）、grep、find、ls。

### 2.2 Agent loop（pi-agent-core）

- 一个 **turn** 是一次模型请求加上它的工具调用；一个 **run** 是从一次输入到最终回答的若干 turn。
- 两条消息队列：
  - **steer**：在当前这轮工具执行完后插入，加入正在进行的工作。
  - **follow-up**：等 agent 完成手头工作后再处理，开启下一个 run。
  - 中止时把排队的消息退回编辑器。
- loop 对外暴露的钩子很少，但都在关键位置：
  - `transformContext`：裁剪旧消息、注入外部上下文。
  - `convertToLlm`：把应用自定义的消息类型转成模型能懂的 user、assistant、toolResult，并过滤掉只给 UI 看的消息。
  - `prepareRequest`：每次请求前替换上下文，例如从会话存储加载规范化的上下文。
  - `beforeToolCall` / `afterToolCall`：拦截、改写工具调用和结果。
  - `finishTurn`：一轮结束时决定"结束"或"再请求一次"。
- 工具默认并行执行，持久化的结果仍按模型给出的顺序排列；单个工具可声明必须串行。
- 消息类型可以通过声明合并扩展，应用可以在历史里存自己的消息类型，只在发给模型前转换。

### 2.3 上下文与压缩（pi-coding-agent）

- 系统提示词由基础指令和发现的上下文文件（AGENTS.md 等）组成，请求里带上工具定义和技能简介。技能全文按需加载（遵循 agentskills.io 规范）。
- 初始提示词和工具集记录在会话的第一条系统消息里；之后工具或提示词变化时，在下一次请求前追加变更。厂商无法表示这种变化时，才发一份完整的转录检查点，此时可能让缓存前缀失效。和 dsh 一样，Pi 也是为了 prompt cache 而尽量只追加。它甚至有 `cache_warming_decision` 事件，用来在空闲时决定是否刷新缓存。
- **压缩**：
  - 当 `上下文 token > 窗口 - reserveTokens`（默认预留 16384）时触发。从后往前保留约 `keepRecentTokens`（默认 2 万）的近期消息，其余部分调用模型做结构化摘要；上一次的摘要会作为迭代输入。
  - 压缩结果作为一条 `CompactionEntry` 追加到会话，原始条目保留在会话树里。
  - 多轮运行中，每批工具执行完会检查一次；上下文溢出报错时压缩并重试一次。
  - 摘要请求不写 prompt cache，因为这种一次性提示词不会被复用。

### 2.4 会话

- 会话是 JSONL 文件，条目通过 `id` / `parentId` 组成一棵**树**。从较早的条目继续对话就在同一个文件里长出新分支；当前条目所在的分支就是发给模型的历史。
- 切换分支时可以对被放弃的分支做"分支摘要"，保留其中的信息。
- 格式有版本号，加载时自动迁移（v1 线性 → v2 树 → v3）。

### 2.5 扩展机制

- **扩展**是 TypeScript 模块，导出一个工厂函数 `export default function (pi: ExtensionAPI)`，在同一进程内运行，用 jiti 直接加载 TS，不需要编译。来源有三处：用户目录、项目目录、命令行 `-e`。
- **注册点**：`pi.on()` 事件、`registerTool`、`registerCommand`、`registerShortcut`、`registerFlag`、`registerProvider`（模型厂商）、`registerMcpServer`、`registerVirtualModel`（按请求路由模型）、渲染器与 TUI 组件。扩展之间通过 `pi.events` 通信。
- **事件**按生命周期分组：

  | 阶段 | 事件 |
  | --- | --- |
  | 输入 | `input`、`before_agent_start` |
  | run | `agent_start`、`agent_end`、`agent_before_settle`、`agent_settled` |
  | turn | `turn_start`、`context`、`context_with_system`、`turn_end` |
  | 厂商调用 | `before_provider_headers`、`before_provider_request`、`after_provider_response`、`provider_stream_event` |
  | 消息 | `message_start`、`message_update`、`message_end` |
  | 工具 | `tool_execution_start`、`tool_call`、`tool_execution_update`、`tool_result`、`tool_execution_end` |
  | 会话 | `project_trust`、`session_start`、`session_shutdown`、`session_before_switch`、`session_before_fork`、`resources_discover` |

- **处理器的组合方式**由事件决定：只通知、原地修改、链式变换（后一个处理器拿到前一个的返回值）、闸门（阻止后续流程）、可操作边界（`turn_end` / `agent_before_settle` 可以追加条目或要求再继续一次）、最后覆盖。
- **扩展的状态放在哪里**，有明确约定：

  | 状态 | 存放位置 |
  | --- | --- |
  | 跟随当前分支的工具状态 | 工具结果的 `details` |
  | 要持久化但不进模型上下文 | `pi.appendEntry()` |
  | 要持久化且发给模型 | `pi.sendMessage()` |
  | 跨会话的数据 | 外部存储 |

- **生命周期约定**：工厂函数里不能启动进程、定时器、监听器，因为有些调用只加载扩展而不启动会话。长期资源在 `session_start` 中创建，在幂等的 `session_shutdown` 中释放。reload 会整体替换扩展运行时。
- **工具的可见性**（`exposure`）：`direct`（声明给模型）、`model-only`、`codemode`（只能通过 codemode 脚本调用）、`deferred`（需要通过 `tool_search` 找到并激活）、`hidden`。工具不能注销，只能重新注册为 hidden。这套设计用于工具数量很多时控制上下文。
- **分发**：Pi package 把扩展、技能、提示词模板、主题打成一个 npm 包或 git 仓库，用 `pi install` 安装。项目级资源要先通过"项目信任"才加载。
- 没有内置权限系统，需要隔离时建议放进容器或沙箱。

### 2.6 pi-durable：持久化 harness（实验）

这是 Pi 新做的另一套 harness，目标是长期运行、能从崩溃中恢复，和本项目的常驻服务需求高度相关：

- **先提交、再展示**：所有改动都通过一条原子提交线写入存储，提交之前什么都不展示。进程在一轮中间死掉，重新打开存储就能从断点继续。
- **概念**：
  - **Conversation**：不可变条目组成的转录。
  - **Entry**：用户消息、模型回复、工具结果、系统提示词变更、reset、自定义类型。
  - **Document**：和转录存在一起的类型化 JSON 状态，在同一个提交里修改。内置文档有 agent 配置、收件箱、运行中状态、用量。
  - **Task**：持久状态机，每一步保存检查点，重启后从最近的检查点继续。
- **任务**：
  - **睡眠**：任务可以"睡到某个时间点"。睡眠状态持久保存，期间不占用调用，重启后到点继续。可以直接用来实现提醒。
  - **子任务**：父任务提交 `waiting` 状态等待子任务，可选全部完成或一个失败就全部中止；中止从下往上进行。
  - **崩溃保护**：反复让进程崩溃的任务达到重启上限（默认 5 次）后被搁置，等待人工重试。
- **收件箱**：会话忙时，新输入可以选择 steer、follow-up 或 reject。还有一种 `write`，只追加一条记录、不触发模型，可以用来记录"用户打开了某个文件"这类事实。
- **扩展**：一个命名的扩展可以带工具、提示词片段、钩子、装饰器、任务。注册表只存在于进程内；会话里只存扩展名称，每次使用时按名称解析。
- **压缩在后台执行**：低于阈值一段距离就在后台开始压缩，对话继续进行，摘要在空闲时或下一个轮次边界放入；只有达到阻塞阈值时，下一个请求才会等待压缩完成。
- **存储后端**：内存、IndexedDB、SQLite（推荐长期运行使用）、JSONL、Cloudflare Durable Object。同一时间一个进程独占一份存储。

### 2.7 初步启示

- **内核可以非常小**：`pi-agent-core` 只有约 2500 行，只管 loop、队列和少量钩子；会话、压缩、扩展加载都放在上层。这和本项目"内核极小"的设想一致，而且证明了可行。
- **steer / follow-up 两条队列**是处理"agent 工作时用户又说话"的成熟方案，本项目单一会话、多来源事件的场景也需要。
- **扩展的状态归属表**（跟随分支、只持久化、持久化且发给模型、跨会话）值得直接借鉴，可以避免插件随意改上下文。
- **pi-durable 的任务模型**几乎就是本项目"承诺、提醒、后台处理"需要的东西：带检查点的持久状态机、持久睡眠、子任务、崩溃保护，再加上原子提交。它还是实验状态，但思路可以借鉴到 Python 实现中。
- **只追加以命中缓存**：Pi 和 dsh 都让提示词和工具变更以追加方式进入历史。这再次说明"每轮重新组装"需要和 prompt cache 一起权衡。

## 3. Hermes Agent

仓库：`NousResearch/hermes-agent`，2026-10-27 的提交。Python 为主（另有 TypeScript 写的 TUI、桌面和网页端），约 2.1 万个文件，其中测试 7400 多个。定位是"会自我改进的个人 agent"：跨会话学习（记忆和技能），同一个 agent 内核同时服务 CLI、约 20 个 IM 平台的网关、TUI、桌面应用。

> 本节为初步阅读，主要依据各目录的 AGENTS.md 和用户文档，没有逐行读代码。

### 3.1 两条核心不变量

Hermes 的设计说明把这两条列为审查每一处改动的标准：

1. **按会话的 prompt cache 神圣不可侵犯**。会话中途不改过去的上下文、不换工具集、不重新加载记忆、不重建系统提示词；唯一允许的例外是压缩。要在会话中途注入内容，只能放进用户消息或工具结果。改变系统提示词状态的斜杠命令推迟到下个会话生效，可以用 `--now` 强制立即生效。
2. **内核是"窄腰"，能力放在边缘**。所有工具每次请求都会发给模型，所以新增核心工具的门槛很高。贡献规范定了一个"占用面阶梯"，从上往下选第一个能解决问题的：
   1. 扩展已有代码
   2. CLI 命令加技能
   3. 按服务开关启用的工具
   4. 插件
   5. 插件目录里的 MCP 服务器
   6. 新的核心工具

另外明确拒绝"没有具体使用方的 hook"，和本项目"扩展点只在至少有两个真实插件需要时才开放"一致。

### 3.2 分层结构

- `run_agent.py`：`AIAgent` 门面，由多个 mixin 组成。
- `agent/`：轮次循环、提示词组装、压缩、记忆管理、模型解析。
- `tools/`、`toolsets.py`、`model_tools.py`：工具注册表与工具集。
- `gateway/`：消息网关，平台适配器在 `gateway/platforms/`。
- `cron/`：定时任务存储与调度。
- `hermes_state*.py`：会话数据库（SQLite，带 FTS5 全文检索），拆成六十多个兄弟模块。
- `plugins/`：记忆、上下文引擎、模型厂商、委派、图像生成、平台适配等。
- `skills/`：内置技能。

代码组织采用"门面 + 兄弟模块"：每个曾经的超大文件拆成门面和按主题划分的兄弟模块，并由脚本限制单文件不超过 2000 行、单函数不超过 300 行。

### 3.3 Agent loop

- 完全同步的 while 循环：调用模型 → 有工具调用就执行并追加结果 → 没有就返回。有迭代上限（默认 500 次，子 agent 共享）、预算、中断检查。
- 循环的每个阶段一个文件（`turn_preflight`、`turn_api_call`、`turn_overflow`、`turn_context_compaction`、`turn_recovery` 等），每个约 600 行。
- **消息流不变量**：
  - 严格的角色交替，不允许连续两条同角色消息，不在循环中途插入合成的用户消息。
  - 唯一例外是 `/steer`：作为一条独立的用户消息放在工具结果之后。
  - 定时任务（cron）的投递放在独立的会话里，也是为了不破坏这条规则。
- 记忆、todo 等"agent 级工具"在工具执行器里被拦截，通过一张注册表分发，不写 if 链。

### 3.4 上下文、记忆与压缩

- **内置记忆**是两个有字数上限的文件：

  | 文件 | 用途 | 上限 |
  | --- | --- | --- |
  | `MEMORY.md` | agent 自己的笔记（环境、约定、学到的东西） | 2200 字符 |
  | `USER.md` | 用户画像（偏好、沟通风格） | 1375 字符 |

  agent 通过 `memory` 工具增删改条目；写满时工具返回错误，让 agent 自己合并或删除条目，不自动压缩。
- **冻结快照**：会话开始时把记忆渲染进系统提示词，整个会话期间不变；会话中途的修改立即写盘，但下个会话才进入提示词。这也是为了保护 prompt cache。
- **记忆 provider 插件**（`MemoryProvider` 抽象基类）：钩子有 `system_prompt_block`、`prefetch` / `queue_prefetch`（按当前消息预取相关记忆）、`sync_turn`（每轮同步）、`on_session_end`（会话结束时提取记忆）、`on_pre_compress`（压缩前抢救信息）、`on_delegation`、`on_memory_write` 等。Honcho、mem0、supermemory 等外部记忆服务都通过它接入，现在都已移出主仓库，改放插件目录。
- **跨会话回忆**：会话存在 SQLite 里，用 FTS5 全文检索；检索结果由辅助模型摘要后返回。
- **压缩**有两层：
  - 网关层按 85% 阈值整理会话。
  - agent 层的 `ContextCompressor` 默认 50% 触发：先修剪旧的工具结果（不调用模型），再选边界，最后用辅助模型生成结构化摘要。
  - 压缩是"唯一被允许的缓存破坏"。
- **辅助模型**：压缩、标题、检索摘要、视觉、技能整理等"旁路"任务各自可以配置独立的模型。
- 定时任务的会话默认跳过记忆（`skip_memory=True`）。

### 3.5 插件机制

插件分几类，各有自己的发现机制：

| 类型 | 位置 | 机制 |
| --- | --- | --- |
| 通用插件 | 内置目录、`~/.hermes/plugins/`、项目目录、pip entry points | `register(ctx)` 注册 hook（`pre_tool_call`、`post_tool_call`、`pre_llm_call`、`post_llm_call`、`on_session_start`、`on_session_end`）、工具、CLI 子命令、语言包；后加载的覆盖先加载的 |
| 记忆 provider | `plugins/memory/<name>/` 等 | 实现 `MemoryProvider` 抽象基类，按配置名称激活；内置优先 |
| 模型厂商 | `plugins/model-providers/<name>/` | 首次使用时懒加载，后注册的覆盖先注册的 |
| 上下文引擎、委派、图像生成等 | `plugins/<kind>/` | 抽象基类 + 编排器 + 每个插件一个目录 |
| 平台适配器 | `plugins/platforms/<name>/` | 由网关加载（飞书是标准示例之一） |

- **硬规则：插件绝不修改核心文件**。插件需要框架没有的能力时，就扩展通用的插件接口（新 hook、新 ctx 方法），不在核心里写针对某个插件的逻辑。
- **插件目录**（plugin-catalog）是外部插件唯一的发现渠道：每个条目一个 YAML，必须固定到 40 位提交 SHA，人工审核合并，CI 会拉取并校验；另有吊销名单。
- 第三方产品的集成（可观测性、SaaS 连接器等）不进主仓库，作为独立插件仓库发布。

### 3.6 后台与主动能力

- **cron**：
  - agent 通过 `cronjob` 工具创建任务，用户也可以用命令创建。时间格式支持时长、"every" 短语、五字段 cron、ISO 时间点。
  - 每个任务可指定技能、模型、工作目录，还可以带一个预先运行的数据采集脚本（脚本输出注入提示词），或者把 A 任务的输出接到 B 任务的提示词里。结果可投递到多个平台。
  - 健壮性约束：
    - 先推进下一次运行时间再执行（崩溃时至多执行一次），配合执行台账防止重复触发。
    - 错过的执行有补发窗口。
    - 空闲 600 秒的任务会被强制中断。
    - 每个 home 一把 tick 锁。
- **heartbeat**：给当前会话设一条周期指令（如每 10 分钟检查一次部署），只在会话空闲且到点时作为普通用户消息进入，共享会话上下文和 prompt cache。和 cron 的区别是：heartbeat 需要当前对话的上下文，cron 每次在独立的新会话里运行。
- **网关**：一个进程同时服务多个平台；一个进程还可以服务多个 profile（相互隔离的 home），密钥和路径都按 profile 作用域绑定。
- **其他**：子 agent 委派、kanban 多 agent 工作队列、goals、技能自动沉淀与整理（curator）、6 种终端后端（本地、Docker、SSH、Modal、Daytona、Vercel Sandbox）。

### 3.7 初步启示

- **同为 Python 的个人助手**，但 Hermes 的复杂度主要来自多平台、多 profile、多终端后端和研究用途。本项目 MVP 只需要其中很小一部分。
- **"prompt cache 神圣"是第三次出现**：dsh、Pi、Hermes 都把稳定前缀当作硬约束，Hermes 最极端（系统提示词整个会话逐字节不变，记忆用冻结快照）。本项目"每轮按当前状态重新组装"的决定必须正视这一点，至少要区分"稳定前缀"和"每轮变化的部分"。
- **记忆 provider 的钩子集合**（预取、每轮同步、会话结束提取、压缩前抢救）可以作为本项目记忆插件接口的参考清单。
- **heartbeat 与 cron 的区分**（需要对话上下文 / 独立会话）对本项目"多层主动提醒"有直接参考价值。不过本项目是单一会话，需要重新考虑主动消息如何进入那唯一一个会话。
- **"没有使用方的 hook 一律拒绝""插件不改核心"**这两条治理规则，和本项目已有的约束一致，可以写进插件规范。

## 4. Letta（只看特色）

`letta-ai/letta` 主仓库已改为指引页，原来的 Python 服务端（MemGPT 时代的 V1 API）已归档。当前源码在 `letta-ai/letta-code`（TypeScript，2026-10-26 的提交）。定位是"更像人而不是工具的 agent"：有记忆、身份和长期经验，能改写自己的记忆、技能、提示词，甚至通过 mod 改写 harness。

- **记忆块**：把系统提示词拆成多个可被 agent 改写的块。agent 通过改写这些块"学习"，也就是在系统提示词层面学习。
- **MemFS**：agent 的全部上下文（记忆块、技能、提示词）都是文件，用 git 管理，可以同步到用户自己的 GitHub 仓库。改动可追溯、可回滚。
- **反思与做梦**（`/sleeptime`、`/reflect`、`/dream`）：
  - 由一个后台的 reflection 子 agent 阅读最近的对话记录，按优先级提取：错误与纠正 → 偏好与模式 → 值得保留的新事实 → 与现有记忆的矛盾 → 可复用的流程（沉淀为技能）。
  - 跳过一次性的任务状态，把相对日期改写为绝对日期；没有值得保留的内容就不提交。
  - 跨多个会话反思时，优先采用多个会话都支持的模式，有矛盾时以最新证据为准。
- **记忆改动的落地方式**：reflection agent 在独立的 git worktree 上修改记忆，由 harness 负责合并回主分支。
  - 合并结果分为干净合并、无改动、需重试、合并失败待处理四种。
  - reflection agent 从不自己解决冲突。
  - 只有记忆真正合并后，对应的对话记录才标记为"已消费"。
  - 触发计数按完成的 assistant 步骤累计。
- **其他**：跨所有消息和所有 agent 的搜索；heartbeat 与 cron；Slack、Telegram、Discord 等渠道；agent 之间可以互相作为子 agent 调用。

启示：

- 后台反思子 agent 加"在副本上改、由 harness 合并、合并成功才标记已消费"的流程，是"记忆整理放在后台、不阻塞回复"的一个完整做法。
- 记忆用文件加 git 管理，天然满足"可追溯、可回滚"。
- 反思提示词里的优先级和过滤规则可以直接参考。

## 5. Graphiti（只看特色）

`getzep/graphiti`（Python，2026-10-26 的提交），Zep 的开源图谱构建框架。

- **时序上下文图**由三部分组成：
  - 实体（节点）：带随时间演化的摘要。
  - 事实（边）：三元组 实体 → 关系 → 实体。
  - episode：原始输入，所有派生事实都能追溯到它。
- **双时间轴**：每条事实边都带两类时间：
  - 世界时间：`valid_at`（事实何时开始为真）、`invalid_at`（何时不再为真）。
  - 系统时间：`created_at`、`expired_at`（系统何时记录、何时作废）。
  - 有两条细节规则：未来的计划或承诺，`valid_at` 取 episode 的参考时间而不是计划发生的时间；软截止期限不算 `invalid_at`。
- **矛盾处理**：新事实写入时，先检索候选的旧事实，由模型判断哪些旧事实被新事实**明确否定**。被否定的旧事实设置失效时间，而不是删除。提示词反复强调：不能因为"更新"或"时间更晚"就推断矛盾，必须两者不可能同时为真。
- **增量更新**：新数据进来只更新局部，不重算整张图；支持查询"某个时间点什么是真的"。
- **混合检索**：语义向量 + BM25 关键词 + 图遍历，再重排。
- **代价**：需要图数据库（Neo4j、FalkorDB、Kuzu、Amazon Neptune），每个 episode 要调用多次模型（抽取实体、抽取边、去重、判断矛盾、更新摘要）。

启示：

- "episode 是事实来源、派生事实可追溯"与本项目"原始日志是唯一事实来源"完全一致。
- 双时间轴和"失效而非删除"直接对应 M2"写入前与已有事实比对"的需求。
- 判断矛盾的提示词规则（必须明确否定才算矛盾）值得借鉴。
- 引入图数据库对 MVP 偏重，可以只借鉴数据模型，用关系库实现。

## 6. mem0（只看特色）

`mem0ai/mem0`（Python 与 TypeScript，2026-10-26 的提交）。

- **2026 年 4 月改了算法，方向值得注意**：放弃以前"抽取后再做 ADD / UPDATE / DELETE 决策"的两阶段写入，改成**一次调用、只追加（ADD-only）**，记忆只累积、不覆盖。官方基准上 LoCoMo 从 71.4 升到 92.5，LongMemEval 从 67.8 升到 94.4，抽取延迟约减半（数据来自其托管平台，开源版"方向上相近"）。
- **写入流程**（`Memory._add_to_vector_store`）：
  1. 取本会话最近 10 条消息作为上下文。
  2. 按新消息检索最多 10 条相关的已有记忆；把记忆 UUID 映射成 0、1、2 这样的小整数，防止模型编造 ID。
  3. 一次模型调用，输出要新增的记忆。抽取提示词要求同时从用户和助手消息中抽取，助手给出的建议、计划也算记忆；参考"最近已抽取的记忆"去重。
- **难题移到检索阶段**：语义、BM25 关键词、实体匹配三路并行打分再融合；"时间推理"在检索时选出与问题时间匹配的那条记录（当前状态、过去事件、未来计划）。记忆可以带过期时间，过期的默认不返回。
- 图记忆已从开源版移除，只在其托管平台提供。

启示：

- mem0 和 Graphiti 代表两条路线：**写入时消解矛盾**（Graphiti 让旧事实失效）与**写入时只追加、检索时按时间挑选**（mem0）。后者写入简单、便宜、不丢信息，与本项目"只追加"的原则更契合，但检索和排序需要做好。M2 需要在两者间取舍，或者组合使用。
- 把 ID 映射成小整数防止编造、显式传入"最近已抽取的记忆"去重，这些工程细节可以直接用。

## 7. OpenHands（只看特色）

`OpenHands/OpenHands` 主仓库已变成前端控制台 Agent Canvas，可以接入 OpenHands、Claude Code、Codex、Pi 等任何支持 ACP 的 agent。agent 内核在 `OpenHands/software-agent-sdk`（Python，2026-10-27 的提交）。

- **只追加的事件日志是核心状态**：几乎所有非环境的状态变化都是事件。可以转换成模型消息的事件继承 `LLMConvertibleEvent`。持久化方式是一个 `base_state.json` 加 `events/` 目录（每个事件一个 JSON 文件，按序号命名）。
- **压缩用"墓碑"事件**：只追加的日志无法删除，于是压缩时追加一条 `Condensation` 事件，记录"哪些事件被遗忘、在哪里插入摘要"，类似 Cassandra、Kafka 里的墓碑。再由 `View` 类把所有压缩事件应用到事件流上，得出"当前与模型相关的事件"。View 还会检查消息结构，保证压缩不会破坏模型要求的格式（例如工具调用必须与结果成对）。
- **压缩策略**：默认把前一半事件替换成一条摘要，后一半原样保留；摘要会在后续压缩中再被摘要。
  - 理由是定期压缩虽然会破坏 prompt cache，但能让每次重建缓存的成本保持很低。
  - 软触发（达到大小上限）时，如果当前无法安全压缩就跳过，下一步再试。
  - 硬触发（上下文溢出）时，无法平衡压缩就整体摘要重置。
- **兼容性纪律**：旧事件必须永远能加载；删除字段时要加永久的弃用处理器；持久化设置有 `schema_version` 和逐版本迁移函数，并用"黄金样本"测试。
- **其他**：技能按触发方式区分（始终生效、关键词触发、任务触发）；插件格式兼容 Claude Code 的插件结构（技能、hooks、MCP 配置、子 agent、命令打包在一起）；Agent Server 用 REST / WebSocket 把 agent 运行在本地、Docker 或远程。

启示：

- "事件日志 + 墓碑式压缩事件 + View 投影"是"原始日志只追加、上下文由它派生"的一个干净实现，而且是 Python 写的，可以直接参考。
- "每轮从日志投影出模型视图"与本项目"每轮按当前状态重新组装"的思路一致。OpenHands 明确接受了压缩对缓存的破坏，用"定期压缩、保持低重建成本"来平衡。这给"重新组装与 prompt cache"之争提供了另一个答案。
- 旧事件永远可加载的纪律和本项目"不维护向后兼容"的原则相冲突。但原始日志是唯一事实来源，日志格式本身大概率需要例外处理，这需要你来决定。

## 8. Codex CLI（只看特色）

`openai/codex`（Rust，2026-10-27 的提交），`codex-rs/` 下有一百多个 crate。

- **app-server 协议**：用 JSON-RPC 把 agent 运行时和各种前端（终端 UI、IDE 插件、桌面）分开。基本原语是：
  - **thread**：会话。支持 start、fork、compact、archive、rollback、goal 等。
  - **turn**：一轮。
  - **item**：一轮中的条目，如消息、命令执行、文件修改、工具调用、推理摘要。条目有 started、delta、completed 事件，审批请求也是 item 事件。

  这是一套成熟的"运行时常驻、客户端只是渠道"的协议设计。
- **线程存储边界**（`codex-thread-store`）：`append_items` 是唯一的原始历史追加接口，不从内容推断元数据；元数据只能通过 `update_thread_metadata` 写入。历史和元数据分两个通道。
- **后台记忆流水线**（`codex-rs/memories`）：根会话启动时在后台异步运行，分两个阶段：
  - **阶段 1（逐会话抽取）**：从状态库中领取最近的、已空闲足够久的会话记录（避免总结仍在进行的会话），并行交给模型，产出详细记忆和简短摘要，脱敏后写回状态库。每个任务先租约领取，防止重复处理；失败按退避重试。
  - **阶段 2（全局整理）**：拿到全局锁后，按使用次数和最近使用时间挑出一批阶段 1 的产出，同步成文件（`raw_memories.md`、每个会话一份摘要）。记忆目录本身是一个 git 仓库，生成与上次成功时的差异文件；有变化时启动一个专门的整理子 agent（无审批、无网络、只能本地写），由它更新 `MEMORY.md`、`memory_summary.md` 和技能；成功后重置 git 基线。
  - 长期不用的记忆（超过 `max_unused_days`）不再入选。
- **hooks**：`SessionStart`、`SessionEnd`、`UserPromptSubmit`、`PreModelRequest`、`PostModelRequest`、`PreToolUse`、`PostToolUse`、`PermissionRequest`、`PreCompact`、`PostCompact`、`SubagentStart`、`SubagentStop`、`Stop`、`Interrupt`。
- **执行策略**（execpolicy）：用 Starlark 写命令前缀规则，决定 allow、prompt 还是 forbidden；规则自带正反例，加载时校验，相当于规则的单元测试。
- **沙箱**：macOS Seatbelt、Linux bwrap / Landlock、Windows 专门的沙箱实现。

启示：

- thread / turn / item 三层原语和 item 的生命周期事件，可以作为本项目渠道协议（飞书、网页、以后的 App）的参考。
- 两阶段记忆流水线（逐会话抽取 → 全局整理，租约、退避、空闲判定、git 差异驱动）是"后台记忆提取与整理"最完整的工程样本之一，对 M2 很有参考价值。
- Letta 和 Codex 都用 git 管理记忆文件、都用子 agent 做整理，属于两家独立得出的同一结论。

---

## 横向对比与对本项目的启示（初稿）

### 横向对比

| 维度 | dsh | Pi | Hermes | OpenHands SDK |
| --- | --- | --- | --- | --- |
| 语言 | TypeScript | TypeScript | Python | Python |
| 内核 | 没有特权内核，连 loop 都是插件 | `pi-agent-core` 约 2500 行，只有 loop、队列和钩子 | 同步 while 循环，按阶段拆文件 | Agent + Conversation + 事件日志 |
| 插件模型 | Cordis：服务容器、依赖注入、可撤销副作用、5 种事件分发模式 | 工厂函数 + `pi.on()` 事件 + 各种 register | 多类插件各有发现机制：hook + 抽象基类 + 编排器 | 兼容 Claude Code 插件结构 |
| 日志 | 只追加 JSONL，"模型可见即已记录" | JSONL 会话树，可分支 | SQLite + FTS5 | 每个事件一个 JSON 文件 |
| 上下文 | 增量追加，系统提示词作为历史节点 | 增量追加，尽量保持缓存前缀 | 系统提示词整个会话逐字节不变 | 每步从日志投影出 View |
| 压缩 | 先修剪工具结果，再摘要 | 摘要条目 + 保留原始条目 | 先修剪工具结果，再结构化摘要 | 墓碑事件，前一半换成摘要 |
| 主动能力 | schedule、goal、jobs、webhook | pi-durable：持久睡眠任务 | cron + heartbeat | 自动化（定时、webhook） |

### 初步结论（待讨论）

1. **"每轮重新组装"和 prompt cache 的冲突是最需要先拍板的问题。** dsh、Pi、Hermes 都把稳定的缓存前缀当作硬约束；OpenHands 接受定期破坏缓存；本项目定的是每轮重新组装。一个可能的折中：上下文分成"稳定前缀"（人设、长期档案快照、工具定义，低频变化）和"动态尾部"（时间线、承诺、检索结果，每轮重组），再加上类似 Hermes 冻结快照或 OpenHands 定期压缩的策略。DeepSeek 的上下文缓存是按前缀命中的，这个问题对本项目同样成立。
2. **内核边界**：Pi 证明了"约 2500 行的 loop 内核 + 上层插件"可行；dsh 证明了"一切皆插件"可行但代价很高。本项目暂定的内核范围（事件总线、loop、上下文组装框架、原始日志、插件加载、配置）介于两者之间，看起来合理。
3. **插件机制**：
   - Python 没有 Cordis。可选方案有：Hermes 式的"hook + 抽象基类 + 编排器"（简单直接）；自己写一个很薄的"服务容器 + 可撤销注册 + 少量分发模式"（接近 Cordis）；或者 `pluggy`（只有 hook）。
   - 共同的好做法：插件依赖接口而不依赖实现；每个注册都可以撤销；状态归属有明确约定（参考 Pi 的四类状态表）；没有使用方的 hook 不开放。
4. **单一会话的输入排队**：所有来源（用户消息、定时提醒、后台任务完成、外部信号）进同一个收件箱，区分 steer 和 follow-up 两种插入方式（Pi、dsh、Hermes 都有）。主动消息如何进入唯一的会话而不破坏角色交替，参考 Hermes 的 heartbeat 规则。
5. **后台记忆**：Codex 的两阶段流水线和 Letta 的反思子 agent 是两个成熟样本；mem0 与 Graphiti 代表"只追加、检索时选"和"写入时让旧事实失效"两条路线。M2 设计时再细比。
6. **长生命周期任务**：pi-durable 的"带检查点的持久状态机 + 持久睡眠 + 原子提交"与 dsh schedule 的投递语义，是 M3"承诺"的两个参考。
