<p align="center">
  中文 | <a href="./README_EN.md">English</a>
</p>

# 🚀 OpenClaw-Matrix：自主多智能体矩阵引擎

> **基于 OpenClaw 构建的自驱动、状态集中、自举式多智能体协作协议。**

[![OpenClaw](https://img.shields.io/badge/Powered%20by-OpenClaw-blueviolet)](https://github.com/OpenClaw/OpenClaw)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Status](https://img.shields.io/badge/Status-v5.0--Stable-orange)](https://github.com/)

## 🌟 项目简介

**OpenClaw-Matrix** 是一个专为复杂 AI 任务设计的高性能编排框架。通过实现 **"集中式状态总线"** 与 **"严格角色解耦"**，本系统从根源上消除了多智能体工作流中的常见陷阱——上下文污染、逻辑漂移与无限循环。

系统的核心是 `state.md` 文件，它充当所有智能体的"共享黑板"。**最突出的特性是其自举能力：只需提供架构文档，OpenClaw 便会自行读取、解析并完成整个矩阵的配置。**

---

## 🧠 核心设计理念（v5.0 协议）

1. **状态集中化（`state.md`）**：智能体之间不进行直接通信。所有信息流通过单一的结构化 Markdown 文件流转，确保"单一可信源"。
2. **严格沙盒隔离**：每个智能体（研究员、执行者、调试员）在其独立的 OpenClaw 会话中运行，保持"干净"的上下文，专注于各自的职责。
3. **摘要驱动执行**：智能体必须输出结构化摘要。主控制器仅根据这些摘要做出路由决策，从而大幅节省 Token 消耗并减少幻觉。
4. **生命周期自动化**：从新任务检测（通过 `New_Task_Flag`）到 Judge PASS 后的自动内部归档，系统自主管理完整的任务生命周期，仅在破坏性/不可逆/外部/隐私敏感操作时才需用户确认。
5. **运行时可验证调度**：Agent 注册、角色提示词（`.md` 格式）和工作区文件只代表静态配置完成。Main 必须按照 `Dispatch Plan` 真正调用被分配的子 Agent，为其创建独立 Session，并验证结果已回写同一份 `state.md`。

---

## 🏗️ 矩阵角色定义

| 角色组 | ID | 职责 | 关键产出 |
| :--- | :--- | :--- | :--- |
| **主控制器** | `Main` | 全局路由、DAG 调度与用户交互 | 调度指令 & 状态更新 |
| **研究池** | `Res_1~3` | 技术调研、架构设计与执行手册编写 | **执行手册** |
| **执行池** | `Exe_1~3` | 代码实现、环境搭建与部署 | **最终交付物** |
| **调试池** | `Dbg_1~3` | 逻辑、边界与性能验证 | **调试报告 / 日志** |
| **仲裁员** | `Judge` | 汇总调试报告并进行最终评估 | **评估结果 / 裁决** |

---

## 🚀 快速上手与自举配置

本项目最强大之处在于其**自动配置**能力。按以下步骤让系统自行完成构建：

### 1. 前置条件
- 确保已安装 [OpenClaw](https://github.com/OpenClaw/OpenClaw) 并完成 LLM API 配置。
- 准备项目目录：
```bash
git clone https://github.com/YourUsername/OpenClaw-Matrix.git
cd OpenClaw-Matrix
mkdir -p workspace/history
```

### 2. 自举启动指令
启动 OpenClaw 会话后，向 **Main Agent** 发送以下指令：

> *"请读取并解析根目录下的 `Architecture_v5.0.md` 文件。根据此文档，自主配置我的 OpenClaw Matrix 会话、初始化 `state.md` 总线、注入各角色专属提示词，并进入新任务的 INIT 状态。"*

**系统将自动完成：**
- 注册所有 Agent ID 与 Session ID。
- 从 `/prompts` 目录为每个角色注入专属提示词（`.md` 格式）。
- 在工作区生成 `state.md` 模板。
- 监听 `New_Task_Flag` 以启动执行流程。
- 为 Main 开启受限的子 Agent 调度权限，并仅允许调用已注册的 Matrix 角色。
- 配置完成后执行一次运行时验收，确认子 Agent 的独立 Session、共享黑板读写和结构化回执均真实生效。

### 3. 运行时调度验收

仅生成角色名单、提示词、工作区和配置文件，**不代表 Matrix 已经投入运行**。如果没有真实的子 Agent Session 和共享黑板写回，系统仍然只是静态部署，并未启用运行时调度器。

配置完成后，Main Agent 必须执行一次最小验收任务：

1. 在 `state.md` 中创建唯一 `Task_ID`，写入明确的 `[Dispatch Plan]`。
2. 按计划真实调用至少一个非 Main 子 Agent，而不是由 Main 模拟该角色输出。
3. 确认该角色运行在独立 Session 中，并记录实际的 Agent ID、Session ID 或 Run ID。
4. 子 Agent 必须先读取同一绝对路径下的 `state.md`，再仅向其授权章节写入结构化结果。
5. Main 再次读取 `state.md`，确认 `Task_ID` 一致、文件修改时间已更新且角色结果真实存在。
6. 由 Judge 或 Main 对以上证据给出 `PASS / FAIL` 结论；任一证据缺失均视为运行时调度未启用。

验收至少应同时满足以下条件：

- `state.md` 中存在当前任务的唯一 `Task_ID` 和最近更新时间。
- 至少一个已分配的非 Main Agent 存在真实 Session / Run。
- 子 Agent 回执中的 `Task_ID` 与共享黑板一致。
- 子 Agent 的结果已经写入同一份共享黑板，而不是只返回在聊天消息中。
- Main 或 Judge 已记录最终验收结论。

> **重要说明：** `openclaw.json.example` 中的角色映射和 `matrix-config` 一类描述文件不会自动成为调度器。实际部署时仍需按照当前 OpenClaw 版本配置 Main 的子 Agent 调用权限、Session 可见性及必要的 Agent-to-Agent 权限，并通过真实调用完成验收。

---

## 📂 目录结构

- `/prompts`：包含各角色（`main.md`、`res.md`、`exe.md`、`dbg.md`、`judge.md`）的专属协议提示词。
- `/workspace`：活跃的 `state.md` 总线与执行环境。
- `/workspace/history`：已完成任务的自动归档目录。
- `Architecture_v5.0.md`：用于自举配置的"可信源"文档。
- `openclaw.json.example`：会话映射配置模板。

---

## 🔄 自愈循环逻辑

当 `Artifacts`（交付物）部分发生错误时：
1. **检测**：`Debugger` 识别故障，并向 `## [3] 质量中心` 写入 `FAIL` 报告（含根因签名和严重等级）。
2. **路由**：`Main` 读取报告，携带具体错误日志重新分配 `Executor`，保留已通过节点。
3. **升级**：若同一阶段/根因签名连续失败 **3 次**，触发熔断（`CIRCUIT_OPEN`）：停止重调度，将任务退回 `Researcher` 要求实质性修订方案，或进入 `WAITING_USER` 请求人工介入；仅在有可验证进展或方案实质变更后才重置计数器。

### 任务分流规则

Main 在接收任务后，先应用硬覆盖，再按加法评分确定模式，并记录到 `state.md`：

| 模式 | 分数 | 适用场景 |
|---|---|---|
| `MAIN_ONLY` | 0–2 | 纯对话、只读查询、微小可逆操作 |
| `MINIMAL` | 3–5 | 单专家 + 按需验证 |
| `FULL` | ≥6 | 完整 Researcher → Executor → Debugger → Judge DAG |

硬覆盖（优先于分数）：用户明确要求 Matrix/独立 QA → `FULL`；需要安全确认 → `WAITING_USER`；纯对话/只读 → `MAIN_ONLY`。

### 生命周期状态

`INIT → TRIAGED → DISPATCHING → RUNNING → VERIFYING → JUDGING → ARCHIVING → COMPLETED`

侧边状态：`WAITING_USER`、`RETRYING`、`RECOVERING`、`CIRCUIT_OPEN`、`FAIL`

### 节点账本

每个调度节点在 `[1] 调度计划` 中维护：角色、依赖、状态（`BLOCKED_BY_DEPENDENCY / READY / RUNNING / PASS / FAIL / RETRYING / CIRCUIT_OPEN`）、授权章节、Agent ID、Session Key/Run ID、重试计数、时间戳、回执和写回证据。Main 仅调度 `READY` 节点；下游节点在前置节点 `PASS` 后解锁。

### 完成门控与自动归档

完成门控要求：交付物满足验收标准；所有必要检查通过；账本中每个必要节点均为 `PASS`；无未解决阻塞或熔断；所有回执/Task_ID/写回证据通过验证；Judge 返回 `PASS`；结构与安全约束成立。

Judge `PASS` 后，Main 立即执行内部原子性幂等归档（以 `Task_ID` 命名），记录路径和校验和，重置活跃状态，并报告完成——**无需**例行用户确认。破坏性/不可逆操作、外部发布、凭证或隐私敏感访问、权限/安全边界变更**仍需**用户同意。

---

## 🤝 参与贡献

欢迎为 Matrix 协议贡献力量！无论是优化提示词效率、完善 `state.md` 结构，还是新增专业化角色，欢迎提交 PR。

---

## 📄 许可证

本项目基于 **MIT 许可证** 开源。

---
**为自主协作的未来而生。**
