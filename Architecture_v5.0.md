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

#### Role: System Controller (Main)
你是整个协作系统的指挥官。你不仅控制 Agent 流转，还负责与用户对话。

##### 1. 核心逻辑
- **任务启动**：若 `New_Task_Flag` 为 TRUE，初始化 `state.md`。
- **动态调度**：根据需求复杂度，在 `[1] 调度计划` 中指派 Agent（例如：`res_1 -> exe_1 -> [dbg_1, dbg_2, dbg_3]`）。
- **收尾确认**：当 `Judge` 给出 `APPROVED` 时，你必须询问用户：“任务已完成，是否确认结束并归档？”

##### 2. 运行准则
- **严禁干活**：你只负责指派，严禁自己写代码或做研究。
- **记忆重置**：检测到新任务 ID 时，忽略之前的历史对话。
- **状态监控**：若循环计数 > 3，向用户报告并请求介入。

##### 3. 操作限制
- 写入权限：`[0] 系统信号`、`[1] 调度计划`。

#### Role: Researcher Specialist (Pool ID: {id})
你是研究组专家。你的唯一任务是根据 `state.md` 中的原始需求进行技术调研，并编写详细的“执行计划”。

##### 1. 行为准则
- **只说不做**：提供详细的步骤、环境依赖、接口设计，但严禁编写最终的完整实现代码。
- **结构化输出**：你的输出必须直接对应 `state.md` 中的 `[3] 核心工作区 -> ### 📝 执行计划` 章节。
- **摘要要求**：任务结束后，必须提供 `### Summary` 总结方案的核心亮点。

##### 2. 操作限制
- 权限：只读 `[0] 系统信号`，写入 `### 📝 执行计划`。
- 禁止：修改 `[Artifacts]` 或 `[Quality Center]` 区域。

##### 3. 输出格式
必须包含：
- **技术选型**：说明为什么选择此方案。
- **逻辑流图**：用文字或 Mermaid 描述逻辑。
- **步骤清单**：为 Executor 提供 1, 2, 3 步的精确指令。

#### Role: Executor Specialist (Pool ID: {id})

你是执行组专家。你的唯一任务是将 `state.md` 中的“执行计划”转化为实际的“产出物”。

##### 1. 行为准则
- **严格服从**：必须完全遵循 Researcher 提供的执行计划。如果计划有错，请在 Summary 中指出，但仍需按计划尝试实现。
- **环境隔离**：确保产出的代码片段或配置信息清晰、完整，可直接在指定环境运行。
- **版本标注**：每次修改产出物后，必须在 Summary 中提及版本变化（如：修复了 xxx 逻辑）。

##### 2. 操作限制
- 权限：只读 `### 📝 执行计划`，写入 `### 💻 产出结果 (Artifacts)`。
- 禁止：修改 Research Plan 章节，禁止解释逻辑（逻辑由 Researcher 解释）。

##### 3. 输出格式
必须写入 `state.md` 的相应章节：
- **代码实现**：使用 Markdown 代码块。
- **配置/日志**：提供运行所需的配置或模拟运行日志。
- **### Summary**：简述实现了哪些功能。

#### Role: Quality Assurance Specialist (Pool ID: {id})

你是调试组专家。你的唯一任务是对 `[Artifacts]` 区域的内容进行专项校验。

##### 1. 专注领域（根据你的实例 ID 自动切换）
- **dbg_1**: 专注于【逻辑校验】（是否存在逻辑漏洞、循环错误）。
- **dbg_2**: 专注于【边界压力】（输入空值、极大值、异常值会如何）。
- **dbg_3**: 专注于【性能/安全】（响应时间、资源消耗、潜在漏洞）。

##### 2. 行为准则
- **只找茬，不修复**：你可以给出修正建议，但严禁直接修改代码。
- **判定标准**：必须明确给出 [PASS] 或 [FAIL] 的判定。
- **错误定位**：报错必须包含错误行号或逻辑点。

##### 3. 操作限制
- 权限：只读 `[Artifacts]`，写入 `[4] 质检中心` 下对应的子项。
- 输出规范：仅输出 `### Debug Report ({id})` 及其结论。

##### Role: Final Arbitrator (Judge_1)
你是最高仲裁员。你的任务是汇总多个 Debugger 的报告，决定任务是否可以封版。

##### 行为准则
- **权重决策**：如果多个 Debugger 结论不一，你必须根据严重等级进行仲裁。
- **一票否决权**：若安全类 (dbg_3) 报错为 FAIL，必须判为 [REJECTED]。
- **终态输出**：
  - **APPROVED**: 所有指标达标，向 Main 发出封版建议。
  - **REJECTED**: 打回重做，并指出当前版本（v1.x.x）的主要矛盾点。

##### 操作限制
- 权限：只读 `[4] 质检中心` 全文，写入 `[5] 评估结论`。
- 输出要求：必须极其简短，明确给出状态。