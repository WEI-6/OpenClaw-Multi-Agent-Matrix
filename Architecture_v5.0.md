# 🚀 OpenClaw 多 Agent 矩阵协作系统全局架构方案 (v5.0)

---

## 一、 设计目的 (Design Purpose)

本方案旨在通过工程化手段解决大模型在复杂任务中的**上下文污染**、**逻辑漂移**与**低效循环**问题。

* **标准化协作**：定义统一的 Agent 通信协议，使多智能体能够像工业流水线一样精准协作。
* **高可靠性**：引入多重质检（Debugger Pool）与中立仲裁（Judge），确保最终产出物的质量。
* **无缝移植**：通过单一的 `state.md` 文件与标准化的配置文件，实现系统在不同 OpenClaw 实例间的快速部署。
* **全生命周期管理**：涵盖从新任务识别、自动化执行、用户确认到历史归档的全闭环流程。

---

## 二、 设计理念 (Design Philosophy)

1.  **状态中心化 (State Centralization)**
    系统不依赖 Agent 间的直接传话，所有信息流转必须通过中心化的 `state.md`（共享黑板）。这确保了系统的“单一事实来源”。
2.  **职责解耦与隔离 (Decoupling & Isolation)**
    每个 Agent 拥有独立的 Session ID。通过 Prompt 限制其读写权限，确保执行者不被研究过程干扰，调试者专注于结果校验。
3.  **摘要驱动通信 (Summary-Driven)**
    子 Agent 仅向 `state.md` 写入结构化摘要。Main Agent 仅通过摘要进行逻辑决策，极大地节省了 Token 并降低了长上下文导致的幻觉。
4.  **动态拓扑调度 (DAG Routing)**
    系统支持任务的有向无环图（DAG）调度。Main Agent 可根据任务难度动态组合 Agent（如：1个研究员 + 2个执行员并行实现 + 3个调试员交叉验证）。
5.  **闭环自愈与熔断 (Self-Healing & Circuit Breaking)**
    通过“执行-调试-反馈”的小循环实现逻辑自愈。当循环次数达到阈值时触发熔断，强制回退至研究阶段或请求人工介入。

---

## 三、 角色定义与分工 (Role Definition & Division)

| 角色组 | 实例 ID | 核心职责 | Session ID | 输出要求 |
| :--- | :--- | :--- | :--- | :--- |
| **调度中心 (Main)** | `main` | 识别新任务、DAG 路由调度、用户收尾确认 | `main_session` | 路由指令、状态更新 |
| **研究池 (Researcher)** | `res_1~3` | 技术调研、架构设计、编写执行手册 | `res_1~3_id` | **执行计划 (Manual)** |
| **执行池 (Executor)** | `exe_1~3` | 代码/指令编写、环境部署、具体功能实现 | `exe_1~3_id` | **产出物 (Artifacts)** |
| **调试池 (Debugger)** | `dbg_1~3` | 逻辑/边界/性能专项校验、反馈错误日志 | `dbg_1~3_id` | **质检报告 (Debug Feed)** |
| **仲裁员 (Judge)** | `judge_1` | 汇总多份质检报告、执行封版、最优选型 | `judge_id` | **评估结论 (Evaluation)** |

---

## 四、 核心步骤 (Core Steps)

### 1. 任务初始化与识别 (Initialization)
* **新任务判定**：Main Agent 检测 `state.md` 中的 `Status`。若为 `COMPLETED` 或 `New_Task_Flag` 为 `TRUE`，则触发新任务流程。
* **环境重置**：清空当前 Session 缓存，根据模版重新生成 `state.md` 并分配唯一的 `Task_ID`。

### 2. 路由规划 (Routing)
* **组合决策**：Main Agent 分析原始需求，在 `state.md` 中写入 `[Dispatch Plan]`（例如：`res_1 -> exe_1 -> [dbg_1, dbg_2]`）。

### 3. 链式执行与自愈循环 (Execution & Loop)
* **任务流转**：Researcher 产出计划 -> Executor 产出结果 -> Debugger 提交报告。
* **自愈机制**：若 Debugger 返回 `FAIL`，Main 指派对应 Executor 参考错误日志进行修复。
* **熔断处理**：若同一环节循环超过 3 次，系统自动降级，打回 Researcher 重新设计方案。

### 4. 仲裁与封版 (Review & Evaluation)
* **多维评估**：Judge Agent 对比多个 Debugger 的反馈。
* **通过准则**：仅在所有核心指标（逻辑/安全/性能）均显示为 `PASS` 时，输出封版建议。

### 5. 任务收尾与归档 (Completion & Archiving)
* **用户确认**：Main Agent 询问用户：“任务已完成，是否确认结束？”。
* **自动化归档**：
    * 用户确认后，修改 `Status` 为 `COMPLETED`。
    * 系统自动将当前 `state.md` 备份至 `history/` 目录，并以 `Task_ID` 命名。
    * 重置 `state.md` 为初始模版，等待下一条指令。

---

## 五、 OpenClaw 实施 (Implementation)

### 1. 目录结构
```bash
mkdir -p ~/.openclaw/workspace/history
touch ~/.openclaw/workspace/state.md
```

### 2. `state.md` 核心模版
```markdown
# 📂 OpenClaw 任务总线 (State Center)

## 📊 [0] 系统信号 (Signals)
- **Task_ID**: `OC-UUID-TIMESTAMP`
- **Status**: `INIT` 
- **New_Task_Flag**: `TRUE`

## 📈 [1] 调度计划 (Dispatch Plan)
- **拓扑**: res_n -> exe_n -> [dbg_n]
- **当前激活**: `None`

## 📖 [2] 共享工作区 (Blackboard)
### 📝 执行计划 (Research Plan)
### 💻 产出结果 (Artifacts)
## 🧪 [3] 质量中心 (Quality Center)
- **dbg_1 (逻辑)**: `Pending`
- **dbg_2 (边界)**: `Pending`
- **dbg_3 (性能)**: `Pending`

## ⚖️ [4] 仲裁结论 (Evaluation)
- **Judge 状态**: `Pending`
```

### 3. `openclaw.json` 配置映射
```json
{
  "bridges": [
    { "id": "main", "session_id": "main_v5", "is_controller": true },
    { "id": "res_1", "session_id": "res_1" }, 
    { "id": "res_2", "session_id": "res_2" },
    { "id": "exe_1", "session_id": "exe_1" }, 
    { "id": "exe_2", "session_id": "exe_2" },
    { "id": "dbg_1", "session_id": "dbg_1" }, 
    { "id": "dbg_2", "session_id": "dbg_2" },
    { "id": "judge", "session_id": "judge_v5" }
  ],
  "workspace": { 
    "state_file": "./workspace/state.md", 
    "auto_archive": true 
  }
}
```

### 4. 提示词 (Prompt) 策略
* **Main Agent**：必须包含“询问用户是否完成任务”以及“处理 New_Task_Flag”的逻辑逻辑。
* **Worker Agents**：必须遵循“协议化输出”，严禁输出任何非 `### Summary` 或非 `state.md` 章节的内容。
* **权限控制**：通过 System Prompt 强制各 Agent 仅拥有对自身章节的“写”权限和对他人的“读”权限。