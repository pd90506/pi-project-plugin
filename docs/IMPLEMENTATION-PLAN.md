# pi-projects 实施计划：在 pi-coding-agent 中复刻 Claude Projects (BETA)

> 状态：设计稿（待评审） · 日期：2026-11-23
> 依据：参考实现 `/Users/panda/repo/dsh-project-plugin`（只读）源码通读 + pi-coding-agent 源码/文档
> （`earendil-works/pi` main @ `69a6993`，2026-11-22；`@earendil-works/pi-tui` / `pi-coding-agent` 均 1.6.0）。
>
> 硬约束（用户已确认）：
> 1. **不修改** `dsh-project-plugin`（只读参考）。
> 2. 交互**全部发生在终端内**：**不考虑**“外部本地 Web 视图桥接到 agent”的方案，也不引入浏览器面板。
> 3. 本文只做设计与计划，不在本线程实现插件。

---

## 0. 结论摘要（TL;DR）

1. **推荐 UI 方案 B：REPL 原生分层屏幕**（`ctx.ui.custom()` 屏幕 + 原生 dialog + 1 行状态条 + 全屏模式下的满屏 overlay 宿主）。
   不采用“接管 alternate screen、自建分栏布局”；不采用“浏览器/Web 视图”（用户否决）；tmux 集成只作为 M7 的可选增强。
2. **线程运行时推荐 `pi --mode rpc` 子进程**（每个活跃 thread 一个），由 coordinator 会话里的插件进程做 supervisor；
   进程隔离、可后台并行、审批走 RPC 的 `extension_ui_request` 子协议，且与 pi 自带 subagent 扩展同款思路。
3. **`project.json` 单写者**：只有 supervisor（插件）写项目文件；线程通过 **file inbox（JSONL 追加）** 上报
   status/decision/report，插件负责合并。避免跨进程写同一 JSON 的锁竞争与丢更新。
4. **M1 可以一个线程做完**：项目实体 + `/projects` 仪表盘 + 创建/打开 coordinator 会话 + instructions 注入 +
   状态条；**不含 thread**。M2 才引入“真正的线程”。
5. 三大风险：overlay/`custom()` UI 在 fullscreen 与 regular 两种宿主下的行为差异；RPC 子进程生命周期与
   审批管道；`switchSession` 触发的扩展重载与会话文件归属——三条都安排 M1/M2 内的小 spike 验证。需要用户
   拍板的问题收敛为 5 个（§6：UI 形态、打开 thread 的方式、项目数据放哪、M3 优先级、线程运行形态）。

---

## 1. 产品定义与范围

### 1.1 一句话定义

> **一个 project = 一个持续的协调对话（coordinator session）+ N 个并行 thread（各自一个 pi session）；
> instructions / memory / roots 一次设定、所有 thread 继承；一个 Overview 屏按“谁在等我”排序；
> thread 干完活自动向 coordinator 汇报。**

这是参考实现 `dsh-projects` 的产品定义（`dsh-project-plugin/README.md` 概念映射表），也是 Claude Projects
(BETA) 的官方定位："a project is one ongoing conversation where Claude coordinates a stream of related work for you."

### 1.2 概念映射（Claude Projects → pi 实现）

| Claude Projects | pi 实现 | 依据 |
|---|---|---|
| Project conversation（coordinator） | 一个 pi session（JSONL 会话文件），`cwd` = 项目首个 context folder | `SessionManager.create(cwd)`，`packages/coding-agent/src/core/session-manager.ts` |
| Threads（云端完整会话，可打开/续聊） | 独立 pi session 文件 + 运行期为 `pi --mode rpc` 子进程（非运行期只是一个会话文件） | `docs/rpc.md`、`docs/cli-integration.md`、`examples/extensions/subagent/index.ts` |
| Project instructions（≤16k） | `project.json.instructions`（canonical）+ 由插件在 `before_agent_start` 注入 system prompt section | `docs/extensions.md` §before_agent_start |
| Project memory（MEMORY.md 索引 + 文件） | `<projectDir>/MEMORY.md` + `memory/<topic>.md`，工具 `memory_*`；注入方式同 instructions | 参考实现的 dsh-memory 存储格式（见 §2.6） |
| Environment（repos / folders） | `project.roots[]`（context folders）；一个 thread 的 cwd = 其中一个 root；无沙箱写根概念，靠 OS 权限 | `docs/sessions.md`（session 记录 cwd） |
| Library（上传 + thread 产出） | `<projectDir>/library/` + `library/uploads/`；命令 `/project upload <path>` | 参考实现 `library.js` |
| Overview 分组 | 8 个组（Waiting / Failed / Review / Working / Landing / Idle / Reminder / Resolved），注意力优先排序 | 参考 `client.js` `GROUP_ORDER` |
| 决策卡 / Suggested threads / Setup recommendations | 终端 dialog + 卡片屏 + 结构化 tool 输出 | 参考 §2.4 工具面 |
| Pause / Archive / Delete | `project.status ∈ active/paused/archived` + 删目录（外部 git 产物不动） | 参考 `updateProject` / `deleteProject` |

### 1.3 范围裁决（core / nice / skip）

沿用参考实现的功能盘点结论（详见 §2.1 的“移植裁决”列），这里只列**边界**：

- **core（M1–M5 必须）**：项目实体与存储、coordinator 会话、thread 生命周期与状态机、Overview 注意力排序、
  instructions / memory / roots / library、thread↔coordinator 消息、决策与审批、汇报合并、等待派生。
- **core for coding（M6）**：每 thread 的 git worktree 隔离、PR 链接/轮询/CI 失败唤醒、merge guard、
  `gh repo create` 发布。这一档是“编码 agent 场景”的实义核心，但对非编码用户可整体跳过。
- **nice（M7 或按需）**：scheduled tasks / reminders、usage 统计、auto thread title、memory tidy / recall、
  thread 间直连消息、通知模式（default/turns/off）、`--plain` 输出。
- **skip**：React 面板、侧边栏 slot fork、拖拽上传、网页通知、云端 environment / GitHub App / SSO、
  plan 限额、`dshp://` 深链、DSH teamwork / permission preset 等 DSH 专属 UI 概念。

---

## 2. 参考实现功能盘点与移植裁决

> 以下行号均指 `/Users/panda/repo/dsh-project-plugin` 当前源码（Host：`dsh-projects/index.js` 5 772 行、
> `schedule.js` 763 行、`github.js` / `library.js` 各 172 行；Client：`client.js` 5 727 行；
> 记忆：`dsh-memory/index.js` 1 013 行）。注意：**README/design.md 落后于源码**（源码 26 个工具、7 个状态、
> 已含 PR 轮询），移植时以源码为准。

### 2.1 功能清单（按组，仅保留需要决策的项）

**项目生命周期（core）**

| 功能 | 终端移植裁决 | 参考位置 |
|---|---|---|
| 建项目（name/goal/folders/instructions/icon/pin） | core：表单改为一屏 wizard（name→goal→root→instructions 可跳过） | `index.js:1463-1515`，`client.js:2187-2322` |
| 仪表盘列表（pinned/others/archived、搜索、排序） | core：Dashboard 屏，卡片→单行 | `client.js:2323-2513` |
| 打开项目 = 打开 coordinator 会话 | core：`switchSession()`（pi 原生） | `client.js:2393-2404` |
| 改名/goal/icon/pin、Pause/Resume、Archive、Delete | core：Settings 屏 + 命令 | `index.js:1600-1690`，`1658-1677`，`4516-4532` |
| 项目状态徽标、行置灰 | nice：状态词 + 灰度 | `client.js:4367-4368` |

**coordinator（core）**

| 功能 | 移植裁决 | 参考位置 |
|---|---|---|
| 自动创建 coordinator 会话（cwd=项目目录） | core：`SessionManager.create` + `switchSession` | `index.js:1499-1512` |
| 首轮 kickoff（有 goal 则当作用户第一条需求；无 goal 只打招呼） | core：kickoff 作为 user message 写入新会话 | `index.js:1291-1323` |
| Setup recommendations（首轮建议卡） | nice（M5） | `index.js:5484-5543` |
| coordinator 只读（禁止 shell/write/subagent） | core：`tool_call` gate 拒绝并提示“开 thread” | `index.js:78-83`，`4955-4966` |
| coordinator 状态提醒（open threads 摘要） | core：每轮注入 display:false message | `index.js:961-1004` |
| coordinator 模型/思考强度 | nice：会话级 `/model` 天然可用 | `index.js:1644-1650` |

**threads（core）**

| 功能 | 移植裁决 | 参考位置 |
|---|---|---|
| 开 thread（title、自包含 brief、cwd、引用/附件） | core：`projects_thread_start` → 建会话 + spawn 子进程 + 投递任务 | `index.js:3129-3199` |
| thread 自报状态（working/waiting/idle/resolved…） | core：子进程内 tool + inbox 上报 | `index.js:5182-5203` |
| resolve / reopen、auto-resolve（7 天） | core | `index.js:3615-3665`，`4850-4882` |
| 每 thread 独立 worktree | core for coding（M6）：`git worktree` via `pi.exec` | `index.js:1771-2218` |
| thread 转录读取（分页、无工具调用） | core：`projects_thread_read` 走 RPC `get_entries` 或直接读 JSONL | `index.js:5343-5364` |
| thread 标题自动生成 | nice | `index.js:3048-3127` |

**Overview / 状态（core）**

- 分组顺序：`waiting → failed → review → working → landing → idle → reminder → resolved`（`client.js:841-843`）。
- 行内容：状态 glyph → 标题 → running/current 徽标 → PR chips → reminder 标记 → 右对齐相对时间 → note 第二行
  （`client.js:2547-2570`，相对时间 `1918-1941`）。
- **“Waiting on you” 是派生量，不是声明量**：待审批（permission/ask_user）、未答决策卡、显式 `waiting`
  三类来源（`index.js:4669-4701`，`4585-4591`）。终端的等价物见 §4.8。
- 搜索 / 隐藏 resolved / 分组折叠计数（`client.js:2572-2605`，`4402-4425`）。

**memory / instructions / library（core）**

- 记忆索引格式（`MEMORY.md`：标题 + 生成注释 + 每行 `- [Title](memory/x.md) — [type] 描述 (日期)`）；
  条目 4 KB、索引提示内 200 行 / 25 KB 截断、写协议带 revision、删除进 `memory-trash/`
  （`dsh-memory/index.js:280-294`，`150-155`，`311-313`，`461-469`，`490-509`）。
- instructions 双通道：canonical `project.json.instructions` + 每次装配的 prompt section；参考实现另镜像
  `AGENTS.md`（`index.js:320-330`，`881-884`）。pi 侧见 §3.3。
- library：`library/`（产出）+ `library/uploads/`（用户输入，只读）；上限 1000 文件 / 单文件 480 MB
  （`library.js:22-23`）。终端里“上传”= 复制路径，等价语义保留。

**决策与建议（core / M5）**：决策卡 = 1 问 2–4 选项 + 每项“后果” + 推荐 + 理由，用户答完回写并视为
“用户本人选择”，≈10 个上限（`index.js:3795-3797`，`3843-3893`，`3909-3923`）；Suggested threads /
Setup recommendations 先提案后执行，`apply` 支持 `start: "all" | indices` 与 `skip`（`4177-4225`）。
终端映射 = 编号化 dialog/overlay 屏 + 可勾选列表（§4.7）。

**汇报与通知（core → M3）**：per-thread 覆盖 + 静默窗 `reportBatchMs=15s`，urgent（waiting/failed）
不延迟，away 时例行消息攒 digest（`3245-3271`，`3361-3433`）；终端等价物 = bell / OSC 9/777 / 标题栏 /
footer 徽标 + `/project inbox`，保留 default/turns/off 三档（§4.12）。

### 2.2 数据模型（移植后的 `project.json` 草案）

```jsonc
{
  "id": "pi-project-plugin",              // slug，目录名
  "name": "pi-project-plugin",
  "goal": "在 pi 里复刻 Claude Projects",
  "status": "active",                     // active | paused | archived
  "pinned": true,
  "createdAt": "2026-11-23T07:00:00.000Z",
  "updatedAt": "2026-11-23T07:30:00.000Z",
  "instructions": "……",                   // ≤ 16000 chars（可配置）
  "roots": ["/Users/panda/repo/pi-project-plugin"],
  "coordinator": {                        // 替代参考的 coordinatorSessionId
    "sessionFile": "/Users/panda/.pi/agent/sessions/…/2026-…_uuid.jsonl",
    "createdAt": "…"
  },
  "threads": [{                           // 新→旧
    "sessionFile": "…jsonl",              // 身份 = 会话文件（不是进程）
    "title": "升级 vitest 到 v2",
    "status": "working",                  // 7 值 + UI 派生 reminder
    "note": "正在跑 pnpm -r test",
    "cwd": "/Users/panda/repo/pi-project-plugin",
    "model": { "provider": "anthropic", "model": "claude-sonnet-4-5" },
    "createdAt": "…", "updatedAt": "…", "lastActivityAt": "…", "error": null,
    "prs": [], "worktree": null,          // M6
    "reminder": null,                     // M7
    "unreadReport": false
  }],
  "decisions": [], "suggestions": [], "alerts": [],   // 同参考 schema（dec_/sug_ 前缀）
  "defaults": {
    "threadModel": null,                  // null = 跟随当前 pi 会话设置
    "thinkingLevel": null,
    "maxThreads": 4,
    "autoResolveDays": 7,
    "notifications": "default"            // default | turns | off
  },
  "paths": {                              // 相对 projectDir
    "memory": "MEMORY.md", "memoryDir": "memory",
    "library": "library", "uploads": "library/uploads",
    "inbox": "inbox",                     // 线程→supervisor 的命令队列
    "lease": "owner.lock", "uiState": "ui-state.json"
  }
}
```

存储布局（`dataDir` 默认 `$PI_AGENT_DIR/projects`，即 `~/.pi/agent/projects`）：

```
~/.pi/agent/projects/<projectId>/
  project.json            # canonical 元数据（单写者：supervisor）
  AGENTS.md               # instructions 镜像（人类可读 + 手动在项目目录开会话时原生注入）
  MEMORY.md  memory/*.md  # 项目记忆
  library/  library/uploads/
  inbox/<threadKey>.jsonl # 线程→supervisor 追加写；threads/<key>.log = 子进程 stderr/诊断
  owner.lock              # 多实例互斥：{pid, host, startedAt, heartbeatAt}
  ui-state.json           # 上次屏幕/光标/过滤条件（可选）
```

与参考实现的差异：**线程 id 从 sessionId 变为 sessionFile 路径**（pi 的会话身份就是文件）；项目数据默认放
全局 agent 目录而不是仓库内（待用户定，见 §6-Q3）；不再有 `schedule-claims/`（M7 自建 tick 时直接复用
`owner.lock` 的单实例假设）。

### 2.3 线程状态机

参考实现 7 个状态 + UI 伪组 `reminder`（`index.js:102-105`，`client.js:841-843`）。移植：

```
                       ┌────────── answer / new message ──────────┐
 (new) ──startThread──▶ working ──settle, no ask──▶ idle ──7d idle──▶ resolved
                        │  ▲                        │  ▲
      approval / decision│  │answer                 │  │reopen / message
                        ▼  │                        │  │
                     waiting ───────────────────────┘  │
 working/idle ──PR opened──▶ review ──all approved──▶ landing ──all merged──▶ resolved
 working/idle ──PR closed unmerged──▶ idle（或仍在等其它 PR 时 waiting）
 any ──bridge error / spawn fail──▶ failed ──user message──▶ working
 idle ──reminder set──▶ (UI 组 reminder；status 仍 idle)
```

派生规则（与参考一致，实现见 §4.8）：
`waiting = 有待审批 RPC 请求 ∨ 有 open decision 卡（asker=该 thread）∨ 显式自报`；
`review/landing/failed` 由 PR/子进程错误驱动；`resolved` 由工具/面板/自动扫描写入。

### 2.4 工具面（pi 版，名称与参考对齐）

coordinator 会话（插件主实例）注册：

| 工具 | 关键参数 | 语义 |
|---|---|---|
| `projects_list` | — | 项目列表 + 每组 thread 计数 + 运行中标记 |
| `projects_info` | `projectId?` | 详情：instructions、roots、threads、decisions、memory 索引、library |
| `projects_update` | `name? goal? instructions? pinned? status? roots?` | 元数据/生命周期（线程模型等“用户设置”不受理） |
| `projects_thread_start` | `title, prompt, cwd?, model?, thinkingLevel?, worktree?` | 建会话 + spawn 子进程 + 投递任务 |
| `projects_thread_status` | `status, note?, sessionFile?` | 改状态（thread 自报走 inbox，coordinator 直接改） |
| `projects_thread_resolve` | `resolved?, note?, sessionFile?` | resolve/reopen，带 blocker 列表 |
| `projects_thread_send` | `sessionFile, text, mode: steer\|queue\|interrupt` | 消息投递（RPC `steer`/`follow_up`/`abort`） |
| `projects_thread_read` | `sessionFile, limit?, cursor?, fullBodies?` | 分页转录（RPC `get_entries`） |
| `projects_thread_pr` | `pr, sessionFile?` | M6 链接 PR |
| `projects_coordinator_send` | `text` | thread 专用（inbox → coordinator） |
| `projects_decision_ask` / `_close` / `_show` | 同参考 | 决策卡 |
| `projects_threads_suggest` / `projects_setup_propose` / `projects_suggestion_apply` | 同参考 | M5 |
| `projects_notify` | `text, threadKey?` | coordinator 发一行提醒 |
| `projects_task_*` / `projects_send_later*` / `projects_thread_reminder` | 同参考 | M7 |

线程子进程（thread-runtime 扩展）只为子集：`projects_thread_status`、`projects_decision_ask`、
`projects_coordinator_send`、`projects_thread_read`（只读自身）、`projects_thread_pr`（M6）；所有写操作
**不直接写 `project.json`**，而是追加到 `inbox/<key>.jsonl`：

```jsonl
{"v":1,"at":"2026-11-23T07:31:00.000Z","thread":"<key>","kind":"status","status":"waiting","note":"需要权限"}
{"v":1,"at":"…","kind":"decision","decision":{…}}
{"v":1,"at":"…","kind":"report","text":"完成：…","status":"idle"}
{"v":1,"at":"…","kind":"coordinator-send","text":"…"}
```

supervisor 每 500 ms（或 fs.watch 去抖）消费并合并，**幂等**：按 `(thread, at, kind, hash)` 去重。

**命令面（非 agent）**：`/projects`（交互仪表盘）+ `/project <sub>` 文本模式（`list|new|open|info|thread-start|
thread-status|thread-read|thread-send|thread-resolve|pause|resume|archive|delete|upload|memory-add|decide|inbox`），
供脚本与 RPC/JSON 模式降级时使用。

### 2.5 prompt 组装（移植要点）

参考实现的 prompt 是“**稳定 system section + 低抖动运行时快照 + kickoff/任务消息 + 信封 + 汇报**”
（`index.js:849-944`，`961-1004`，`1291-1338`，`3211-3271`）。pi 下的对应实现：

1. **稳定 section**（`before_agent_start` 改 `systemPromptOptions.sections`，名字 `projects`）：
   identity（coordinator / thread 角色）、项目名/goal、roots、instructions 原文、MEMORY 索引、
   `library/` 与 `uploads/` 路径约定、角色规则（coordinator 只组织 / thread 单任务并回报）。
   只在 instructions/memory/roots 变化时改变 → 命中 pi 的 section 级 diff 缓存
   （`docs/extensions.md` §before_agent_start）。
2. **低抖动状态快照**：每轮返回一条 `display:false` 的 custom message（而不是 system section），
   内容为 open threads（≤15，仅 waiting/failed/review 带 note）、open decisions、suggestions；
   刻意不含 working/idle 翻转，避免 per-turn cache miss。
3. **kickoff**：有 goal → “把 goal 当作第一条需求，直接回答或开 thread”；无 goal → 只打招呼并问起点。
   **thread 任务消息**：标题 + 自包含 brief + cwd/worktree + “交付物放 library/ + 完成后
   `projects_thread_status(idle, note)` + 需要用户输入先 `waiting`”。
4. **信封与汇报**：`<project-message from="coordinator|thread" …>` + `<cited author="user">…</cited>`
   （纯文本渲染，quotes 由工具参数携带并校验，参考 `verifiedForward`，不依赖 UI）；子进程
   `agent_settled` → inbox report（正文截 6 000 chars）→ supervisor 合并 → `pi.sendMessage({customType:
   "project-report", display:true, triggerTurn:true})` 进 coordinator 会话。

### 2.6 记忆模型（移植裁剪）

- 只保留 **project scope**（`user`/`workspace` 留给未来独立插件）；格式与限制对齐参考实现（§2.1），
  条目上限 4 KB，索引 200 行 / 25 KB 截断，删除进 `memory-trash/`；`MEMORY.md` 索引作为
  `<memory-index nonce scope>` 围栏的**数据**放进稳定 section。
- 工具：`memory_read`（列表/读单条）、`memory_save`（新建或整条替换，带 revision）、`memory_delete`；
  **不做** per-turn 语义召回、变更通知、tidy latch（M7 nice）；instructions 与 memory 严格分离。

### 2.7 有意保留的差异（诚实声明，写进 README）

1. threads 跑在本机 pi 进程树内：关掉 pi 就停，**没有云端/手机连续性**（参考实现同款限制）。
2. 无 GitHub App / 云端 environment / SSO；PR 流程靠本地 `gh` CLI + worktree（M6）；无 plan 限额/计费，
   `maxThreads` 是硬上限（默认 4，同参考源码行为）；无“把已有会话变成项目”的迁移路径。
3. 终端通知是 bell / OSC / 标题栏，不是系统通知中心；不保证所有终端都支持。

---

## 3. pi-coding-agent 集成分析

> 依据：`earendil-works/pi` 源码（路径相对仓库根）+ 官方文档 `packages/coding-agent/docs/*`。
> 结论先行：**产品所需的集成点全部存在官方扩展 API**；真正的风险集中在 UI 宿主与线程运行时，而不是能力缺口。

### 3.1 扩展点总表（我们实际会用的）

| 需要 | pi 提供 | 依据 |
|---|---|---|
| 插件打包/安装 | pi package：`package.json` 的 `pi` manifest（`extensions`/`skills`/`prompts`），`pi install <path\|npm\|git>` | `docs/packages.md` |
| agent 工具 | `pi.registerTool()`，含 `parameters`(TypeBox)、`execute`、`renderCall`/`renderResult` | `docs/extensions.md` §Custom Tools |
| 用户命令 | `pi.registerCommand(name, {description, handler(args, ctx), getArgumentCompletions})`，handler 拿 `ExtensionCommandContext` | `docs/extensions.md` §registerCommand |
| 快捷键 | `pi.registerShortcut("ctrl+shift+p", {handler})` | `docs/extensions.md` §registerShortcut |
| 提示注入 | `before_agent_start` 事件：可改 `systemPromptOptions.sections`、返回 message | `docs/extensions.md` §before_agent_start |
| 拦截/审批工具调用 | `tool_call` 事件返回 `{block, reason}`，可先 `await ctx.ui.confirm(...)` | `docs/extensions.md` §tool_call，`examples/extensions/permission-gate.ts` |
| 状态与生命周期 | 事件 `session_start/shutdown`、`agent_start/agent_settled`、`ui_prompt_start/end`、`input` | `docs/extensions.md` §Events |
| 会话持久化/切换 | `SessionManager.create/open/list`、`ctx.sessionManager`；仅**命令上下文**可 `ctx.switchSession(path)`/`newSession`/`fork` | `session-manager.ts`、`docs/extensions.md` §ExtensionCommandContext |
| 自定义 UI | `ctx.ui.select/confirm/input/editor`、`ctx.ui.custom(factory, opts)`（可 `overlay:true`）、`setWidget`、`setStatus`、`setTitle`、`setFooter/Header/Editor` | `docs/tui.md`、`docs/extensions.md` §Custom UI |
| 聊天内自定义行 | `registerMessageRenderer`/`registerEntryRenderer` + `pi.sendMessage`/`appendEntry` | `docs/extensions.md` §Message and Entry Rendering |
| 主题/键位 | `theme.fg/bg/style` 语义 token；`KeybindingsManager` + `matchesKey`；`~/.pi/agent/keybindings.json` | `docs/tui.md` §Apply themes，`docs/keybindings.md` |
| 子进程/多会话 | `getPiInvocation()`（定位自身 CLI）、`pi.exec`、`RpcClient`、`createAgentSession`（SDK） | `examples/extensions/subagent/index.ts`、`docs/rpc.md`、`docs/sdk.md` |
| 终端能力 | `tui.terminal.rows/columns`、`TuiAltScreen`（fullscreen）、OSC 8/图像/超链接、`program status`（OSC 7501） | `packages/tui/src/terminal.ts`、`docs/tui.md` |

### 3.2 会话与状态持久化

- pi 的会话 = `~/.pi/agent/sessions/--<path>--/<timestamp>_<id>.jsonl`（v3 JSONL，`id`/`parentId` 树），
  session 记录自己的 `cwd` 且永不改变（`docs/sessions.md`）。**这正好是 thread 的天然身份**：
  thread = 一个 session 文件，可被 `/resume` 原生发现（把名字设成 `"<project>/<thread title>"`）。
- 插件不能把状态塞进 session log 之外的地方？可以：项目状态写自己的文件（如参考实现），
  UI 偏好可写 `ui-state.json`；**不要**用 `appendEntry` 存项目状态（会污染会话上下文与导出）。
- `switchSession` 会重建运行时（资源、扩展、cwd），旧扩展实例收到 `session_shutdown`；因此插件必须
  在 `session_shutdown` 清理定时器/子进程/overlay，并在 `session_start` 重新从磁盘加载项目状态
  （`docs/extensions.md` §Session replacement lifecycle and footguns）。
- **推论**：`/projects` 这类导航动作必须是命令（或由已打开屏幕的命令上下文在关闭屏幕后调用），
  快捷键上下文没有 `switchSession`（见 §3.7）。

### 3.3 instructions / memory 注入（pi 的三种可用通道）

| 通道 | 是否采用 | 说明 |
|---|---|---|
| `<cwd>/AGENTS.md` 自动注入 | 只做镜像，不依赖 | pi 从 agent dir、cwd 及父目录加载，**不需要 project trust**（`docs/configuration.md` §Context files）。我们把它写进 project dir 供人类阅读与手动会话用。 |
| `.pi/APPEND_SYSTEM.md` | 不采用 | 受 project trust 门控（`docs/configuration.md`），子进程还要额外 `-a/-na`；不适合作为产品数据通道 |
| 扩展 `before_agent_start` 改 `sections` | **采用** | 无 trust 依赖、可控、可缓存；thread 子进程同样加载 thread-runtime 扩展，所以两条路径一致 |
| `--append-system-prompt` 或 `-e` 临时扩展 | 备选 | 参考 subagent 扩展用临时 prompt 文件传任务（`examples/extensions/subagent/index.ts:425`）；我们的 brief 走首条 user message 即可 |

### 3.4 工具 / 命令 / 渲染器

- 工具返回值用 `{content:[{type:"text",text}], details}`；`details` 放结构化数据，`renderResult` 把
  “开 thread / 汇报 / 决策”渲染成聊天里的一行摘要（替代参考的聊天卡片）。
- coordinator 的“只读”用 `tool_call` gate：命中 denylist（bash/write/edit/…）→ `{block:true, reason:
  "请用 projects_thread_start 交给 thread"}`，可配置关闭（参考 `coordinatorReadOnly`）。
- `/projects` 是 TUI 仪表盘入口，`/project <sub>` 是文本/脚本接口（RPC、JSON 模式下唯一可用的面）。

### 3.5 线程运行时：三种形态对比（关键架构决策）

| 维度 | **T1：`pi --mode rpc` 子进程（推荐）** | T2：进程内 SDK `createAgentSession` | T3：单会话串行（无后台） |
|---|---|---|---|
| 并行/后台 | 真并行（每活跃 thread 一个 node 进程） | 真并行（同进程内多 AgentSession） | 无：只有当前会话在跑 |
| 隔离性 | 崩溃/内存泄漏/工具乱写不殃及 TUI；可单杀 | 一处未捕获异常可拖垮整个 TUI | 无隔离 |
| 审批/提问 | RPC `extension_ui_request`（select/confirm/input/editor）天然桥接 | 需自造 ExtensionUIContext 或改线程扩展把审批回调到插件（SDK 是否可注入 UI context 未验证） | 原生 UI |
| 流式读取 | stdout JSONL 事件（`message_update`/`tool_execution_*`/`agent_settled`） | `session.subscribe()` 事件流，类型更友好 | 天然 |
| 转录历史 | RPC `get_entries`/直接读 JSONL | `session.messages` | 天然 |
| 成本 | 每活跃 thread ≈ 一个 pi 进程（启动 ~百 ms、内存 ~50–100 MB） | 省进程 | 零 |
| 风险 | stdin/stdout JSONL 背压与帧解析；子进程生命周期/孤儿进程；RPC 模式 `custom()` 不可用 | 共享进程全局状态（模型 runtime、auth、telemetry、settings）耦合未验证；审批管道自造 | 违背产品核心（dispatch 后并行） |
| 先例 | `examples/extensions/subagent/index.ts`（spawn pi JSON 模式）；`docs/rpc.md` 的“IDE/自定义 UI”用途 | `docs/sdk.md`（嵌入宿主） | — |

**结论**：T1。理由：(a) 审批桥是现成的（RPC 扩展 UI 子协议）；(b) 崩溃隔离对“后台并行”是刚需；
(c) 子进程与 TUI 完全解耦，`/projects` 屏幕刷新不影响线程跑；(d) 与 pi 自带 subagent 模式同源，
维护者心智一致。T2 作为 M2 的对照 spike（若 T1 的进程开销不能接受再评估）。

线程子进程启动命令（草案）：

```
pi --mode rpc \
   --session  <thread session file> \
   --name     "<project>/<thread title>" \
   -na \
   -ne -e <abs path to thread-runtime.ts> \
   [--model <provider/model>] [--thinking <level>]
   # cwd = thread.cwd（某 root）；stdin/stdout = JSONL；stderr 落 threads/<key>.log
```

- `-na`：忽略 project-local trust 资源，避免子进程卡在 trust 提示；instructions/memory 由我们的扩展注入。
- `-ne -e …`：只加载线程运行时扩展（可配置为“也加载用户扩展”，但 RPC 模式 `custom()` 不可用，谨慎）。
- `--session` 指向由 supervisor 预先用 `SessionManager.create(threadCwd)` 建好的文件 → 路径在 spawn 前已知，
  UI 可立即列出；`--name` 让 `/resume` 里可读。

### 3.6 审批 / “等待”语义在 pi 里的落点

| 需要 | pi 机制 | 说明 |
|---|---|---|
| thread 内危险操作审批 | thread-runtime 扩展在 `tool_call` 里 `await ctx.ui.select(...)` | RPC 模式下变成 `extension_ui_request`（select），阻塞直到 supervisor 回 `extension_ui_response`；这正是 “Waiting on you” 的来源（`docs/rpc-extension-ui.md`） |
| thread 里的多步提问 | `ctx.ui.confirm/input/editor` | 同上；`custom()` 在 RPC 模式返回 undefined，不要用 |
| coordinator 里问用户 | 原生 `ctx.ui.custom` 决策卡 / `ctx.ui.select` | TUI 模式完整可用 |
| 判断会话“正在等用户” | `ui_prompt_start` / `ui_prompt_end` 事件 | 对当前 TUI 会话可直接标记 `waiting`（`docs/extensions.md`） |
| 线程完成/失败 | `agent_settled`（完成）/ 子进程非零退出或 `stopReason:error`（失败） | `agent_end` 不是终态（可能自动重试/压缩） |
| 用户“亲口要求合并” | `input` 事件（`event.text`/`event.source`） | merge guard（M6）只看 `source:"interactive"` 的最近 N 分钟原话，不依赖记忆里的长期授权；转发/引用不算 |

### 3.7 可行 vs 不可行（含风险等级）

**可行（官方路径）**

- 项目实体/工具/命令/快捷键/提示注入/聊天内自定义行/主题/状态条；仪表盘、线程查看器、决策卡等 TUI 屏幕
  （`ctx.ui.custom` ± overlay）；后台线程（RPC 子进程）+ 流式事件 + 审批桥；worktree/`gh`/库/记忆文件；
  命令上下文里的 `switchSession` 打开 coordinator ✓
- 定时任务：pi 没有内建 scheduler → 插件自建 tick + 持久化 + 「pi 进程在跑才有效」声明 ⚠

**受限（要设计绕行）**

- **快捷键不能切会话**：`registerShortcut` 的 handler 拿 `ExtensionContext`（无 `switchSession`；
  `packages/coding-agent/src/core/extensions/types.ts:1729`）；且扩展快捷键由编辑器组件分发
  （`interactive-mode.ts:2482` `defaultEditor.onExtensionShortcut`）——**当自定义屏幕/overlay 持有焦点
  时快捷键不触发**。绕行：快捷键用 `pi.sendUserMessage("/projects", {expandPromptTemplates:true})`
  走命令分发（命令上下文才有 `switchSession`）；屏幕内导航由屏幕自己的 `handleInput` 处理。
- **扩展不能拥有 alt-screen 布局**：`ViewportTUI.setLayoutRoot` 是给 app 用的（`tui.ts:588`，
  interactive mode 在 `interactive-mode.ts:974` 设置布局根），**没有 getter**，扩展接管后无法还原；
  官方文档明确“不要在扩展里创建第二个终端渲染器”（`docs/tui.md`）。
- **持久侧栏会盖住 transcript**：overlay 是合成在已有内容之上（`compositeTuiLine`/`compositeOverlays`），
  pi 的 transcript 按全宽渲染，右侧常驻 38 列 overlay 会遮住长行右侧文字。
- **RPC 子进程里 `custom()` 不可用、`setFooter/setHeader/setEditorComponent` 等降级为 no-op**
  （`docs/rpc-extension-ui.md` §Limitations）——线程的 UI 只能靠我们外层的面板表达。
- **无沙箱写根**：不移植 `sandboxPolicy.registerWritableRoot`；线程 cwd 之外的写权限靠 OS 权限与
  `tool_call` gate（可选地在 thread 上限制可用工具集）。

**高风险 / 不做**

- 接管 alternate screen 自建 pane 布局、复用 `renderLayoutFrame` 深导入：非公开 API（`layout.ts` 未从
  `packages/tui/src/index.ts` 导出），版本漂移即碎 → 不做（只在 spike 里验证可能性，用于否决方案 A）。
- 让线程对话以“原生 pi 渲染”出现在面板里：需要切换会话，见 §4.6/§6-Q2。
- 多实例同时写同一项目（用 `owner.lock` 降级为只读 viewer，不做分布式一致性）；云端/手机连续性（不做）。

---

## 4. 终端 UI 设计（本计划的核心）

### 4.1 不变量与约束（先钉死，后面所有方案都受它约束）

1. **原生会话 UI 归 pi 所有**：transcript、editor、footer、`/resume`、`/tree`、模型选择器都是 pi 的；
   我们不替换、不覆盖长期区域（最多用 1 行 widget 做环境信息）。
2. **默认 `tuiMode = "fullscreen"`**（`packages/coding-agent/src/core/settings-defaults.ts`），
   即 alt-screen + app 自管滚动；`"regular"` 模式仍要可用（降级，不崩）。
3. 扩展能拿到的“屏幕”只有两种：**编辑器槽位**（`ctx.ui.custom()` 非 overlay，`/resume` 就是这种）和
   **overlay**（可 100% 宽、按内容高，盖在现有内容上）。没有第三种。
4. 渲染只给 `width`，**高度自己从 `tui.terminal.rows` 读**；每行必须 ≤ width（否则 TUI 报错）。
5. 输入同一时刻只有一个焦点组件；自定义屏幕拿到全部按键（包括 Esc），全局扩展快捷键在此时失效。
6. 会话导航（切项目/切 thread 的原生视图）只能在命令上下文里做。
7. 终端宽度、CJK/emoji、无颜色终端都要能读。

### 4.2 三种候选方案对比

#### 方案 A：满屏分栏 + 常驻会话侧栏（接管 alternate screen）

**形态**：常驻 sidebar（项目列表或 threads）+ 中间会话区，类似 Claude Projects 的 web 布局。**需要**：
`TuiAltScreen` + `ViewportTUI.setLayoutRoot` 自建 `VStack/HStack/ScrollView` 布局，或 100% overlay +
自绘 panes；但**无法缩窄原生 transcript**，只能重画 chat（等于重写渲染）或让 overlay 盖住它。
**工作量**：高（布局/滚动/焦点/resize/鼠标/主题 × 2 模式）。**风险**：布局根无 getter 不可还原、
文档禁止第二渲染器、覆盖 transcript 破坏阅读、版本漂移。**结论**：**否决**；只借用“fullscreen 下用
100% overlay 承载模态仪表盘”这一小块。

#### 方案 B：REPL 原生分层屏幕 + 1 行环境条（推荐）

**形态**：原生 pi 会话是主体；项目感来自 (a) **1 行 project strip**（`ctx.ui.setWidget`，editor 上方）
显示当前项目与线程计数；(b) **屏幕栈**：`/projects` → Dashboard → Project workspace → Thread viewer /
Decision / Settings，每屏一个 `ctx.ui.custom()` 组件，可嵌套推入（pi 的 dialog 栈保留并恢复上一层），
`Esc` 逐层返回；(c) **原生 dialog**：确认、输入、编辑器、决策卡（`ctx.ui.select/confirm/input/editor`，
或小 overlay）。**需要**：`ctx.ui.custom`（非 overlay = regular/fullscreen 通用宿主；overlay = fullscreen
满屏宿主）、`setWidget`/`setStatus`/`setTitle`/`registerCommand`/`registerShortcut`、`SelectList`/`Input`/
`Markdown`/`Text`/`theme`、`matchesKey`/`KeybindingsManager`、`fuzzyFilter`、`truncateToWidth`/
`visibleWidth`/`wrapTextWithAnsi`。**工作量**：低-中（列表窗口、分组折叠、搜索、屏幕状态机；无自建布局
引擎）。**风险**：低（全部官方路径，与 `/resume`、`/settings`、`model-selector` 同构）。**做不到**：
并排常驻侧栏；鼠标（regular 模式无）；不能把 thread 的“原生富渲染”放进面板（用查看器近似，或切会话）。
**结论**：**推荐**。

#### 方案 C：tmux / 编辑器集成（每 thread 一个 pane）

**形态**：project = tmux session，coordinator = 一个 pane 里的 pi，threads = 其它 window/pane 里的 pi；
`/projects` 面板只是控制平面（列表、跳转、开/杀 pane）。**需要**：`pi.exec("tmux", …)`
（new-session/new-window/select-window/list-panes/capture-pane）、子 pi 进程、状态轮询、跨 pane 通知。
**工作量**：中；**风险**：tmux 依赖（Windows 原生不可用）、嵌套 tmux、版本差异、审批出现在别的 pane
（注意力分散）、状态聚合靠 poll。**做不到**：非 tmux 用户、单窗口聚合视图。**结论**：**不做基座**；
作为 M7 的 power-user 集成（`/project attach` 把 thread 丢进新 pane）保留。

#### 推荐与理由（一句话版）

选 **B**：唯一把“原生 pi 会话体验”完整保留、又能在终端表达项目/线程/注意力信息的方案；A 的成本和脆弱性
都花在重画 pi 已经画好的东西上，C 把产品体验外包给 tmux 且覆盖不了非 tmux 用户。**B 的双宿主**（同一套
screen 组件）：Host-1（默认，全模式）= `ctx.ui.custom()` 非 overlay，编辑器槽位、高 = 内容高（≈`/resume`）；
Host-2（fullscreen 增强）= `{overlay:true, overlayOptions:{width:"100%", maxHeight:"100%", row:0, col:0,
margin:0}}` 的满屏仪表盘（组件须渲染 `tui.terminal.rows` 行）。overlay 文档标注 experimental，所以 Host-1
是兜底、Host-2 只是观感增强。

### 4.3 界面骨架（coordinator 会话，fullscreen）

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  native transcript (pi)                                                      │
│  …                                                                           │
│  ⏺ coordinator: 我已把「升级依赖」交给 thread，先继续调研 UI 方案…           │
│                                                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│ ▣ pi-project-plugin   ⣾2 working   ⚠1 waiting   ○3 idle   ✓2 resolved       │ ← setWidget
├──────────────────────────────────────────────────────────────────────────────┤
│ > 继续                                                       native editor   │
├──────────────────────────────────────────────────────────────────────────────┤
│ sonnet-4.5 · ctx 23% · thinking low        project: pi-project-plugin · ⌃⇧P  │ ← footer
└──────────────────────────────────────────────────────────────────────────────┘
```

- strip 只在“当前会话绑定到项目”（coordinator 或 thread 会话）时出现，否则显示
  `no project · /projects` 一次，或在用户关闭后不显示（配置）。
- 计数只显示非零组，顺序即注意力顺序；`⣾` 是 1 行 spinner（可关）。

#### 导航模型与键位（所有屏幕共用）

**屏幕栈**：`Dashboard → Project workspace → {Thread viewer | Decision | Settings | Memory | Library |
Help}`，每层是一个 `ctx.ui.custom()` 组件；pi 的 dialog 栈会隐藏并保留上一层，关闭后原样恢复。
`Esc`＝弹栈（在看板上是关闭，在 viewer 上是“稍后回答”并保持 waiting）；`?`＝当前屏帮助。
**会话级导航**（真正切换 pi session，只能在命令上下文里做）：
`Enter` on 项目 → `switchSession(coordinator)`；`c` in workspace → 回 coordinator；`o` on thread →
M7 的“原生打开”（§4.6/§6-Q2）。所有入口都收敛到 `/projects` 命令（`/p` 别名）；
`ctrl+shift+p` 用 `pi.sendUserMessage("/projects", {expandPromptTemplates:true})` 走命令分发
（若 spike 发现会污染会话，则降级为“把 `/projects` 填进编辑器等一次 Enter”）。

| 上下文 | 按键（默认值，尽力复用 pi 的 keybinding id） | 作用 |
|---|---|---|
| 编辑器（全局） | `ctrl+shift+p` · `/p` | 打开/回到 Projects 面板 |
| 列表类屏幕 | `↑/↓`（`tui.select.up/down`）、`PgUp/PgDn`、`Home/End`、`j/k`、`g/G` | 移动光标 / 翻页 |
| 列表类屏幕 | `Enter`（`tui.select.confirm`）、`Esc`（`tui.select.cancel`）、`q` | 进入 / 返回 / 关闭 |
| 列表类屏幕 | `/`、`Tab`/`Shift+Tab`、`?` | 搜索、切分页、帮助 |
| Project workspace | `n` `i` `m` `l` `g` `p` `x` `r` `c` `t` | 新 thread、instructions、memory、library、设置、pause、stop、resolve、回 coordinator、转录 |
| Thread viewer | `Enter` 发送、`Tab` steer/queue、`ctrl+c` 中止、`y/a/n` 审批、`t` 转录、`R` 重启失败线程 | 追问与处置 |
| Decision 屏 | `1–4` 选择、`Esc` 稍后、`w` 撤回 | 回答决策卡 |

按键实现：共享动作（上下/确认/取消）用 `keybindings.matches()` 以尊重用户改键；屏幕私有键用
`matchesKey()`。**已知限制**：目前 pi 没有公开“扩展注册自定义 keybinding id”的 schema，
私有键不可改（写在 README；M7 评估用 settings 里的 JSON 映射兜底）。

**每屏状态与数据流**（屏幕只读 store 快照；所有写操作走 store/supervisor，屏幕不直接改文件）：

| 屏幕 | 数据来源（只读） | 本地状态 | 副作用 |
|---|---|---|---|
| Dashboard | ProjectStore 快照（`projects_list`） | cursor / filter / showArchived | 建/开/归档/删项目 |
| Project workspace | project.json 快照 + supervisor 运行时表 | tab / cursor / 折叠 / search | 开 thread、pause、resolve、进编辑屏 |
| Thread viewer | 子进程事件缓冲 + `get_entries` 历史 | scrollTop / follow / composer / sendMode | prompt·steer·abort、审批回答 |
| Decision | `project.decisions[]` | cursor | answer / withdraw |
| Settings·Memory·Library | project.json + 文件系统 | 表单 / cursor / dirty | 原子保存 / 上传 / 删除 |

### 4.4 屏幕 1：Dashboard（`/projects`）

```
 Projects                                   / search   n new   a archived   ? help
 ─────────────────────────────────────────────────────────────────────────────────
 ▸ pi-project-plugin        active     ⣾2  ⚠1  ○3  ✓2        updated 2m
   dsh-port-notes            paused     ·   ·   1  2         updated 1d
   api-refactor              archived   ·   ·   ·  ·         updated Nov 12
 ─────────────────────────────────────────────────────────────────────────────────
 [enter] open   [n] new   [/] search   [a] show archived   [q] close
```

- 行 = 一个项目；`▸` 当前项目；右对齐“最近活动”（Claude 相对时间，参考 `client.js:1918-1941`）。
- 新建项目 = 一屏 wizard：`name`（必填）→ `goal`（可空）→ `root`（默认当前 cwd）→ `instructions`
  （可跳过，`ctx.ui.editor`）；Esc 任一步返回且不丢已填项。空状态：`No projects yet. Press n to create one.`

### 4.5 屏幕 2：Project workspace（项目内主屏，签名界面）

```
 ▣ pi-project-plugin   active                                      [c]oordinator
 ┌─ threads (5) ────────────────┬─ detail ──────────────────────────────────────┐
 │ ⚠ Waiting on you 1          │ 升级 vitest 到 v2                              │
 │   ⚠ 权限审批 · 升级依赖      │ working · 6m · last: bash pnpm -r test        │
 │ ⣾ Working 2                  │ session: …/2026-11-23_ab12.jsonl               │
 │   升级依赖                   │ [enter] open   [x] stop   [r] resolve         │
 │   终端 UI 设计               │ [t] transcript                                 │
 │ ○ Idle 2                     │                                                │
 │   reference research         │ goal     在 pi 里复刻 Claude Projects          │
 │   memory 迁移                │ roots    ~/repo/pi-project-plugin              │
 │ ✓ Resolved 1                 │ instr    1.2k chars · memory 12 · library 4    │
 └──────────────────────────────┴────────────────────────────────────────────────┘
 [n] new thread  [i]nstr  [m]emory  [l]ibrary  [g]settings  [p]ause  [c]oordinator  [q] back
```

- 左栏是**唯一需要滚动窗口的列表**：分组折叠（组头 `▸/▾` + 计数）、组内按 `updatedAt` 倒序、
  光标跨组移动、组头也可被选中（Enter 折叠/展开）；窄屏时隐藏右栏，Enter 进入 detail 子屏。
- 右栏 detail 是“选中项的 8–12 行摘要”，不做聊天渲染；`enter` 打开 Thread viewer，`t` 是只读转录分页。
- 组顺序固定为注意力顺序（§2.1）；`reminder` 组（M7）插在 idle 之后、resolved 之前。

### 4.6 屏幕 3：Thread viewer（读结果 / 追问 / 审批）

```
 ← pi-project-plugin / 升级依赖          ⣾ working · 6m · turn 3 · [o] native (M7)
 ─────────────────────────────────────────────────────────────────────────────────
 [user]  把 vitest 升到 v2，所有 packages 一起升，最后跑全量测试
 [tool]  ✓ read packages/foo/package.json
 [tool]  ✓ bash pnpm -r update vitest@^2   (2.1s)
 [tool]  ⣾ bash pnpm -r test · 142/188 tests
 [agent] 已升级 4 个包，测试进行中；dsh-utils 有 3 个用例失败，正在看输出…
 ┌─ needs you ───────────────────────────────────────────────────────────────────┐
 │ 允许执行 `pnpm -r test --force` 吗？   [y] once   [a] always   [n] deny   [v] view │
 └───────────────────────────────────────────────────────────────────────────────┘
 > 只看 dsh-utils 的失败                                                        │
   [enter] send   [tab] steer|queue   [ctrl+c] abort   [t] transcript   [esc] back
```

- **不是完整 chat 渲染器**：工具调用压成一行，assistant 文本按 `Markdown` 组件渲染但只保留最后
  ~2 000 行；thinking 折叠成 `思考中…` 一行；图片/附件显示为 `[image 1] path`。
- `t` = 打开 `projects_thread_read` 的分页只读视图（不切会话）；M7 的 `o` = 原生打开（需切会话，见 §6-Q2）。
- 审批块出现时自动把焦点放到审批块（`[y]/[a]/[n]` 单键），Esc 不关闭屏幕而是“稍后回答”并把状态保持
  waiting（有 badge/通知）。

### 4.7 屏幕 4：Decision（决策卡）

```
 ┌ Decision · from thread "升级依赖" ────────────────────────────────────────────┐
 │ 要不要把 vitest 升级限制在 devDependencies？                                 │
 │ context: 3 个包把 vitest 放在 dependencies，升级会改变 production 依赖树。    │
 │                                                                              │
 │  [1] 只升 devDependencies（推荐）                                             │
 │      生产依赖不动，需要改 3 个 package.json；线程继续跑测试                   │
 │  [2] 全部升级                                                                │
 │      依赖树最小差异，但 production 依赖会变；需要重新回归                     │
 │  [3] 先停下，等我看完影响面                                                   │
 │                                                                              │
 │ 为什么推荐 [1]：本次目标是测试框架升级，改动集中在 package.json，可回滚       │
 │ [1][2][3] 选择    [w] 撤回（提问方可撤）    [esc] 稍后再答（线程保持 waiting）│
 └──────────────────────────────────────────────────────────────────────────────┘
```

- 单屏 1 问，选项 2–4 个，**每项必须有一行“后果”**，推荐项标 `（推荐）`；理由单独一行。
- 回答后：写入 decision（`chose`, `answeredBy: user`, `closedAt`）→ 通过 `projects_thread_send(steer)`
  把“用户选择 + 原话”回给提问者；若用户在聊天里用文字回答，agent 调 `projects_decision_close`。
- 决策同时以**一行提示**出现在 coordinator transcript（`customType:"project-decision"` 渲染器），
  用户也可以在聊天里直接答。

### 4.8 裁决与审批的终端映射（两类，不要混）

| 类型 | 来源 | 终端表现 | 数据落点 |
|---|---|---|---|
| **工具审批**（allow once/always/deny） | 线程子进程 `tool_call` gate → RPC `extension_ui_request` | Thread viewer 内嵌审批块（首选）/ 若用户不在看，则 coordinator strip 变 `⚠1 waiting` + 通知；按 `y/a/n` | `project.permissions.allow[]`（always 规则 = glob/命令前缀；一次性授权不落盘） |
| **项目决策卡** | agent 工具 `projects_decision_ask`（coordinator 或 thread 经 inbox） | Decision 屏（overlay/编辑器槽）；同时聊天里一行入口 | `project.decisions[]` |
| **合并授权**（M6） | 用户亲口说“merge” | 不需要 UI：`input` 事件（`source:"interactive"`，排除 extension/RPC 注入）记录最近原话 + 时间，merge gate 校验 `mergeGrantMs=1h` | 内存（不落盘长期授权） |
| **不可逆操作确认** | pause/archive/delete project、删记忆、stop thread | `ctx.ui.confirm` 原生确认，默认焦点在“取消” | — |

“Waiting on you” 的判定（实现级，三源取并集）：
1. 该 thread 的 RPC 子进程有**未响应**的 `extension_ui_request`（dialog 类）；
2. `project.decisions[]` 中存在 `asker` 为该 thread 的 open 卡；
3. thread 通过 `projects_thread_status(waiting, note)` 显式上报（inbox）。
另外 `failed` 单独成组但仍算 urgent（通知/排序与参考一致）。

### 4.9 流式输出如何渲染

**coordinator 会话**：原生渲染，插件只做三件事：
1. 汇报/决策/建议/thread 汇报行用 `registerMessageRenderer` 渲染成紧凑自定义行（参考的“安静行”语义）；
2. `ctx.ui.setStatus("projects", "⚠1 waiting")` 与 strip 计数随状态变化节流刷新；
3. `waiting` 时 `ctx.ui.setTitle("⚠ pi — pi-project-plugin")`（有其它扩展占用则退让）。

**thread 子进程**（面板内）：事件 → 行模型（只在内存里，不写会话文件）：

| RPC 事件 | UI 行为 |
|---|---|
| `message_start`（user/assistant） | 新增一条消息行；user 用 `[user]` 前缀，assistant 用 Markdown 组件 |
| `message_update` `text_delta` | 追加到当前 assistant 缓冲；**≤50 ms 合并一次**重渲染；只保留尾部 2 000 行 |
| `message_update` `thinking_delta` | 折叠为一行 `思考中… (N 字)`，不流式展示正文 |
| `tool_execution_start` | 立即插入 `⣾ <tool> <args 摘要>` |
| `tool_execution_update` | 更新该行的“最后一行输出”（截断 1 行，最多 120 列） |
| `tool_execution_end` | 改为 `✓/✗ <tool> (duration)`；失败时展开 3 行 |
| `extension_ui_request`（dialog） | 插入审批/提问块，状态 → waiting，聚焦到该块；回答后发 `extension_ui_response` |
| `agent_settled` | 状态 → idle（或 waiting/review）；未读标记；触发 report 合并 |
| 子进程退出（非 0） | 状态 → failed + note（stderr 末 3 行）+ 通知 |
| 打开 viewer（首次） | `get_entries` 拉最近 200 条 → 转成行模型，再接实时事件；切换 thread 时丢旧缓冲 |

性能预算：单 thread 缓冲 ≤ 2 MB，面板不可见时不保留 delta 文本（只记计数），重渲染 ≤ 20 fps；
supervisor 同时最多 4 个活跃子进程（`maxThreads`），超出拒绝。

### 4.10 长列表 / 滚动 / 搜索（终端里最容易做砸的部分）

- **窗口算法**（所有列表共用）：`maxVisible = max(5, rows - 固定行数)`；`start = clamp(sel - maxVisible/2,
  0, n - maxVisible)`；`scrolloff=2`；末行显示 `(12/57)` 或 `⋮ 45 more`。
- **分组列表**：先展开成“行数组”（组头行 + 线程行）再做窗口，折叠只影响行数组；窗口起点落在组内时把
  组头插到第一行（sticky header）。
- **搜索**：`/` 进入内联输入（pi-tui `Input`），`fuzzyFilter()` 过滤标题/note/PR 号；搜索时折叠失效
  （扁平化 + 命中高亮），Esc 清空并恢复分组。
- **大数据**：所有记录在一个 `project.json`；>200 线程时 virtual window + `updatedAt` 排序、不全量重绘；
  library >1 000 文件只列顶层 + 分页。**不要用 `ScrollView` 做面板内滚动**：它依赖 layout 引擎
  （`LAYOUT_NODE`），在 overlay/编辑器槽里拿不到视口高度（`packages/tui/src/components/scroll-view.ts`）；
  自己维护 `scrollTop` + 窗口切片（参考 `/resume` 的 `SessionList` 做法）。

### 4.11 自适应与 resize

| 宽度 | 布局 |
|---|---|
| ≥ 100 | Project workspace 双栏（左 threads / 右 detail）；strip 显示计数与按键提示 |
| 72–99 | 单栏：threads 列表 → Enter 进 detail 子屏；strip 只显示计数 |
| 48–71 | 列表隐藏相对时间/PR chips；detail 只显示标题 + 状态 + note；strip 缩成 `▣ p1 ⣾2 ⚠1` |
| < 48 | 只显示 `标题 + 状态字`；提示 `resize to ≥48 cols for full view`；所有边框改用单字符 |
| 高度 < 16 | 屏幕退化为“单行提示 + 关闭键”；不渲染仪表盘 |

实现规则：所有行用 `truncateToWidth()`/`wrapTextWithAnsi()`、宽度用 `visibleWidth()`（CJK/emoji = 2 列，
pi 内置 `get-east-asian-width`）；不硬编码 80 列，`render(width)` 内不缓存跨宽度字符串。高度从
`tui.terminal.rows` 读（`packages/tui/src/terminal.ts:111-112`），overlay 设 `maxHeight:"100%"` 且
**组件渲染行数必须 ≥ 需要行数**，否则 overlay 高度会缩；resize 每帧重渲染（overlayOptions 可用函数求值）。

### 4.12 颜色与无障碍

- 只用 pi 语义 token（`accent/success/warning/error/muted/dim` + `selectedBg`/`toolSuccessBg` 等背景
  token），不写裸 ANSI，dark/light 主题自动生效（`docs/tui.md` §Apply themes correctly）。
- **状态永不仅靠颜色**：`glyph + 状态词`（`⚠ waiting`、`⣾ working`、`✗ failed`、`✓ resolved`），
  单色终端/`NO_COLOR` 可读；spinner 默认 4 fps（可关），不闪烁；regular 模式不做全屏重绘动画。
- 通知：bell（可选）+ OSC 9/777（失败静默）+ `setTitle` 前缀，长文本用 `/project inbox` 查看不刷屏；
  屏幕阅读器友好 = 固定行数窗口 + 纯文本可复制错误；**不**每帧改变行数。

### 4.13 空状态 / 错误 / 首次运行 / 模式降级

- 首次运行：空状态 + `n` 建项目 → 自动打开 coordinator（可配置不自动跳）；空列表/空搜索有明确文案。
- coordinator 会话文件缺失：dashboard 行 `⚠ missing coordinator`，Enter 重建（提示丢历史）；项目
  JSON 损坏：备份 `project.json.bad-<ts>` 并从 `AGENTS.md`/inbox 重建最小记录（尽力而为）。
- 子进程崩溃：thread → failed，viewer 保留最后 200 行 + stderr 尾部，`R` 重启（同会话文件继续）。
- `owner.lock` 被其它 pi 实例持有：本项目降级只读（`🔒 owned by pid 1234`），不 spawn/不写；
  `/project take` 确认后抢占（另一实例心跳检测后自动降级）。
- 模式降级：`ctx.mode !== "tui"`（RPC/JSON/print）时 `/projects` 打印纯文本列表；所有 UI 调用前走
  `ctx.hasUI` 判断（`docs/extensions.md` §Mode Behavior）。

---

## 5. 里程碑（M1–M7，每个可独立发布）

> 依赖：M1→M2→M3 主线，M4/M5 可与 M3 并行，M6 依赖 M2，M7 可选；每个 milestone 结束时仓库可
> `pi install ./`、README 更新、至少一个 demo 脚本/测试。

### M1 项目外壳（单线程可完成，≤ 3 天）

**交付**
1. `pi-projects` 包骨架：`package.json`（`pi.extensions`）、`tsconfig`、`node --test` 单测、`README`。
2. `ProjectStore`：`~/.pi/agent/projects/<id>/` 创建/读取/原子写（tmp+rename）、`owner.lock`、slug 规则。
3. `/projects` 命令 + Dashboard 屏幕（Host-1 编辑器槽；fullscreen 用 Host-2 满屏 overlay）+ 新建 wizard
   （name→goal→root→instructions）+ 打开/归档/删除；strip（`setWidget` 1 行）+ `setStatus`。
4. coordinator 会话：`SessionManager.create(root)` 建文件 → 写 kickoff user message → `switchSession()`；
   绑定检测（`session_start` 按 session file 匹配）；instructions 注入（`before_agent_start` 拼稳定
   section：identity/goal/roots/instructions）+ 镜像写 `<projectDir>/AGENTS.md`。
5. Spike 记录：overlay 宿主在 fullscreen/regular 下的输入、resize、与全局键位的交互（`docs/SPIKE-M1.md`）。

**验收**：`/projects` 建项目 → 自动进入新 coordinator 会话（cwd=root，kickoff 可见）→ 项目再打开可回到
该会话；instructions 出现在 system prompt（用 `/session` 或一次性 debug 工具验证）；dashboard 三态可切；
`pi install ./` 后重开 pi 数据仍在。

**不做**：thread、memory、library、decision、PR。

### M2 会跑的 thread（最大的一步，~1 周）

**交付**
1. `thread-runtime` 子进程扩展（`status/decision/coordinator_send/report` + 审批 gate + prompt 注入）。
2. supervisor：spawn/kill/steer/queue/abort、JSONL 解析（按 LF 切帧，不用 readline）、状态机、重启、
   孤儿进程回收（`session_shutdown`/SIGINT 时 kill 进程组）。
3. `projects_thread_start` / `_send` / `_status` / `_resolve` / `_read` 工具 + 线程自报 inbox 消费。
4. Thread viewer 屏幕（§4.6 完整行为：流式 tail、审批块、composer、stop）。
5. 汇报：`agent_settled` → report → 合并 → coordinator 的 `project-report` 渲染行（`triggerTurn`）；
   `maxThreads` 硬上限；session 命名 `<project>/<title>`。

**验收**：coordinator 里说“开一个 thread 升级 vitest” → `projects_thread_start` → dashboard 出现 working →
viewer 看到流式输出 → 线程请求审批 → viewer 里 `y` → 线程继续 → 完成后 coordinator 收到一行汇报；
`/resume` 里能看到该 thread 会话；关掉 pi 后无残留进程。

### M3 Overview 与注意力（~4 天）

**交付**：8 组排序与折叠、相对时间、计数徽标、搜索/隐藏 resolved、`waiting` 三源派生、
`failed` 状态与重试、auto-resolve（7 天）、pause/resume 中断所有活跃线程、archive 行为、
通知（bell/OSC/title/footer 徽标 + inbox 命令）、strip 计数、屏幕内 `?` 帮助。
**验收**：构造 5 个不同状态的线程，dashboard 顺序/计数/等待徽标与参考截图一致（文字版）；
释放/归档/暂停都有可见状态变化；通知只在“需要你”时出现一次（去重）。

### M4 常驻上下文：memory / instructions / library / roots（~4 天）

**交付**：project memory store（格式对齐 §2.6）+ `memory_read/save/delete` + Memory 屏；
instructions 编辑器（`ctx.ui.editor`，16k 校验 + dirty 保护）；library 列表（产出 + uploads）+
`/project upload <path>` 复制语义（重名 Replace/Skip）；roots 增删（目录选择用 `ctx.ui.input` 输入绝对路径 +
存在性校验）；`projects_info` 汇总。
**验收**：`memory_save` 后新 thread 的 prompt 里出现索引；instructions 修改后新 thread 生效、
 coordinator 立即生效（下一轮）；upload 后 library 列出且线程可读。

### M5 决策与建议（~4 天）

**交付**：`projects_decision_ask/_close/_show` + Decision 屏 + 聊天内入口行 + answered-in-words；
`projects_threads_suggest` / `projects_setup_propose` / `projects_suggestion_apply` + 勾选式应用屏
（`start: all|indices`、`skip`、publish 的 name/visibility 编辑）；建议状态生命周期。
**验收**：from-thread 决策在 dashboard 显示 waiting，回答后线程继续；建议卡默认全选、可跳过一项再应用；
文字回答也能关闭卡片（agent 调 close）。

### M6 编码能力对齐：worktree / PR / merge guard / publish（~1 周）

**交付**：每 thread git worktree（`pi.exec git worktree add`，分支前缀 `projects/`，dirty 仓库 preflight）；
thread 内 PR 规则注入 + `projects_thread_pr` 链接 + 后台轮询（默认 2 min，`gh pr view --json`）→
review/landing/resolved 状态迁移 + CI 失败/新评审意见唤醒线程；merge guard（用户近期 input 授权）；
`projects_github_publish`。**验收**：一次真实（或本地 bare repo + fake gh）闭环：thread 开分支→开 PR→
轮询到 CI 失败→唤醒修复→合并后 resolve；未授权 merge 被拒。**风险**：`gh` 环境/认证/轮询频率（用 `--fake-gh`
脚本做 CI 测试）。

### M7 可选增强与打磨（按需）

scheduled tasks / reminders（自建 tick + 持久化 + “pi 在跑才有效”提示）；`/project attach` tmux 集成；原生打开
thread（切会话）；memory tidy / recall；usage 统计；`--plain`/`--json`；10k 行列表与 4 并发压测；i18n。

---

## 6. 需要用户拍板的 5 个问题

> 每个都有推荐值；不定也不阻塞 M1（M1 先按推荐实现，后续改动有明确回退点）。

**Q1. UI 形态：REPL 分层屏幕（推荐）vs 全屏常驻分栏 vs 先只做命令行？**
- 推荐**分层屏幕**（§4.2-B）：完整保留 pi 原生会话体验、成本最低、与 pi 的 UI 语言一致；代价是没有
  Claude 那样的“常驻右侧面板”。备选 1 = fullscreen 加可折叠**右侧常驻 rail**（overlay ≈36 列，更像
  Claude 但遮住 transcript 右侧文字，阅读体验下降，放 M7）；备选 2 = 先只做 `/project …` 纯命令
  （最快，但没有“一眼看到谁在等我”）。

**Q2. “打开 thread”的语义：面板查看器（推荐）vs 切到该 thread 的原生 pi 会话？**
- 推荐**先做面板查看器**（coordinator 始终在线、thread 后台继续跑）；原生打开放 M7（切走期间 coordinator
  不在线，汇报要落盘补投）。选原生打开优先：阅读体验最好，但牺牲并行汇报与会话切换的顺滑度。

**Q3. 项目数据目录放哪？**
- 推荐**全局 `~/.pi/agent/projects/<id>/`**（同参考实现的 `$DSH_HOME/projects`）：与仓库解耦、不会被误
  提交、多仓库项目可共用；代价是不随仓库迁移。备选 = 仓库内 `.pi-projects/`（可团队共享，但会把可变
  JSON/记忆/库提交进 git，且同仓多项目/非 git 目录要另设计）。

**Q4. M3 之后的优先级：PR/GitHub 工作流（M6）vs 记忆/库深度（M4+M5）vs 定时任务（M7）？**
- 推荐 **M4 → M5 → M6**（先把“上下文 + 决策”闭环，再上 PR）；若你主要做代码任务，把 M6 提到 M4 之前。

**Q5. 线程运行形态：RPC 子进程（推荐）vs 进程内 SDK？**
- 推荐 **RPC 子进程**（隔离、可后台并行、审批桥现成）；代价是每活跃 thread 一个 node 进程（约 50–100 MB）
  与 JSONL 帧/进程管理复杂度。备选进程内 `createAgentSession` 省进程、事件类型更友好，但要自造审批管道、
  共享进程状态风险未验证 → 只在 M2 做一天 spike 对比，不阻塞主线。

---

## 7. 风险登记与验证 spike

| # | 风险 | 影响 | 缓解 / spike | 里程碑 |
|---|---|---|---|---|
| R1 | `ctx.ui.custom()` overlay 标注 experimental；扩展快捷键在自定义屏幕持焦时失效 | 满屏仪表盘/决策卡体验、导航 | adapter 层隔离（`host-fullscreen.ts`/`host-editor-slot.ts`）+ Host-1 兜底；导航走 `/projects` 命令分发；pin pi 版本 | M1 |
| R2 | RPC 子进程 JSONL 解析、背压、孤儿进程 | 卡死/泄漏 | 按 LF 切帧（不用 readline，Unicode 分隔符问题见 `docs/rpc.md` §Framing）；stdin 背压；`session_shutdown`/SIGINT 杀进程组；每线程日志文件 | M2 |
| R3 | `switchSession` 重载扩展，定时器/子进程/overlay 泄漏 | 切换后状态丢失、孤儿 | 所有资源挂在 `session_shutdown` 清理；状态只从磁盘重建；spike 覆盖“切走再切回” | M1/M2 |
| R4 | 多实例打开同一项目写冲突 | 数据损坏 | `owner.lock`（pid+心跳 10s）→ 第二实例只读；`/project take` 显式抢占 | M2 |
| R5 | 终端宽度/CJK/emoji/主题差异 | 界面错乱 | 只用 width-aware 工具函数；`visibleWidth` 断言测试；48/72/100 三档快照测试（虚拟终端） | M1/M3 |
| R6 | pi API 漂移（overlay/extension types/快捷键分发） | 升级即碎 | 只在 adapter 层引用实验 API；CI 里对 pi 版本跑 typecheck + 冒烟（装 latest 与 pin 两档） | 持续 |
| R7 | 线程子进程里用户扩展行为不可控（RPC 限制） | 线程功能降级/崩溃 | 默认 `-ne`，只加载 thread-runtime；配置项允许“加载用户扩展”，文档标注风险 | M2 |
| R8 | 汇报风暴/渲染卡顿（4 线程同时流式） | TUI 卡 | 合并重渲染（≤20 fps）；不可见面板不保留 delta；每个 thread 行缓冲上限；压测脚本 | M2/M7 |
| R9 | 用户对“等待/提醒”不敏感（终端无系统通知） | 错过需要处理的事 | strip 计数 + 标题前缀 + bell/OSC（可配）+ `/project inbox` + `waiting` 组置顶 | M3 |

---

## 8. 附录

### A. pi 关键 API 引用清单（实现时按此查）

| 用途 | 文件 / 文档 |
|---|---|
| 扩展总览、事件、ctx、API 方法 | `packages/coding-agent/docs/extensions.md` |
| TUI 组件模型、overlay、主题、键位 | `packages/coding-agent/docs/tui.md`；`packages/tui/README.md` |
| 组件/布局/列表/滚动源码 | `packages/tui/src/{index.ts,tui.ts,tui-alt-screen.ts,layout.ts}`、`packages/tui/src/components/*` |
| 键位定义与匹配 | `packages/tui/src/keybindings.ts`（`TUI_KEYBINDINGS`）、`packages/tui/src/keys.ts`（`matchesKey/Key`） |
| 交互模式实现（编辑器槽、dialog 栈、快捷键分发） | `packages/coding-agent/src/modes/interactive/interactive-mode.ts`（`pushExtensionDialog`、`showSelector`、`showExtensionCustom`、`setupExtensionShortcuts`） |
| fullscreen 布局根 | `packages/coding-agent/src/modes/interactive/{tui-renderer.ts,chat-viewport.ts}` |
| 会话文件/SessionManager | `packages/coding-agent/src/core/session-manager.ts`；`docs/session-format.md`、`docs/sessions.md` |
| RPC 协议与命令 | `docs/rpc.md`、`docs/rpc-commands.md`、`docs/rpc-extension-ui.md`、`docs/json.md` |
| 子进程先例 | `packages/coding-agent/examples/extensions/subagent/index.ts`（`getPiInvocation`、spawn、Ctrl+C） |
| 审批 gate 先例 | `packages/coding-agent/examples/extensions/permission-gate.ts` |
| SDK / 进程内会话 | `docs/sdk.md`；`packages/coding-agent/examples/sdk/*` |
| 打包/CLI/设置/trust | `docs/packages.md`、`docs/cli.md`（`pi install`、`-e`、`--session`、`-na`/`-ne`）、`settings-schema.ts`/`settings-defaults.ts`（`tuiMode: "fullscreen"`）、`docs/configuration.md` |

### B. 参考实现（dsh-project-plugin）关键文件

| 文件 | 内容 |
|---|---|
| `dsh-projects/index.js`（5 772 行） | Host facility：项目/线程生命周期、worktree、状态机、汇报合并、决策/建议、记忆接线、prompt 组装、guards、18 个工具、RPC action、`/project` 命令 |
| `dsh-projects/schedule.js`（763） | 定时任务/消息/reminder + 8 个工具 |
| `dsh-projects/github.js` / `library.js`（各 172） | PR/发布与文件库，几乎可照搬逻辑 |
| `dsh-memory/index.js`（1 013） | 记忆 store、scope、索引格式、revision 协议、快照/召回/通知 + 3 个工具 |
| `dsh-projects/client.js`（5 727） | **仅作 UX 参考，不移植**：分组顺序、行解剖、导航、卡片、通知 diff |
| `research/claude-projects-beta.md`（487）/ `research/design.md`（280） | 产品语义（core/nice/skip 依据）/ 早期架构决策与偏差记录（注意落后于源码） |

### C. 本仓库目录规划（实现阶段）

```
pi-project-plugin/
  package.json                 # name: pi-projects; pi.extensions: ["./src/index.ts"]
  src/
    index.ts                   # 主扩展：工具/命令/事件/UI 装配
    store/ · supervisor/       # project store、inbox/lock；thread 运行时（spawn RPC、状态、汇报合并）
    prompt/                    # 指令/记忆/状态/kickoff 组装
    ui/                        # screens + strip；ui/host/ = Host-1 编辑器槽 / Host-2 overlay adapter
    commands/ · tools/         # /projects、/project <sub>；projects_* / memory_*
    thread-runtime.ts          # 子进程扩展（-e 加载）
  docs/IMPLEMENTATION-PLAN.md · docs/SPIKE-*.md · test/(单测+虚拟终端快照+fake-gh/fake-pi 集成)
```

### D. 命名与约定

- 工具/命令/字段英文 identifier，用户可见文案默认英文（M7 加 zh 文案表）；状态值
  `working | waiting | review | landing | idle | failed | resolved`（与参考一致，别自创），组名
  `Waiting on you / Failed / Ready for review / Working / Landing / Idle / Reminder set / Resolved`。
- 时间存 ISO 8601 UTC，显示 Claude 风格相对时间（`just now / 5 min / 1 hour / yesterday / Nov 13`）。
- ID：project = slug（冲突 `-2`…）；decision = `dec_<10 hex>`；suggestion = `sug_<10 hex>`；
  thread key = session 文件 basename，日志/inbox 用它。
