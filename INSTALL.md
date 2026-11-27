# PaseoAgent 安装与自动配置手册（面向 AI）

本文档专门写给负责帮助用户初始化的 AI 阅读，用于通过 Paseo 官方机制**自动开启 Paseo Tools、确认必需技能、配置 `Linguist` / `Builder` / `Grandmaster` 三类执行 Profile，并在 Paseo 中为本仓库准备好置顶工作区与“个人助理”标签页**。整个配置过程均支持热更新，**无需重启正在运行的 Paseo daemon**。

## 1. 配置前确认：先由用户指定三类模型

在写入配置前，确保用户已经明确指定了三类角色各自使用的模型（个人助理本身默认使用 `Linguist` 模型，除非用户另行指定；如果用户尚未指定，先查询本机可用 Provider 与模型列表，请用户确认后再动手配置，不要替用户擅自决定）：

| Profile 名称 | 角色定位与明确短板 | 参考配置举例 |
|---|---|---|
| **`Linguist`** | **全局默认首选 · 强语言 / 弱编程**：常规任务、调研分析、大上下文文本/日志/评论理解、质量归因、方案梳理、Prompt/Skill 优化（含取数+语义分析混合任务）。**短板**：编程与复杂代码任务交付质量较差，不靠谱做硬核工程落地。 | `provider`: `antigravity`<br>`model`: `gemini-4-neon`<br>`modeId`: `full-access`<br>`thinkingOptionId`: `high`<br>`options`: 不传（省略） |
| **`Builder`** | **工程默认首选 · 强编程 / 弱语言**：代码探索、写/改代码、写脚本、常规排障、单测构建与跨机运维同步。**短板**：自然语言表达与大上下文文本理解弱，写 Prompt/文档易堆砌黑话与叠甲。 | `provider`: `opencode`<br>`model`: `openai/gpt-6.2-sol-fast`<br>`modeId`: `build`<br>`thinkingOptionId`: `xhigh`<br>`options`: `{ "auto_accept": true }` |
| **`Grandmaster`** | **终极攻坚 · 极强逻辑 / 弱语言（绝非万能）**：仅用于复杂算法设计、硬核系统/并发疑难排障、重度跨模块架构重构，或 `Builder` 搞不定的硬骨头。**短板**：同 `Builder`，语言与文本理解弱、绝不用于文本任务且成本更高，平时不轻易启用。 | `provider`: `opencode`<br>`model`: `openai/gpt-6.2-astra-fast`<br>`modeId`: `build`<br>`thinkingOptionId`: `xhigh`<br>`options`: `{ "auto_accept": true }` |

> 上表右侧为作者本机的实际配置示例，用户可自由替换为自己环境中任意可用的 Provider 与强模型（具体 `provider`、`model`、`modeId`、`thinkingOptionId` 与 `options` 须以用户本机实际支持的值为准）。

## 2. 确认目标 Paseo Daemon

执行配置前，先确认目标 daemon 的状态与实际 `PASEO_HOME` 目录（默认通常为 `~/.paseo` 或 `%USERPROFILE%\.paseo`；如操作非默认目录，下方所有命令统一加上 `--home "<PASEO_HOME>"`）：

```bash
paseo daemon status --json
```

## 3. 开启 Paseo Tools 并确认 `paseo` / `paseo-help` 技能已启用

### 3.1 开启 Paseo Tools（MCP 工具注入）

必须开启 `daemon.mcp.injectIntoAgents`，个人助理才能调用 Paseo MCP 工具（此开关仅控制 MCP 工具注入，不会强制开启全部内置 Skill）：

```bash
paseo daemon config set daemon.mcp.injectIntoAgents true --json
```

- 该命令会自动校验、保存并触发热重载，**无需重启 daemon**。
- **生效说明**：新启动的 Agent 会直接带入 Paseo Tools；已启动的旧 Agent 不会当场获得新工具，需新开会话或在空闲时经用户同意执行 `paseo agent reload <agentId>`（仅重载该单个 Agent，不重启 daemon）。

### 3.2 检查并确保 `paseo` 与 `paseo-help` 技能可用（不强开全部 Skill）

Paseo 的内置技能开关独立于 MCP 工具，由 `agents.skills.selection` 控制（仅支持 `{ "mode": "all" }` 或 `{ "mode": "custom", "skills": [...] }`，未配置等同于 `all`，全不选为空数组 `[]`）。**不需要也不应该强迫用户开启全部内置技能**，只需确保 `paseo` 和 `paseo-help` 两项已启用并实际安装：

1. 先查看当前选择，并核对目标 Provider 的技能目录中 `paseo/SKILL.md` 与 `paseo-help/SKILL.md` 是否已安装可读：
   ```bash
   paseo daemon config get agents.skills.selection --json
   ```
2. 按当前状态分情况安全处理：
   - **已选中且两项 `SKILL.md` 均已安装可读**：无需任何改动；
   - **`mode: "custom"` 且缺少 `paseo` 或 `paseo-help`**：保留用户原有 `skills` 列表，仅追加缺失的项后热写入（例如原本为空数组时变为仅含这两项）：
     ```bash
     paseo daemon config set agents.skills.selection '{"mode":"custom","skills":["paseo","paseo-help"]}' --json
     ```
   - **未配置（`set: false`）或 `mode: "all"`，但两项技能尚未安装落盘**：不要自动替用户同步整套内置技能，先请用户确认是改为仅安装 `paseo` 与 `paseo-help`（推荐），还是按默认全选安装。
3. **确保技能文件实际安装**：`paseo daemon config set agents.skills.selection` 只更新选中配置，**不会顺带安装技能文件**。若检查发现 `paseo` 或 `paseo-help` 的 `SKILL.md` 尚未落盘到目标 Provider 的技能目录，需通过 Paseo 的 **Settings → Host → Tools & prompts → Paseo skills**（或同款技能同步接口）执行安装（如提示存在待确认的删除项，不要自动确认），安装后新开 Agent 或空闲重载即可发现。

## 4. 自动配置三类 Agent Profiles

### 4.1 先读取现有配置，避免覆盖用户原有条目

`daemon.agentProfiles` 是**整表替换**，没有按 ID 局部更新。写入前必须先读出现有数组，按稳定 `id` 合并新增或更新，完整保留用户原有的其他 Profile：

```bash
paseo daemon config get daemon.agentProfiles --json
```

### 4.2 组装完整 JSON 数组并热写入

将用户指定的 `provider`、`model`、`modeId`、`thinkingOptionId`、`options`（若该 Provider 无此项则省略该字段）填入下方模板，并与用户原有其他 Profile 合并：

```json
[
  {
    "id": "agent_profile_linguist",
    "name": "Linguist",
    "provider": "<user-linguist-provider>",
    "model": "<user-linguist-model>",
    "modeId": "<unattended-mode-id>",
    "thinkingOptionId": "<thinking-option-id>",
    "notes": "常规任务与非代码任务的默认首选；涉及大量文本/日志/评论/对话的语义理解、质量归因、方案梳理、Prompt/Skill 优化时优先使用（含“取数 + 语义分析”混合任务，只要瓶颈在语义理解就选它）。\n优点：大上下文理解深，自然语言表达清晰、贴合人类习惯。\n缺点：编程能力与复杂代码任务交付质量较差，代码根因定位与硬核工程落地不靠谱；且单次工具返回超大文本时易网络断连（须分批小步读取、控制单次返回体量，不要降思考档位）。"
  },
  {
    "id": "agent_profile_builder",
    "name": "Builder",
    "provider": "<user-builder-provider>",
    "model": "<user-builder-model>",
    "modeId": "<build-mode-id>",
    "thinkingOptionId": "<thinking-option-id>",
    "options": { "auto_accept": true },
    "notes": "代码探索、代码编写、脚本、常规排障与运维同步的默认首选，但不适合大量非结构化文本理解任务。\n优点：逻辑严谨、执行靠谱、代码交付质量高、速度快。\n缺点：不擅长自然语言表达与大上下文语义分析，写 Prompt、文档或规则容易堆砌黑话、法条腔和防御性叠甲；涉及文案或大文本分析应交 Linguist 或由个人助理把关。"
  },
  {
    "id": "agent_profile_grandmaster",
    "name": "Grandmaster",
    "provider": "<user-grandmaster-provider>",
    "model": "<user-grandmaster-model>",
    "modeId": "<build-mode-id>",
    "thinkingOptionId": "<thinking-option-id>",
    "options": { "auto_accept": true },
    "notes": "仅用于极其困难的代码与深度推理任务（如复杂算法设计、硬核系统/并发疑难排障、重度跨模块架构重构，或 Builder 难以解决的硬骨头），绝不用于大量文本理解任务。\n优点：深度推理与逻辑能力最强，复杂代码完成和交付质量极高。\n缺点：同 Builder，自然语言表达与大上下文语义分析差，写 Prompt/文档易生造黑话与叠甲，且耗时成本更高，非极难任务优先用 Builder。"
  }
]
```

使用官方 CLI 写入并热加载：

```bash
paseo daemon config set daemon.agentProfiles '<合并后的完整 JSON 数组>' --json
```

**字段与安全注意事项**：
- **必填字段**：`id`（任意唯一字符串；更新已有条目时保留原 `id`）、`name`、`provider`（仅填 Provider ID，与 `model` 分开写）。
- **常用可选字段**：`model`、`modeId`、`thinkingOptionId`、`options`（以 Provider 特性 ID 为键的对象，无则省略）、`notes`（供个人助理选型时阅读）。
- **切勿与 Paseo 桌面端设置页同时编辑**：避免旧列表覆写刚提交的配置。
- **如采用直接修改 `<PASEO_HOME>/config.json` 兜底**：daemon 不会自动监听这两项改动，修改文件后必须显式运行 `paseo daemon reload --json` 触发热加载；任何情况下都**不要自动重启 daemon**。

## 5. 核验配置生效

配置完成后执行只读核验，确认 `injectIntoAgents` 为 `true`、`paseo` 与 `paseo-help` 技能已启用且 `SKILL.md` 可读、三个 Profile 均已热生效：

```bash
paseo daemon config get daemon.mcp.injectIntoAgents --json
paseo daemon config get agents.skills.selection --json
paseo daemon config get daemon.agentProfiles --json
```

若当前会话已具备 Paseo MCP 工具，也可直接调用 `list_profiles` 确认三个 Profile 及其 `notes` 已被运行时正常读出。

## 6. 落位工作区、创建个人助理标签并打开对话

配置生效后，不要只停在“已完成”的文字提示上——接着帮用户在 Paseo 里准备好直接可用的个人助理入口，让用户一眼就能开始对话：

### 6.1 复用或创建 `PaseoAgent` 本地工作区

`create` 命令不会按路径去重，因此必须先查再建。先列出已有工作区，按 `cwd` 精确匹配用户 clone 后的 `PaseoAgent` 绝对路径：

```bash
paseo workspace ls --background --json
```

（若通过 Paseo MCP，调用 `list_workspaces({ includeBackground: true })`。）

- **已有同路径的活跃可见 `local` 工作区**：直接复用其 `workspaceId`，不重复新建（若只存在后台工作区，不要把它当成不存在，应明确复用或请用户确认）；
- **不存在匹配项时**：创建本地工作区（Paseo 会自动复用或创建对应 Project，无需先执行 `project create`）：
  ```bash
  paseo workspace create --isolation local --path "<repo-abs-path>" --title "PaseoAgent · 个人助理" --no-background --json
  ```
  （若通过 Paseo MCP，调用 `create_workspace({ isolation: "local", path: "<repo-abs-path>", title: "PaseoAgent · 个人助理", background: false })`。）

### 6.2 置顶该工作区

Paseo 支持将 workspace 置顶（daemon 持久化 `pinnedAt`），但目前仅提供 SDK/WebSocket 接口，CLI 与 Paseo MCP 尚无置顶命令：

- **可调用 `@getpaseo/client` SDK 时**：直接执行 `await workspace.setPinned(true)`（对应 WebSocket `workspace.pin.set.request`）完成置顶；
- **仅使用 CLI 或 MCP 时**：使用清晰的标题（如 `PaseoAgent · 个人助理`），并在最终汇报中提醒用户在 Paseo 侧栏右键该工作区点击 **Pin workspace** 置顶。

### 6.3 在该工作区创建预设好模型的“个人助理”标签页

默认使用用户指定的 **`Linguist`** 模型配置作为个人助理（除非用户另行指定）。即使当前执行安装的 AI 已经位于该工作区，也建议新建一个干净的“个人助理”标签页，确保它直接带入刚刚开启的 Paseo Tools 与技能：

- **首选：通过 `@getpaseo/client` SDK（零多余回合，配置最完整）**
  使用普通客户端（不传 `parent`）调用 `workspace.agents.create`，**省略 `prompt` 与附件**，即可直接创建独立的空闲待命标签页：
  ```ts
  await workspace.agents.create({
    title: "个人助理",
    config: {
      provider: "<linguist-provider>/<linguist-model>",
      modeId: "<linguist-mode-id>",
      thinkingOptionId: "<linguist-thinking-option-id>",
      // 仅在有额外选项时传入 options
    },
  });
  ```
- **备选 A：仅有 `paseo` CLI（安装 AI 在 Paseo 外部）**
  `paseo run` 需要非空 prompt 且没有通用的 `options` 参数：
  - 若助理模型无需额外 `options`，可发送一条极简就绪语创建标签页（省略无值的可选参数）：
    ```bash
    paseo run --workspace "<workspaceId>" --title "个人助理" --provider "<linguist-provider>/<linguist-model>" --mode "<linguist-mode-id>" --thinking "<linguist-thinking-option-id>" --no-wait --json "只回复「个人助理已就绪，请直接发需求」。不要调用工具、修改文件或启动其他 Agent。"
    ```
  - 若助理模型必须依赖额外 `options`，不要静默丢弃选项；跳过自动建标签，直接打开工作区并在汇报中提示用户新建标签时选好配置。
- **备选 B：通过 Paseo MCP（安装 AI 在 Paseo 内部且未走 SDK）**
  调用 `create_agent`（传入 `workspaceId`、`title: "个人助理"`、`provider: "<linguist-provider>/<linguist-model>"`、`settings: { modeId, thinkingOptionId, features: options }` 以及上述极简就绪语）。**注意**：从 Agent 内通过 MCP 创建的会话默认会挂载为当前安装 Agent 的子 Agent，创建完成后执行 `paseo agent detach "<agentId>"` 将其转为独立个人助理。

### 6.4 打开对话并明确指引用户开始使用

1. **自动定位到对话页**：拿到 `agentId` 后，如运行在本地图形桌面环境，执行以下命令直接唤起并切到该助理标签页（仅建了工作区、未建 Agent 时可用 `paseo "<repo-abs-path>"`）：
   ```bash
   paseo agent open "<agentId>"
   ```
2. **输出清晰的上手提醒**：汇报配置结果时，明确告知用户工作区名称（`PaseoAgent · 个人助理`）、标签名称（`个人助理`）以及直达链接 `paseo://h/<serverId>/agent/<agentId>`（便于无桌面或远程环境定位）；如果本轮未通过 SDK 自动置顶，提醒用户在侧栏点击 **Pin workspace**；最后提示用户直接切进该标签页发送需求即可开始使用。
