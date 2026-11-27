# PaseoAgent 安装与自动配置手册（面向 AI）

本文档专门写给负责帮助用户初始化的 AI 阅读，用于通过 Paseo 官方机制**自动开启 Paseo Tools 并配置 `Linguist` / `Builder` / `Grandmaster` 三类执行 Profile**。整个过程均支持热更新，**无需重启正在运行的 Paseo daemon**。

## 1. 配置前确认：先由用户指定三类模型

在写入配置前，确保用户已经明确指定了三类角色各自使用的模型（如果用户尚未指定，先查询本机可用 Provider 与模型列表，请用户确认后再动手配置，不要替用户擅自决定）：

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

## 3. 自动开启 Paseo Tools 开关

必须开启 `daemon.mcp.injectIntoAgents`，个人助理才能加载 Paseo MCP 工具与技能：

```bash
paseo daemon config set daemon.mcp.injectIntoAgents true --json
```

- 该命令会自动校验、保存并触发热重载，**无需重启 daemon**。
- **生效说明**：新启动的 Agent 会直接带入 Paseo Tools；已启动的旧 Agent 不会当场获得新工具，需新开会话或在空闲时经用户同意执行 `paseo agent reload <agentId>`（仅重载该单个 Agent，不重启 daemon）。

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

配置完成后执行只读核验，确认 `injectIntoAgents` 为 `true` 且三个 Profile 均已热生效：

```bash
paseo daemon config get daemon.mcp.injectIntoAgents --json
paseo daemon config get daemon.agentProfiles --json
```

若当前会话已具备 Paseo MCP 工具，也可直接调用 `list_profiles` 确认三个 Profile 及其 `notes` 已被运行时正常读出。完成后告知用户结果（若旧会话需要立即获得 Paseo Tools，提醒用户新开会话或重载该 Agent）。
