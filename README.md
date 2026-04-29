# 🚀 State-Bus Agent Matrix

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![OpenClaw](https://img.shields.io/badge/Powered%20by-OpenClaw-blue.svg)](https://github.com/openclaw/openclaw)

> **全自动 AI 矩阵**：基于状态总线（State-Bus）的动态拓扑调度协作系统，告别长上下文污染与低效死循环。

## 🌟 核心亮点 (Core Features)

* **自配置 (Self-Configuring)**：支持有向无环图 (DAG) 动态路由，主控节点 (Main) 根据任务自动分配研究员、执行员与调试员，无需人工硬编码工作流。
* **状态总线 (State-Bus)**：采用 `state.md` 共享黑板模式，Agent 间不直接通信，杜绝信息畸变。
* **隔离与质检 (Isolation & QA)**：执行与验证分离，多重 Debugger 并行找茬，Judge 负责最终仲裁，实现质量闭环。
* **自动熔断 (Circuit Breaking)**：内置最大重试阈值，超限自动拦截并通知人类，避免 token 无限燃烧。

## 🏗️ 系统架构图 (Architecture)

```mermaid
graph TD
    User([🙋 人类用户]) -->|下发任务| Main[🤖 Main Agent\n(系统指挥官)]
    
    subgraph State_Bus [状态总线 State.md]
        Signal[系统信号 Status]
        Plan[调度计划 Plan]
        Work[共享工作区 Workspace]
        QC[质量中心 QA]
        Eval[仲裁结论 Eval]
    end

    Main -->|解析更新| Signal
    Main -->|路由分发| Plan
    
    Plan -->|触发| Res[🔍 Researcher\n(产出执行计划)]
    Res -->|写入| Work
    
    Work -->|读取| Exe[💻 Executor\n(产出代码/结果)]
    Exe -->|写入| Work
    
    Work -->|读取| Dbg1[🧪 Debugger_1\n(逻辑校验)]
    Work -->|读取| Dbg2[🧪 Debugger_2\n(边界压力)]
    Work -->|读取| Dbg3[🧪 Debugger_3\n(安全性能)]
    
    Dbg1 -->|写入| QC
    Dbg2 -->|写入| QC
    Dbg3 -->|写入| QC
    
    QC -->|读取| Judge[⚖️ Judge\n(终极仲裁)]
    Judge -->|封版或打回| Eval
    
    Eval -->|反馈| Main
    Main -->|归档确认| User
```

## 📂 仓库结构 (Repository Structure)

```text
.
├── Architecture_v5.0.md      # 核心架构与设计文档
├── README.md                 # 项目说明文档
└── prompts/                  # 角色分离的独立 Prompt
    ├── main.md               # 调度中心 (Main)
    ├── res.md                # 研究池 (Researcher)
    ├── exe.md                # 执行池 (Executor)
    ├── dbg.md                # 调试池 (Debugger)
    └── judge.md              # 仲裁员 (Judge)
```

## 🚀 快速开始 (Getting Started)

1. 克隆本仓库到您的 OpenClaw 工作区：
   ```bash
   git clone https://github.com/weiyingji/State-Bus-Agent-Matrix.git
   ```
2. 参考 `Architecture_v5.0.md` 配置您的 `openclaw.json`。
3. 注入 `prompts/` 目录下的角色提示词至对应的 Agent 实例。
4. 启动 OpenClaw，给 Main Agent 丢入一个复杂任务，感受矩阵的力量。