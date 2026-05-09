**中文** | [English](README_EN.md)

---

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
4. **生命周期自动化**：从新任务检测（通过 `New_Task_Flag`）到用户确认归档，系统自主管理完整的任务生命周期。

---

## 🏗️ 矩阵角色定义

| 角色组 | ID | 职责 | 关键产出 |
| :--- | :--- | :--- | :--- |
| **主控制器** | `Main` | 全局路由、DAG 调度与用户交互 | 调度指令 & 状态更新 |
| **研究池** | `Res_1~3` | 技术调研、架构设计与执行手册编写 | **执行手册** |
| **执行池** | `Exe_1~3` | 代码实现、环境搭建与部署 | **最终交付物** |
| **调试池** | `Dbg_1~3` | 逻辑、边界与性能验证 | **调试报告 / 日志** |
| **仲裁员** | `Judge` | 汇总调试报告并进行最终评估 | **评估结果 / 审批意见** |

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
- 从 `/prompts` 目录为每个角色注入专属提示词。
- 在工作区生成 `state.md` 模板。
- 监听 `New_Task_Flag` 以启动执行流程。

---

## 📂 目录结构

- `/prompts`：包含各角色（RES、EXE、DBG、JUDGE）专属协议的 `.txt` 文件。
- `/workspace`：活跃的 `state.md` 总线与执行环境。
- `/workspace/history`：已完成任务的自动归档目录。
- `Architecture_v5.0.md`：用于自举配置的"可信源"文档。
- `openclaw.json.example`：会话映射配置模板。

---

## 🔄 自愈循环逻辑

当 `Artifacts`（交付物）部分发生错误时：
1. **检测**：`Debugger` 识别故障，并向质量中心写入 `FAIL` 报告。
2. **路由**：`Main` 读取报告，携带具体错误日志重新分配 `Executor`。
3. **升级**：若循环 3 次仍未成功，`Main` 触发"计划重置"，将任务退回 `Researcher` 重新设计方案。

---

## 🤝 参与贡献

欢迎为 Matrix 协议贡献力量！无论是优化提示词效率、完善 `state.md` 结构，还是新增专业化角色，欢迎提交 PR。

---

## 📄 许可证

本项目基于 **MIT 许可证** 开源。

---
**为自主协作的未来而生。**
