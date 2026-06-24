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
5.  **动态模型适配 (Dynamic Model Allocation)**
    系统不假设用户拥有固定模型列表。配置 Matrix 时，Main Agent 必须先读取用户 API 当前可用模型，再根据模型价格、上下文长度、能力偏向与子 Agent 任务类型，为每个子 Agent 分配合适模型。
6.  **闭环自愈与熔断 (Self-Healing & Circuit Breaking)**
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

### 2. 模型发现与子 Agent 分配 (Model Discovery & Assignment)
* **可用模型拉取**：Main Agent 在创建或刷新 Matrix 子 Agent 前，必须先读取用户当前 API / OpenClaw 配置可调用的模型列表。
* **模型画像建立**：对每个模型记录或推断其价格、上下文长度、输出能力、推理能力、代码能力、速度、稳定性、工具调用适配度等维度。
* **角色需求匹配**：根据子 Agent 任务类型分配模型。例如研究员偏向长上下文与推理，执行员偏向代码能力与稳定性，调试员偏向逻辑/边界/性能检查能力，Judge 偏向综合判断能力。
* **成本约束**：高价模型只分配给高杠杆角色或高难任务，避免所有子 Agent 无差别使用昂贵模型。若模型价格未知，应记录风险并采用保守分配。
* **写入配置**：分配结果应写入子 Agent 配置，并在 Matrix 元数据中记录分配依据，便于用户审计和后续调整。

### 3. 路由规划 (Routing)
* **组合决策**：Main Agent 分析原始需求，在 `state.md` 中写入 `[Dispatch Plan]`（例如：`res_1 -> exe_1 -> [dbg_1, dbg_2]`）。

### 4. 链式执行与自愈循环 (Execution & Loop)
* **任务流转**：Researcher 产出计划 -> Executor 产出结果 -> Debugger 提交报告。
* **自愈机制**：若 Debugger 返回 `FAIL`，Main 指派对应 Executor 参考错误日志进行修复。
* **熔断处理**：若同一环节循环超过 3 次，系统自动降级，打回 Researcher 重新设计方案。

### 5. 仲裁与封版 (Review & Evaluation)
* **多维评估**：Judge Agent 对比多个 Debugger 的反馈。
* **通过准则**：仅在所有核心指标（逻辑/安全/性能）均显示为 `PASS` 时，输出封版建议。

### 6. 任务收尾与归档 (Completion & Archiving)
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

### 3. 动态模型分配协议 (Dynamic Model Allocation Protocol)

在用户根据本项目文件配置 Matrix 时，系统必须先完成模型发现，再创建或更新子 Agent。

#### 3.1 输入：用户可用模型列表
Main Agent 应从以下来源读取模型列表：

* OpenClaw 当前模型配置或模型目录。
* 用户 API Provider 返回的可用模型。
* 用户手动提供的模型清单。
* 项目提供的 `openclaw.json.example` 或其他配置模板中的模型声明。

模型列表不应在架构文档中写死，因为不同用户的 API 权限、模型供应商、价格和上下文窗口都可能不同。

#### 3.2 模型画像维度
每个模型应尽量建立如下画像：

| 维度 | 说明 |
| :--- | :--- |
| **调用价格** | 输入/输出 token 单价、是否高价、是否未知价格。 |
| **上下文长度** | 能处理的最大上下文窗口，适合长文档/大代码库任务。 |
| **能力偏向** | 推理、代码、写作、调试、多模态、工具调用等偏向。 |
| **输出能力** | 最大输出长度、结构化输出稳定性。 |
| **速度与稳定性** | 延迟、失败率、限流风险。 |
| **用户偏好** | 用户指定的默认模型、禁用模型、预算限制。 |

若价格、能力或上下文数据缺失，必须标记为 `unknown`，不得伪造数据。

#### 3.3 子 Agent 任务类型需求
模型分配应根据角色任务类型进行匹配：

| 子 Agent 类型 | 优先模型特征 | 成本策略 |
| :--- | :--- | :--- |
| `Main` | 稳定调度、长上下文、结构化决策 | 中高能力即可，避免长期占用最高价模型。 |
| `Res_1~3` | 长上下文、强推理、资料综合、架构设计 | 主研究员可用强模型，辅助研究员可用性价比模型。 |
| `Exe_1~3` | 代码能力、工具调用稳定性、错误修复能力 | 主执行员用强代码模型，备用执行员用低成本模型。 |
| `Dbg_1` | 逻辑推理、需求覆盖检查 | 可使用强推理模型。 |
| `Dbg_2` | 边界条件、安全、异常路径 | 使用均衡模型，重点稳定和细致。 |
| `Dbg_3` | 性能、回归、格式、资源检查 | 优先便宜快速模型。 |
| `Judge` | 综合判断、冲突仲裁、最终封版 | 可分配最高杠杆模型，但应受预算限制。 |

#### 3.4 分配原则

1. **先发现模型，再分配角色**：不得预设某个固定模型必然存在。
2. **按任务类型匹配能力**：研究偏长上下文与推理，执行偏代码与工具，调试偏检查能力，仲裁偏综合判断。
3. **按成本控制并发**：高价模型默认只给少数高杠杆角色，不能批量分配给所有子 Agent。
4. **按任务难度动态升降级**：简单任务可整体使用低成本模型；复杂任务才提升关键角色模型等级。
5. **保守处理未知价格**：未知价格模型不应被大量并发使用，除非用户明确允许。
6. **支持用户覆盖**：用户可以指定预算、禁用模型、固定某角色模型或限制高价模型数量。

#### 3.5 推荐分配流程

```text
Fetch Available Models
        ↓
Normalize Model Metadata
        ↓
Score by Cost / Context / Capability / Stability
        ↓
Build Role Demand Profiles
        ↓
Assign Models to Sub Agents
        ↓
Validate Budget and Coverage
        ↓
Write Agent Config + Matrix Metadata
```

伪代码：

```pseudo
models = fetch_user_available_models()
profiles = normalize(models)

for model in profiles:
    model.score = evaluate(
        cost = model.price,
        context = model.context_window,
        capability = model.capability_bias,
        stability = model.runtime_reliability
    )

roles = build_role_profiles([main, res_1..3, exe_1..3, dbg_1..3, judge])
assignments = match_models_to_roles(profiles, roles, user_budget_policy)

validate(assignments)
persist(assignments)
```

#### 3.6 配置与审计输出

分配结果应写入实际创建的子 Agent 配置，例如：

```json
{
  "id": "res_1",
  "model": {
    "primary": "<selected-model-id>",
    "fallbacks": []
  }
}
```

同时，Matrix 应在自己的元数据中记录模型选择依据，便于用户审计：

```json
{
  "model_allocation": {
    "source": "user_available_models",
    "policy": {
      "prefer_low_cost": true,
      "premium_concurrency_limit": 1
    },
    "assignments": {
      "res_1": {
        "model": "<selected-model-id>",
        "reason": "long context + strong reasoning for research planning"
      }
    },
    "warnings": [
      "Some model prices are unknown; conservative allocation was used."
    ]
  }
}
```

#### 3.7 校验要求

* 用户可用模型列表不能为空。
* 每个即将创建或激活的子 Agent 必须有明确模型。
* 高价模型数量不得超过用户预算策略。
* 未知价格模型必须记录风险。
* Matrix 自定义元数据不得写入 OpenClaw 不支持的顶层配置字段。
* 配置写入后必须验证 OpenClaw 配置有效。

### 4. `openclaw.json` 配置映射
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

### 5. 提示词 (Prompt) 策略
* **Main Agent**：必须包含“询问用户是否完成任务”以及“处理 New_Task_Flag”的逻辑逻辑。
* **Worker Agents**：必须遵循“协议化输出”，严禁输出任何非 `### Summary` 或非 `state.md` 章节的内容。
* **权限控制**：通过 System Prompt 强制各 Agent 仅拥有对自身章节的“写”权限和对他人的“读”权限。