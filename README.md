# PaseoAgent

基于 [Paseo](https://paseo.sh) 的个人助理 Agent 规范仓库。在这里启动的个人助理充当**用户与底层执行 Agent 之间的沟通与质量隔离层**——你只需用自然语言把目标说清楚，剩下的任务派发、多路并行、方向纠偏、文案/Prompt 把关和结果提炼，全部交给个人助理来协调。

## 为什么需要一层个人助理

- **扬长避短**：擅长写代码、深度推理的模型往往不擅长说人话（容易写抽象黑话、法条腔和防御性套话）；由表达清晰的模型当个人助理，专门负责双向转译与自然语言把关。
- **把控全局不跑偏**：执行 Agent 专注干活，个人助理守住原始目标、及时纠偏、把复杂过程压缩成结论直给用户。
- **多任务并发不撞车**：支持在本目录下同时运行多个个人助理，每个助理与执行 Agent 保持 1 对 N 归属，不污染 Paseo 原生标签。

## 快速上手

### 1. 克隆仓库并加入 Paseo

```bash
git clone https://github.com/Zzzia/PaseoAgent.git
```

将本地 `PaseoAgent` 目录作为 Project 添加到 Paseo，并在该工作区启动一个**自然语言理解与沟通能力强**的模型作为个人助理。

### 2. 开启 Paseo Tools 并配置三类执行 Profile

个人助理需要通过 Paseo 工具调度其他 Agent，且建议在 Paseo 中预先配好三类执行 Profile（具体选用哪家 Provider / 模型由你自由决定）：

| Profile 名称 | 角色定位与适用场景 | `notes` 写法参考 |
|---|---|---|
| **`Linguist`** | **全局默认首选 · 强文本与大上下文**：常规任务、调研分析、大上下文文本/日志/评论理解、质量归因、方案评估、Prompt/Skill 优化，以及“取数 + 语义分析”混合任务 | 擅长大上下文理解与自然语言表达，默认首选；不适合硬核代码根因排查与复杂代码落地。处理大文本时分批小步读取、控制单次工具返回体量，不降思考档位。 |
| **`Builder`** | **工程默认首选 · 日常开发与运维**：写代码、改需求、写脚本、常规代码排障、单测构建、跨机运维同步等 | 逻辑严谨、代码交付质量高、速度快，工程类任务默认首选；不擅长自然语言表达与大上下文文本分析，涉及 Prompt/文档文案交个人助理把关。 |
| **`Grandmaster`** | **终极攻坚 · 极难推理专用**：平时不轻易启用；仅用于复杂算法设计、硬核系统/并发疑难排障、重度跨模块架构重构，或 `Builder` 搞不定的硬骨头 | 深度推理能力最强，仅在极其复杂困难任务时启用；耗时与成本较高，非极难任务优先用 `Builder`，同样不用于大文本分析。 |

#### 方式 A：在 Paseo 界面手动配置
1. 打开 Paseo 设置，开启 **Paseo Tools**（允许向 Agent 注入 Paseo MCP 工具，否则助理无法调用 Paseo 技能）。
2. 在 **Agent Profiles** 中新增 `Linguist`、`Builder`、`Grandmaster` 三个配置，选择你常用的模型并在 `notes` 中填入分工说明。

#### 方式 B：直接让 AI 帮你命令行自动配置（支持热更新，无需重启 Paseo daemon）

你可以直接把本节交给任意能操作本机的 AI 自动完成配置（如需操作非默认目录，以下命令统一加上 `--home "<PASEO_HOME>"`；可用 `paseo daemon status --json` 确认实际目录）：

1. **开启 Paseo Tools 开关**：
   ```bash
   paseo daemon config set daemon.mcp.injectIntoAgents true --json
   ```
   - 该项支持热生效，无需重启 daemon。已启动的旧 Agent 不会当场获得新工具，新开 Agent 或对空闲旧 Agent 执行 `paseo agent reload <agentId>` 即可生效。

2. **配置三类 Agent Profiles**：
   - `daemon.agentProfiles` 为**整表替换**，写入前请先读取现有配置，按稳定 `id` 合并修改，保留用户原有的其他条目：
     ```bash
     paseo daemon config get daemon.agentProfiles --json
     ```
   - 根据用户本机已启用的 Provider 与模型（可先询问用户偏好，或查 `paseo` 可用列表），按以下字段结构组装完整 JSON 数组并热写入：
     ```json
     [
       {
         "id": "agent_profile_linguist",
         "name": "Linguist",
         "provider": "<your-provider>",
         "model": "<your-model>",
         "modeId": "<unattended-mode-id>",
         "thinkingOptionId": "high",
         "notes": "常规任务与非代码任务的默认首选；擅长大上下文文本/日志/评论理解、质量归因、方案梳理与 Prompt/Skill 优化（含取数+语义分析混合任务）。不适合硬核代码根因排查与复杂代码落地；处理大文本须分批小步读取、控制单次返回体量，不要降思考档位。"
       },
       {
         "id": "agent_profile_builder",
         "name": "Builder",
         "provider": "<your-provider>",
         "model": "<your-model>",
         "modeId": "<build-mode-id>",
         "thinkingOptionId": "xhigh",
         "options": { "auto_accept": true },
         "notes": "代码探索、代码编写、脚本、常规排障与运维同步的默认首选。逻辑严谨、代码交付质量高、速度快；不擅长自然语言表达与大上下文文本分析，涉及 Prompt/文档文案交个人助理把关。"
       },
       {
         "id": "agent_profile_grandmaster",
         "name": "Grandmaster",
         "provider": "<your-provider>",
         "model": "<your-model>",
         "modeId": "<build-mode-id>",
         "thinkingOptionId": "xhigh",
         "options": { "auto_accept": true },
         "notes": "仅用于极其困难的代码与深度推理任务（复杂算法设计、硬核系统/并发疑难排障、重度跨模块架构重构，或 Builder 难以解决的硬骨头）。深度推理最强但成本更高，非极难任务优先用 Builder，不用于大文本理解任务。"
       }
     ]
     ```
     ```bash
     paseo daemon config set daemon.agentProfiles '<完整 JSON 数组>' --json
     ```
   - **字段说明**：`id`（任意唯一字符串）、`name`、`provider` 为必填；`model`、`modeId`、`thinkingOptionId`、`options`（Provider 特性开关，无则省略）、`notes`（给个人助理阅读的选型说明）为可选。
   - **若选择直接编辑 `<PASEO_HOME>/config.json`**：Paseo 不会自动监听这两项变更，改完必须执行 `paseo daemon reload --json` 触发热加载（不会重启 daemon）；切勿与桌面端设置页同时编辑同一配置。

> 即使暂时未配置 Profile 或未填写 `notes`，个人助理也会参考 [AGENTS.md](AGENTS.md) 中的三类分工，通过 Paseo 工具选择当前可用的合适模型。

### 3. 开始使用

- **直接提目标**：直接告诉个人助理想办成的事，助理会自动处理轻量文本任务，或按需拉起对应类型的执行 Agent（遇到多个独立问题会自动并行）。
- **接管已有 Agent**：如果你已经在 Paseo 里开了一个执行 Agent，直接把它的 `agentId` 发给个人助理即可接管。
- **本地任务记录**：每个任务的目标、上下文和 Agent 归属会自动保存在仓库内的 `.note/goal-<任务简写>/` 目录下（已加入 `.gitignore`，不会提交到 Git）。

## 目录说明

- [AGENTS.md](AGENTS.md)：个人助理核心守则（启动即生效）
- [docs/paseo-ops.md](docs/paseo-ops.md)：Paseo 调度、归属防撞车、目标守护与验货实操指南
