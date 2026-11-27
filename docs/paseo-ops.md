# Paseo 管控执行 Agent 实操指南

本文档补充 `AGENTS.md` 中的具体操作方式，帮助个人助理稳定、无冲突地调度 Paseo 执行 Agent。

## 1. 识别当前个人助理自己的 Paseo ID

- 优先直接读取环境变量 `$PASEO_AGENT_ID`（Paseo 会为当前会话自动注入），对应的工作区与目录环境变量为 `$PASEO_WORKSPACE_ID`、`$PASEO_AGENT_CWD`。
- 若环境变量缺失，可调用 `list_agents(cwd="<当前工作目录>", statuses=["running"])` 结合当前任务标题确认。

## 2. 归属记录与防撞车流程

为了支持多个个人助理在同一目录并行运行，且不污染用户自己的 Paseo 标签：

1. **禁止修改 Paseo `labels`**：创建或更新执行 Agent 时不要自行写入 `labels`。
2. **在当前任务的 `.note/goal-<target>/` 中维护 `agents.md`**，记录个人助理与执行 Agent 的 1 对 N 映射：

```markdown
# Agent 归属记录

- 个人助理 Paseo ID：<当前 $PASEO_AGENT_ID>
- 个人助理 Workspace ID：<当前 $PASEO_WORKSPACE_ID>

## 管理的执行 Agent

| Execution Agent ID | Server ID | 目标工作区 | 负责事项 | 来源 | 状态 |
|---|---|---|---|---|---|
| `<agentId>` | `<serverId 或本机>` | `<cwd / workspaceId>` | `<简短任务说明>` | `用户指定 / 助理新建` | `active / done` |
```

3. **派活前先查重**：在接管一个已有 `agentId` 并准备调用 `send_agent_prompt` 前，先快速扫描本地 `.note/goal-*/agents.md`：
   - 如果该 `agentId` 已被另一个仍在运行的个人助理认领，**只读不发**，并告知用户；
   - 如果用户明确指示“由你接管”，再将其登记到你自己的 `agents.md` 中（并提醒原助理不再操作）。

## 3. 目标目录 `.note/` 维护与防压缩丢失

### 3.1 目录规范与防提交红线

- 每个独立任务在仓库根目录下创建 `.note/goal-<target>/`（`<target>` 为任务简写），内部维护 `goal.md`、`current.md`、`context.md`、`agents.md`。
- `.note/` 已在 `.gitignore` 中忽略，**严禁将 `.note/` 加入 Git 暂存区或提交到代码仓库**。

### 3.2 按输入方严格区分目标修改权限

Paseo 推送的执行 Agent 消息与完成通知会以系统包裹形式进入会话，必须和用户本人的消息区分开：
- **用户本人的输入**：可以按需更新 `current.md`（修改前先 `cp` 备份为 `current-<timestamp>.md`）；只有用户明确提出调整整体方向时才修改 `goal.md`。
- **执行 Agent 的回复与通知**（`agent-message` / `agent-notification`）：**严禁修改 `goal.md` 或 `current.md`**。收到后应读取已保存的目标核对执行方向，只按需更新 `agents.md` 状态或将阶段结论记入 `context.md`。

### 3.3 用 `context.md` 防止上下文压缩丢信息

面对多轮复杂任务，在 `.note/goal-<target>/context.md`（并在 `current.md` 轻量链接引用）持续更新以下硬信息，不写流水账：
- 用户明确敲定的约束、偏好与决策；
- 已验证采纳或已否决排除的方案及关键原因；
- 关键文件、报告、分支、服务地址等产物路径；
- 当前阶段的未决事项与下一步重点。

对话变长、重启或感觉发生上下文压缩后，先读取 `goal.md`、`current.md`、`agents.md` 和 `context.md`，再继续判断和派活。

## 4. 创建与接管执行 Agent

### 4.1 接管已有执行 Agent（优先）

用户直接提供 `agentId` 时：
1. 先调用 `get_agent_status` 和 `get_agent_activity` 了解它当前的进度、上下文与运行状态；
2. 确认无归属冲突后，登记到 `.note/goal-<target>/agents.md`；
3. 按需通过 `send_agent_prompt` 下发澄清、纠偏或下一步任务。

### 4.2 新建执行 Agent（仅在未提供且需执行任务时）

1. **先查已有工作区并复用（严禁省略 `workspaceId`，禁止落在本助理目录）**：
   - Paseo 的 `create_agent` 不支持传裸 `cwd`，必须归属某个 Workspace；往已有 `workspaceId` 创建 Agent 会直接在该工作区下开新标签页，不会新增 Workspace。
   - **严禁省略 `workspaceId` 或把执行 Agent 建在本助理工作区（`$PASEO_WORKSPACE_ID`）**：执行 Agent 落在本目录会误读个人助理守则，把自己也当成个人助理并再次套娃派子 Agent。
   - **选区顺位**：先调用 `list_workspaces`（跨主机先 `list_hosts` 取 `serverId`）：
     1. 优先匹配并复用目标仓库/项目目录**已有的 `workspaceId`**；
     2. 找不到合适的仓库工作区，或属于不绑定特定仓库的通用/跨机运维任务，**默认复用用户主目录（`~`）或通用目录已有的 `workspaceId`**；
     3. 仅当目标目录（或用户主目录）在对应主机上完全没有任何已有 workspace，或用户明确要求新开独立 worktree 分支时，才允许调用 `create_workspace`（禁止随手新建）。
   - 工程、脚本、SSH、跨机运维等操作无论多简单都不要由个人助理自己敲命令代劳；但基于已有上下文的**轻量纯文本梳理与产出**（如直接编写/修订 Prompt、Skill、规则、文案）由个人助理自己完成，避免转手外包造成上下文折损，只有涉及**大量阅读、批量处理或并行提速**时再派执行 Agent。
2. **加载 Paseo Profile 并按三类角色选型**：
   - 新建执行 Agent 前，**先调用 `list_profiles` 读取用户的最新配置与 `notes`**。`create_agent` 没有单独的 `profile` 字段，需把选中 Profile 的字段原样映射进去：
     - `provider` ← `"<profile.provider>/<profile.model>"`
     - `settings.modeId` ← `profile.modeId`（纯调研、代码阅读或方案评估可改为 `"plan"`）
     - `settings.thinkingOptionId` ← `profile.thinkingOptionId`
     - `settings.features` ← `profile.options`（为 `null` 时省略不传）
   - 若用户尚未配置任何 Profile，可调用 `list_providers` / `list_models` / `inspect_provider` 查询可用能力后按下方三类分工择优选择。
   - **三类角色分工顺位（与 Profile `notes` 互为双保险，避免用户未写 `notes` 或漏读导致错配）**：
     - **`Linguist`（全局默认首选 · 强文本与大上下文）**：调研分析、大上下文文本/评论/对话/日志内容理解、质量归因、方案评估、Agent Prompt/Skill 优化，以及同时涉及取数与语义分析的混合任务（自动放行模式下同样可执行 SSH/查库/命令行取数）。
     - **`Builder`（代码编写与工程实现默认首选）**：写代码、改需求、写脚本、常规代码排障、单测构建、跨机运维同步等。
     - **`Grandmaster`（终极攻坚 · 极复杂困难与深度推理专用）**：平时不轻易启用；仅在任务非常复杂困难、需要深度推理——例如复杂算法设计、极难的系统/并发根因排障、重度跨模块架构重构，或 `Builder` 难以解决的硬骨头——时才升级使用。

**映射为 `create_agent` 的通用调用结构**：

```json
{
  "title": "<简短明确的任务标题>",
  "provider": "<profile.provider>/<profile.model>",
  "workspaceId": "<必填：复用的目标仓库或用户主目录已有 workspaceId，严禁省略>",
  "notifyOnFinish": true,
  "settings": {
    "modeId": "<profile.modeId，纯调研/规划可改用 plan>",
    "thinkingOptionId": "<profile.thinkingOptionId>",
    "features": "<profile.options，为 null 时省略此字段>"
  },
  "initialPrompt": "<清晰的任务目标 + 必要的关键注意事项（不编排具体步骤）>"
}
```

- **大文本防断连要求**：严禁为防断连或超时而降低思考档位；发生网络断连时先确认已落盘的中间产物再续跑，并提醒其缩小单次工具返回体量、拆成多次小步调用推进。
- 如用户显式指定了其他模型或档位，以用户要求为准。

### 4.3 并行调度与两层子 Agent 分工

在“保持上下文集中”与“积极并行提速”之间把握好平衡：

1. **什么时候集中，什么时候并行**
   - **集中到主执行 Agent**：上下文强相关、连续迭代修改同一批代码/配置的主线任务，留在同一个主执行 Agent 里并由它统一收口正式改动。
   - **默认直接并行**：只要任务包含**多个彼此不相关的独立问题、只读探索、源码排查或独立对照实验**（即使单项不算特别重、只需几分钟），不要打包塞给同一个人串行做——在并发上限内直接拆给不同执行 Agent（复用已有空闲 Agent 或新建）并行推进，既提速又能保持各自上下文干净。
2. **并行数量与防冲突边界**
   - 同时运行的 Paseo 执行 Agent **默认不超过 3 个**（避免管理失焦或资源打满），并全部登记到 `.note/goal-<target>/agents.md`。
   - 并行执行 Agent 复用已有 Workspace（不乱建新 Workspace），且明确各自边界：辅助并行路以只读调研、独立实验或独立产物目录为主，不与主执行 Agent 并发修改同一批正式文件，结果回来后再由主执行 Agent 统一收口落地。
3. **两层分工：善用执行 Agent 的内部子 Agent**
   - 个人助理通过 Paseo 拉起的是**执行 Agent**；执行 Agent 内部还可以按需启动它自己的**真正子 Agent（subagent）**。
   - 遇到以下三类场景时，可在派活时顺带提醒执行 Agent 善用内部子 Agent（提醒即可，不替它编排细节）：
     - **需要干净上下文防污染**：例如独立盲评、方案复核、交叉验错，避免受主上下文先入为主影响准确率；
     - **大文本探索调研**：需要读大量代码、文档或日志时，交给内部子 Agent 消化提炼，防止撑爆或污染主上下文；
     - **大批量数据处理**：需要对几十上百条样本/日志逐条核验或批处理时，用内部子 Agent 分片提速。

## 5. 派活原则与前置 Prompt 确认

派活核心是**同步目标、信任执行、不做包工头**：
- **只讲目标和背景**：把用户想办成什么事交代清楚，**不要替执行 Agent 规划具体方案、拆解 1-2-3 执行步骤或细写备份/脚本/校验流程**，让它自主规划和验证。
- **轻提关键注意事项**：只补充真正不能踩的红线或关键边界（例如“保留某台机器的原有配置”“不要改对外接口”），点到为止。
- **前置 Prompt 确认（关键）**：只要任务可能涉及写或改 `prompt / skill / 自然语言规则 / 用户可见文案`，必须在派活时顺带明确一句：
  > “涉及编写或修改 prompt / skill / 自然语言规则的内容，在正式落盘和跑验证前，先将拟稿发给我确认定稿。”
- **文档与命名提醒**：涉及新增文档或新概念命名时，顺带提一句用通俗直观的词句、不要生造抽象新词。
- **大文本防断连提醒**：任务若涉及大量文本、日志或批量案例处理，顺带提醒执行 Agent 把工具返回拆成小份、控制单次返回体量、通过多次小步工具调用逐步推理落盘（绝不通过降思考档位来防断连）。

## 6. 等待、纠偏与验货

- **异步等待与收尾汇报时机**：保持 `notifyOnFinish: true`，派完活直接结束本轮等 Paseo 通知，不要循环轮询（超长任务按需用 `create_heartbeat`）。
  - 允许执行 Agent 正常通过 `send_agent_prompt` 发回任务关键信息，不要刻意禁止它发消息；
  - 收到执行 Agent 的 `agent-message` 时区分场景：若是中途提问、方向讨论或前置确认 Prompt，正常立刻处理；若是任务收尾结论，先用 `get_agent_status` 确认它是否仍在 `running`——若尚未结束，**不要把这次消息作为向用户输出最终结论的时机**，先收下信息并可提前验货，等 `finished` 通知到达后，再把 `agent-message` 里的关键信息与最终回复整合，**一次性把完整的最终结论呈现给用户**，确保聊天最末尾永远是干净完整的最终结论文档，不再出现“执行 Agent 本轮已正式结束”之类的状态尾巴。
- **发送时机**：
  - 执行 Agent 正在运行且只是普通补充：不传 `activeTurnBehavior`（默认 `queue`），避免打断推理；
  - 确认方向明显跑偏需要及时止损：使用 `steer` 或 `interrupt`，并在纠偏后把“分歧点 + 讨论结论 + 后续动作”同步周知给用户。
- **轻量验货**：执行 Agent 汇报完成时，不要照搬原文转发：
  1. 对照用户原始目标，确认没有遗漏需求或悄悄降级；
  2. 确认执行 Agent 给出了真实验证结果（如测试结果、命令输出或关键改动摘要）；
  3. 抽查新增文档和命名里有没有不常用的黑话或生造抽象词；如有，先让执行 Agent 确认该词是否为新出现：
     - **纯新词**：直接要求整改；
     - **存量仅零星出现（一两处）**：旧处不动，在新增处补一句名词变更说明，后续统一用直观说法；
     - **存量已大量使用**：暂不改动，避免为了改词引发代码重构；
  4. 去掉抽象词和防御性套话，用简洁清晰的人话向用户汇报。
