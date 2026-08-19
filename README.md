<p align="center">
  中文 | <a href="./README_EN.md">English</a>
</p>

# 🚀 OpenClaw-Matrix：自主多智能体矩阵引擎

> **基于 OpenClaw 构建的自驱动、状态集中、自举式多智能体协作协议。**

[![OpenClaw](https://img.shields.io/badge/Powered%20by-OpenClaw-blueviolet)](https://github.com/OpenClaw/OpenClaw)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Status](https://img.shields.io/badge/Status-V5.6--Stable-orange)](https://github.com/)

## 🌟 项目简介

**OpenClaw-Matrix** 是一个专为复杂 AI 任务设计的高性能编排框架。通过实现 **"集中式状态总线"** 与 **"严格角色解耦"**，本系统从根源上消除了多智能体工作流中的常见陷阱——上下文污染、逻辑漂移与无限循环。

系统的核心是 `state.md` 文件，它充当所有智能体的"共享黑板"。**最突出的特性是其自举能力：只需提供架构文档，OpenClaw 便会自行读取、解析并完成整个矩阵的配置。**

---

## 🧠 核心设计理念（V5.6 协议）

1. **Main 单写状态中心**：`state.md` 是任务的唯一主状态总线。仅 Main 可物理写入、更新、重置或归档；所有 Worker（Researcher、Executor、Debugger、Judge）均只读总线，完成后向 Main 返回 `WORKER_COMPLETED` 回执及建议的 `State_Patch`。Main 验证后串行提交，Worker 可并行计算。
2. **严格沙盒隔离**：每个 Assignment 绑定独立 Session/Run、明确文件范围和验收标准。并行 Executor 使用独立工作树、分支或产物目录，不能直接争用同一工作树。
3. **摘要与证据驱动**：状态总线只存状态、摘要、证据索引和 Digest；大产物与日志放在任务文件夹，敏感信息不得进入回执或状态。
4. **渐进式两层 DAG**：FULL 模式先建立 Research DAG，再随已验证研究结果通过 `DAG_Update` 增量建立 Delivery DAG；部分研究完成即可安全解锁无争议节点。
5. **动态模型发现与按 Assignment 分配**：从部署时实际可用模型动态发现并建立能力画像，按每个 Assignment 的需求向量匹配，高能力/高价模型只用于高杠杆节点，轻量模型优先覆盖研究、格式校验与低风险执行，从而节省 Token 与成本。
6. **两级重试与熔断**：失败先进行最多 3 次执行链重试，再退回 Researcher 进行最多 3 次实质修订；不影响无依赖链路。
7. **生命周期自动化**：从 `New_Task_Flag` 检测新任务，到 Judge `PASS` 后自动归档与重置，系统自主管理完整生命周期，仅在破坏性/不可逆/外部/隐私敏感操作时才需用户确认。
8. **运行时可验证调度**：静态配置不等于已启用。配置后必须真实调用非 Main 子 Agent，以 Assignment、Session/Run、回执和 Main 写入证据证明调度有效。

---

## 🏗️ 矩阵角色定义

| 角色组 | ID | 职责 | 状态权限 | 关键产出 |
| :--- | :--- | :--- | :--- | :--- |
| **调度中心 (Main)** | `main` | 分流评分、两层 DAG、Assignment、回执验证、串行状态写入、归档 | 唯一可写 `state.md` | 状态、调度与验收记录 |
| **研究池 (Researcher)** | `res_1~3` | 需求/边界、方案/接口、风险/验收研究 | 只读 | 研究结果、`State_Patch` |
| **执行池 (Executor)** | `exe_1~3` | 隔离实现、工具执行、测试与产物生成 | 只读状态；仅改授权产物 | 产物、证据、`State_Patch` |
| **调试池 (Debugger)** | `dbg_1~3` | 逻辑、边界、性能、安全与回归检查 | 只读 | 质检回执、`State_Patch` |
| **集成角色 (Integration)** | 动态节点 | 独占集成产物、冲突仲裁、合并与回归 | 只读状态；独占集成范围 | 集成产物与回执 |
| **仲裁员 (Judge)** | `judge` | 独立完成门控与最终裁决 | 只读 | `PASS/REJECTED` 回执 |

各职能池并发上限：Researcher 最多 3；Executor 最多 3；Debugger 最多 3；同一集成产物仅 1 个有效 Integration 节点；Judge 默认 1 个实例。实际并发为 READY 节点数、可用 Agent 数、池上限三者最小值。

---

## 🚀 快速上手与自举配置

### 1. 前置条件

- 确保已安装 [OpenClaw](https://github.com/OpenClaw/OpenClaw) 并完成 LLM API 配置。
- 克隆仓库：

```bash
git clone https://github.com/WEI-6/OpenClaw-Multi-Agent-Matrix.git
cd OpenClaw-Multi-Agent-Matrix
```

### 2. 配置 OpenClaw 会话映射

参考当前 OpenClaw 文档，在 `openclaw.json` 中为各角色（`main`、`res_1`、`res_2`、`exe_1`、`exe_2`、`dbg_1`、`dbg_2`、`dbg_3`、`judge`）添加 `agents.list` 条目，并为每个子 Agent 指定独立的 workspace 路径。

> ⚠️ 配置格式以当前安装版本的 OpenClaw schema 为准；仅使用 OpenClaw 支持的配置字段。

### 3. 自举启动指令

完成上面的 OpenClaw 会话映射后，向 **Main Agent** 发送以下指令，让 Main 按 V5.6 协议初始化并执行一次可验证的自举流程：

> *"请读取并解析根目录下的 `Architecture_V5.6.md` 文件。根据此文档，检查我的 OpenClaw Matrix 会话配置，初始化 `state.md` 总线，按 `/prompts` 中的角色提示词建立调度约束，并进入新任务的 INIT 状态。"*

**Main 应按协议完成或核验：**
- 核验已配置的 Agent ID、Session 映射与可用模型，并将模型发现结果绑定到具体 Assignment。
- 确认各角色已在 OpenClaw 配置中使用 `/prompts` 目录下的专属提示词（`main.md`、`res.md`、`exe.md`、`dbg.md`、`judge.md`）。
- 在工作区初始化或检查 `state.md` 任务总线模板。
- Main 在每轮读取 `state.md` 时按协议处理 `New_Task_Flag`；OpenClaw 本身不因文件存在而自动轮询或启用 Matrix。
- 核验 Main 具备已配置的受限子 Agent 调度权限；权限与 allowlist 需由 OpenClaw 配置提供，不能由读取架构文档自动授予。
- 配置完成后，由 Main 发起一次运行时验收，确认子 Agent 的独立 Session、Main 单写状态总线和结构化回执均真实生效。

### 4. 运行时调度验收

仅生成角色名单、提示词和配置文件，**不代表 Matrix 已经投入运行**。没有真实的子 Agent Session 和共享黑板写回，系统仍然只是静态部署。

配置完成后，Main Agent 必须执行一次最小验收任务：

1. 在 `state.md` 中创建唯一 `Task_ID`，写入明确的 `[Dispatch Plan]`。
2. 按计划真实调用至少一个非 Main 子 Agent（不得由 Main 模拟角色输出）。
3. 确认该角色运行在独立 Session 中，记录实际的 Agent ID、Session ID 或 Run ID。
4. 子 Agent 只读 `state.md`，完成后仅向其授权章节返回 `WORKER_COMPLETED` 回执与 `State_Patch`，不直接写入总线。
5. Main 校验回执、重算 `result_digest`，验证通过后串行写入 `State_Patch`。
6. Judge 独立读取证据并返回 `PASS / FAIL` 裁决；任一证据缺失均视为运行时调度未启用。

验收至少应同时满足：
- `state.md` 中存在当前任务的唯一 `Task_ID` 和最近更新时间。
- 至少一个已分配的非 Main Agent 存在真实 Session / Run。
- 子 Agent 回执中的 `Task_ID` 与共享黑板一致，且结果已由 Main 写入同一份 `state.md`。
- Main 或 Judge 已记录最终验收结论。

### 5. 首次运行初始化

首次启动子 Agent 时，OpenClaw 会为其 workspace 初始化 `MEMORY.md` 与 `memory/` 目录。若子 Agent 从未启动过，其记忆索引不会建立——这属于正常现象，启动一次对应会话即可自动完成初始化。活跃运行文件（如 `workspace/state.md`）应保留在运行者 workspace 中，不应提交进源码仓库。

---

## 📂 目录结构

```
<project-root>/
├── prompts/            # 各角色专属协议提示词（main.md, res.md, exe.md, dbg.md, judge.md）
├── Architecture_V5.6.md  # V5.6 全局架构方案（自举配置的"可信源"文档）
├── LICENSE
└── README.md / README_EN.md
```

> `workspace/state.md`、`workspace/history/` 与可选的任务级 `MAM_state.md` 等运行时文件由使用者在 workspace 中生成或维护，不应视为仓库随附源码文件。

---

## 🔄 协议核心机制

### 任务分流与评分模式

Main 在接收任务后，先应用硬覆盖，再按多维加法评分确定模式，并将每个加分项及理由写入 `state.md`：

| 模式 | 分数 | 适用场景 |
|---|---|---|
| `MAIN_ONLY` | 0–2 | 纯对话、只读查询、微小可逆操作；Main 直接处理，不派发子 Agent |
| `MINIMAL` | 3–5 | 轻量 Matrix 路径：Main 跳过 Research DAG，直接建立 Delivery DAG 并派发真实非 Main 角色（EXE-* 等）；链路为 Main 分流/模型分配 → Delivery DAG → EXE-* → DBG-*（按风险）→ Integration（按需）→ Judge（按风险）|
| `FULL` | ≥6 | 完整两层 DAG：Research DAG → Delivery DAG → EXE-* → DBG-* → Integration → Judge |

硬覆盖（优先于分数）：用户明确要求 Matrix/独立 QA → `FULL`；需要安全确认 → `WAITING_USER`；纯对话/只读 → `MAIN_ONLY`。

评分维度（加法）：安全与失败影响 `+3`；多文件或跨组件 `+2`；实现与工具执行 `+2`；独立质量验证 `+2`；独立交付物与并行价值 `+2`；集成复杂度 `+2`；研究与需求不确定性 `+1`；长时运行与恢复敏感 `+1`。

### 两层 DAG

FULL 模式采用 Research DAG → Delivery DAG 两层结构：

```
Research DAG:   RES-1 || RES-2 || RES-3
                         ↓ Main 验证每份回执，串行写入
Delivery DAG:   EXE-* → DBG-* → INTEGRATION（按需）→ REGRESSION（按需）→ JUDGE
```

Main 每收到一个有效研究回执即通过 `DAG_Update` 增量解锁 Delivery 节点，无需等待同批全部完成。

### Assignment 与 WORKER_COMPLETED 回执

Main 为每个子任务创建唯一 Assignment（`Task_ID/Subtask_ID/Attempt`）。Worker 完成后返回标准 `WORKER_COMPLETED` JSON 回执，包含：`task_id`、`subtask_id`、`assignment_id`、`attempt`、`role`、`agent_id`、`session_key`、`run_id`、`status`、`target_section`、`result_summary`、`artifacts`、`evidence`、`errors`、`next`、`State_Patch`（Main 应写入目标章节的完整替换 Markdown）、`result_digest`（Main 必须重算验证，不信任 Worker 值）。

Worker 不直接写 `state.md`；Main 校验回执后串行提交 `State_Patch`。

### 并行隔离与集成

并行 Executor 各使用 `<task_folder>/artifacts/<subtask-id>/` 或独立工作树，不得争用同一工作树。所有相关 Executor 完成并退出后，Integration 节点才获得集成范围的独占修改权，并运行跨模块接口、构建与整体行为回归。

### 动态模型发现与按 Assignment 分配

Main 从 OpenClaw 当前配置与 allowlist、用户显式提供的模型清单、目录/元数据发现以及受控最小探测中动态发现可用模型，为每个模型建立能力、成本、运行时与合规画像。

分配时按每个 Assignment 的需求向量（角色、上下文、工具、预算、隐私等）先硬过滤、再打分。高能力/高价模型只分配给高杠杆或高难节点，轻量模型优先用于研究、格式校验和低风险执行，有效节省 Token 与成本。发现来源分层：`configured_allowed` > `targeted_probe` > `catalog_listed`；仅"看得见"不等于"能执行"，runtime 验收必须由真实子 Agent 运行结果证明。

### 两级重试与熔断

计数键为 `Task_ID + Subtask_ID + Stage + Root_Cause_Signature`。第一阶段：最多 3 次执行链重试（Attempt 1→4）；第三次重试仍失败则执行链熔断，退回 Researcher。第二阶段：Researcher 最多 3 次实质修订；第 3 次修订后仍失败进入 `WAITING_USER`。无依赖链路不受影响。

### Judge 与完成门控

Judge 仅在所有必要 Executor 与 Debugger 通过、无未解决失败/阻塞/熔断时启动，独立核验交付物、证据、Assignment/Digest 与安全边界，裁决仅为 `PASS` 或 `REJECTED`。

Judge `PASS` 后，Main 原子幂等归档 `state.md` 至 `<workspace>/history/<task-id>/`，更新 `MAM_state.md` 长期记忆，重置活跃状态，报告完成——无需例行用户确认。破坏性/不可逆/外部/隐私敏感操作仍需用户同意。

### MAM（跨任务长期记忆）

`<task_folder>/MAM_state.md` 仅保留跨任务有效的项目概况、原则边界、关键决策、近期变更、验收证据与下一步起点。每个 Task 至多更新一次，且仅在 Judge `PASS` 后、归档前写入并回读验证。MAM 不包含完整 DAG、调度流水、运行日志或凭证。

### 生命周期状态

```
INIT → TRIAGED → DISPATCHING → RUNNING → VERIFYING → JUDGING → ARCHIVING → COMPLETED
```

侧边状态：`WAITING_USER`、`RETRYING`、`RECOVERING`、`CIRCUIT_OPEN`、`FAIL`

---

## 🤝 参与贡献

欢迎为 Matrix 协议贡献力量！无论是优化提示词效率、完善 `state.md` 结构，还是新增专业化角色，欢迎提交 PR。

---

## 📄 许可证

本项目基于 **MIT 许可证** 开源。

---
**为自主协作的未来而生。**
