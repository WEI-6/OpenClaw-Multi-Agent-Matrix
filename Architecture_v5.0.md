# 🚀 OpenClaw 多 Agent 矩阵协作系统全局架构方案 (v5.0)

---

## 一、 设计目的 (Design Purpose)

本方案旨在通过工程化手段解决大模型在复杂任务中的**上下文污染**、**逻辑漂移**与**低效循环**问题。

* **标准化协作**：定义统一的 Agent 通信协议，使多智能体能够像工业流水线一样精准协作。
* **高可靠性**：引入多重质检（Debugger Pool）与中立仲裁（Judge），确保最终产出物的质量。
* **无缝移植**：通过单一的 `state.md` 文件与标准化的配置文件，实现系统在不同 OpenClaw 实例间的快速部署。
* **全生命周期管理**：涵盖从新任务识别、自动化执行、Judge PASS 后自动归档的全闭环流程；仅在破坏性/不可逆/外部/隐私敏感操作时才需用户确认。

---

## 二、 设计理念 (Design Philosophy)

1.  **状态中心化 (State Centralization)**
    系统不依赖 Agent 间的直接传话，所有信息流转必须通过中心化的 `state.md`（共享黑板）。这确保了系统的"单一事实来源"。
2.  **职责解耦与隔离 (Decoupling & Isolation)**
    每个 Agent 拥有独立的 Session ID。通过 Prompt 限制其读写权限，确保执行者不被研究过程干扰，调试者专注于结果校验。
3.  **摘要驱动通信 (Summary-Driven)**
    子 Agent 仅向 `state.md` 写入结构化摘要。Main Agent 仅通过摘要进行逻辑决策，极大地节省了 Token 并降低了长上下文导致的幻觉。
4.  **动态拓扑调度 (DAG Routing)**
    系统支持任务的有向无环图（DAG）调度。Main Agent 可根据任务难度动态组合 Agent（如：1个研究员 + 2个执行员并行实现 + 3个调试员交叉验证）。
5.  **动态模型适配 (Dynamic Model Allocation)**
    系统不假设用户拥有固定模型列表。配置 Matrix 时，Main Agent 必须先读取用户 API 当前可用模型，再根据模型价格、上下文长度、能力偏向与子 Agent 任务类型，为每个子 Agent 分配合适模型。
6.  **闭环自愈与熔断 (Self-Healing & Circuit Breaking)**
    通过"执行-调试-反馈"的小循环实现逻辑自愈。同一阶段/根因签名连续失败 **3 次**触发熔断，强制回退至研究阶段或请求人工介入；仅在有可验证进展或方案实质变更后才重置计数器。
7.  **运行时可验证性 (Runtime Verifiability)**
    Agent ID、角色 Prompt、Workspace 和 Session 映射只属于静态部署。系统必须由 Main 按 DAG 真实调用被分配的子 Agent，并以独立 Session、统一 `Task_ID`、共享黑板写回和最终验收记录证明运行时调度已经生效。

---

## 三、 角色定义与分工 (Role Definition & Division)

| 角色组 | 实例 ID | 核心职责 | Session ID | 输出要求 |
| :--- | :--- | :--- | :--- | :--- |
| **调度中心 (Main)** | `main` | 分流/路由/DAG 调度，`MAIN_ONLY` 模式下可直接处理微小可逆任务 | `main_session` | 路由指令、状态更新 |
| **研究池 (Researcher)** | `res_1~3` | 技术调研、架构设计、编写执行手册 | `res_1~3_id` | **执行计划 (Research Plan)** |
| **执行池 (Executor)** | `exe_1~3` | 代码/指令编写、环境部署、具体功能实现 | `exe_1~3_id` | **产出物 (Artifacts)** |
| **调试池 (Debugger)** | `dbg_1~3` | 逻辑/边界/性能专项校验、反馈错误日志 | `dbg_1~3_id` | **质检报告 (Debug Feed)** |
| **仲裁员 (Judge)** | `judge_1` | 汇总多份质检报告、执行完成门控、输出最终裁决 | `judge_id` | **评估结论 (Evaluation)** |

---

## 四、 核心步骤 (Core Steps)

### 1. 任务初始化与分流 (Initialization & Triage)

**新任务判定**：Main Agent 通过 `New_Task_Flag = TRUE` 或用户新请求判定新任务；**不得** 仅因 `Status = COMPLETED` 单独推断新任务。

**环境重置**：分配唯一 `Task_ID`，重置生命周期至 `INIT`，根据模版重新生成 `state.md`。

**确定性分流（先应用 Override，再按加法评分）：**

```
Override 检查（优先于分数）:
  IF 用户明确要求 Matrix 或独立 QA  → mode = FULL
  IF 需要安全确认（破坏性/不可逆/外部/隐私/凭证/权限变更）→ status = WAITING_USER
  IF 纯对话 / 只读 / 无工作区变更   → mode = MAIN_ONLY
  ELSE → 计算加法评分:
    score += 3  if 破坏性/外部/隐私敏感/安全影响
    score += 2  if 多文件/跨组件/架构/配置迁移
    score += 2  if 实现/工具执行
    score += 2  if 需要独立 QA/安全验证
    score += 1  if 研究/模糊需求
    score += 1  if 长时运行/恢复敏感
    mode = MAIN_ONLY  if score 0–2
    mode = MINIMAL    if score 3–5
    mode = FULL       if score ≥ 6
```

**模式行为边界：**
- `MAIN_ONLY`：Main 可直接执行，无需调度专家 Agent。
- `MINIMAL`：Main 编排一个真实专家 Agent + 按需验证（Judge 仅在需要独立仲裁时才必须）。
- `FULL`：完整 Researcher → Executor → Debugger → Judge DAG，Main 仅路由与验证，严禁代替任何专家完成其专属工作。

在 `state.md` `[0] 系统信号` 中记录 `Matrix_Score`、`Matrix_Mode`、`Decision_Reasons`；在 `[1] 调度计划` 中写入 DAG 拓扑和完整节点账本。

### 2. 模型发现与子 Agent 分配 (Model Discovery & Assignment)

* **可用模型拉取**：Main Agent 在创建或刷新 Matrix 子 Agent 前，必须先读取用户当前 API / OpenClaw 配置可调用的模型列表。
* **模型画像建立**：对每个模型记录或推断其价格、上下文长度、输出能力、推理能力、代码能力、速度、稳定性、工具调用适配度等维度。
* **角色需求匹配**：根据子 Agent 任务类型分配模型。例如研究员偏向长上下文与推理，执行员偏向代码能力与稳定性，调试员偏向逻辑/边界/性能检查能力，Judge 偏向综合判断能力。
* **成本约束**：高价模型只分配给高杠杆角色或高难任务，避免所有子 Agent 无差别使用昂贵模型。若模型价格未知，应记录风险并采用保守分配。
* **写入配置**：分配结果应写入子 Agent 配置，并在 Matrix 元数据中记录分配依据，便于用户审计和后续调整。

### 3. 路由规划与显式调度 (Routing & Explicit Dispatch)

* **组合决策**：Main Agent 在 `state.md` 中写入 `[Dispatch Plan]`（例如：`res_1 -> exe_1 -> [dbg_1, dbg_2]`）和完整节点账本。
* **依赖门控**：Main 仅调度 `READY` 节点（所有前置依赖为 `PASS`）；并行节点需所有声明的前置依赖均已 `PASS`。
* **显式调度**：通过 OpenClaw 已注册的显式 Agent ID 真实调用子 Agent，创建独立 Session。严禁 Main 在自身 Session 中模拟 Researcher、Executor、Debugger 或 Judge 的输出。
* **调度证据**：每个被激活角色在 `state.md` 中记录 Agent ID、Session ID 或 Run ID、开始/完成时间、负责章节与结果摘要。
* **事件驱动续行**：Main 使用推送式完成事件续行，不得忙轮询。

### 4. 链式执行与自愈循环 (Execution & Self-Healing Loop)

* **任务流转**：Researcher 产出计划 → Executor 产出结果 → Debugger 提交报告。
* **总线同步**：每个子 Agent 在开始前必须读取同一绝对路径下的 `state.md` 并核对 `Task_ID`；完成后仅更新自身授权章节。Main 根据写回结果决定下一个 DAG 节点。
* **回执验证**：收到回执后，Main 重新读取 `state.md`，依次核验：`Task_ID` 一致、预期角色/Session/Run 一致、仅授权章节被写入、结果状态、新鲜写回证据；全部通过后解锁下游节点。
* **Session 隔离**：每次角色调用必须产生可审计的非 Main Session / Run。不同角色不得共享 Main 的会话上下文，也不得仅依赖聊天转述代替共享黑板。
* **自愈机制**：若 Debugger 返回 `FAIL`，Main 指派对应 Executor 参考错误日志（含根因签名、严重等级、是否瞬时）进行修复，保留已通过节点。
* **熔断处理**：同一阶段/根因签名连续失败 **3 次**，触发熔断（`CIRCUIT_OPEN`）：停止重调度；若安全则退回 Researcher 要求实质性修订方案，否则进入 `WAITING_USER`；**仅在** 有可验证进展或方案实质变更后才重置签名计数器。

### 5. 恢复与重建 (Recovery & Reconciliation)

网关重启、Session 丢失、超时或 Main 中断时，进入 `RECOVERING` 状态：

* 仅从 `state.md` 重建状态，不得依赖聊天记忆。
* 调度前先核对活跃运行以避免重复调度。
* 接受有效的迟到回执；将孤立运行标记后在同一 `Task_ID` 下重试。
* **严禁** 仅凭聊天记忆宣告完成。

### 6. 仲裁与完成门控 (Review & Completion Gate)

* **多维评估**：Judge Agent 独立读取 `state.md` 中所有必要节点的证据，对比多个 Debugger 的反馈。
* **完成门控谓词**（全部为真方可 `PASS`）：
  1. 交付物/验收标准已满足
  2. 所有必要检查/测试通过
  3. 账本中每个必要节点均为 `PASS`
  4. 无未解决阻塞或熔断（`CIRCUIT_OPEN`）
  5. 所有回执 Task_ID / Session ID / 写回证据通过验证
  6. 结构约束成立（无新增/删除/移动/重命名文件）
  7. 安全约束成立（无未授权的权限/凭证/外部操作）
* **裁决输出**：`PASS`（原 `APPROVED` 统一规范为 `PASS`）或 `REJECTED`，写入 `## [4] 评估结论 -> ### judge`。
* **一票否决**：`CRITICAL` 级别问题直接判为 `REJECTED`。

### 7. 自动归档 (Automatic Archiving)

Judge `PASS` 后，Main 立即执行：

1. 以 `Task_ID` 为名执行内部原子性幂等归档，将当前 `state.md` 备份至 `workspace/history/` 目录。
2. 记录归档路径和校验和（或等效证据）至 `state.md` `[5] 归档` 章节。
3. 重置 `state.md` 为初始模版，`Status` 设为 `COMPLETED`，`New_Task_Flag` 设为 `FALSE`。
4. 向用户报告完成——**无需**例行用户确认。

**以下情况仍必须先获得用户同意后才可归档：**
- 破坏性/不可逆操作
- 外部发布/消息/事务
- 凭证或隐私敏感访问
- 权限/安全边界变更
- 任何用户明确设置的审批门

### 8. 配置后运行时验收 (Post-Configuration Runtime Acceptance)

* **目的**：防止系统只创建角色名单、提示词和目录，却没有真正启用运行时调度。
* **最小验收拓扑**：配置完成后执行 `main -> res_1 -> judge -> main`，也可使用与当前部署规模等价的最小链路。
* **验收步骤**：
    * Main 归档旧状态，在 `state.md` 中创建唯一 `Task_ID` 和验收专用 `[Dispatch Plan]`。
    * Main 真实调用 `res_1`，要求其在独立 Session 中读取共享黑板，并仅更新 `res_1` 对应章节。
    * Main 核对共享黑板的绝对路径、`Task_ID`、修改时间与角色结果。
    * Main 真实调用 Judge，由 Judge 独立读取证据并在 `[4] 评估结论` 中写入 `PASS / FAIL`。
    * Main 记录最终状态、真实 Session / Run 标识和归档路径。
* **通过条件**：以下证据必须同时存在：
    1. `state.md` 具有当前唯一 `Task_ID` 和最近修改时间。
    2. 至少一个非 Main 的已配置 Agent 存在真实 Session / Run。
    3. 子 Agent 回执包含与共享黑板相同的 `Task_ID`。
    4. 子 Agent 已将结果写入同一份 `state.md` 的授权章节。
   5. Main 或 Judge 已记录验收结论和对应证据。
* **失败处理**：若任一条件缺失，`Status` 必须标记为 `RUNTIME_NOT_READY` 或 `FAIL`，不得宣称 Matrix 已启用。Main 应检查子 Agent 调用权限、Agent ID 映射、Session 可见性、Workspace 路径、文件权限和 Prompt 注入后重新验收。

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
- **Matrix_Score**: `<0–10+>`
- **Matrix_Mode**: `MAIN_ONLY | MINIMAL | FULL`
- **Decision_Reasons**: `<评分依据和 Override 说明>`
- **Lifecycle_State**: `INIT`

## 📈 [1] 调度计划 (Dispatch Plan)
- **拓扑**: `res_n -> exe_n -> [dbg_n]`
- **当前激活**: `None`

### 节点账本 (Node Ledger)
#### res_1
- **Status**: `BLOCKED_BY_DEPENDENCY | READY | RUNNING | PASS | FAIL | RETRYING | CIRCUIT_OPEN`
- **Depends_On**: `none`
- **Assigned_Section**: `## [2] 黑板 -> ### 执行计划`
- **Agent_ID**: `res_1`
- **Session_Key**: `PENDING`
- **Run_ID**: `PENDING`
- **Retry_Count**: `0`
- **Started_At**: `PENDING`
- **Completed_At**: `PENDING`
- **Receipt**: `PENDING`
- **Writeback_Evidence**: `PENDING`

## 📖 [2] 共享工作区 / 黑板 (Blackboard)
### 📝 执行计划 (Research Plan)
- **Status**: `PENDING`

### 💻 产出结果 (Artifacts)
- **Status**: `PENDING`

## 🧪 [3] 质量中心 (Quality Center)
### dbg_1
- **Status**: `PENDING`

## ⚖️ [4] 评估结论 (Evaluation)
### judge
- **Status**: `PENDING`

## 📦 [5] 归档 (Archive)
- **Archive_Path**: `PENDING`
- **Archive_Checksum**: `PENDING`
- **Archived_At**: `PENDING`
```

###3. 动态模型分配协议 (Dynamic Model Allocation Protocol)

在用户根据本项目文件配置 Matrix 时，系统必须先完成模型发现，再创建或更新子 Agent。

#### 3.1输入：用户可用模型列表
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

####3.6 配置与审计输出

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

> 注意：以下为**概念示意**（早期版本的 `bridges` 字段并非 OpenClaw 真实配置格式）。实际配置请使用仓库根目录的 [`openclaw.json.example`](./openclaw.json.example) —— 基于真实 OpenClaw `agents.list` 结构，包含 Main 与全部 8 个子 Agent 的注册与 workspace 映射，复制后替换模型占位符即可。

```jsonc
// 概念示意：真实格式见 openclaw.json.example
{
  "agents": {
    "defaults": { "workspace": "<matrix-root>/workspace", "model": { "primary": "<main-model>" } },
    "list": [
      { "id": "main" },
      { "id": "res_1",  "workspace": "<matrix-root>/workspace/agents/res_1" },
      { "id": "res_2",  "workspace": "<matrix-root>/workspace/agents/res_2" },
      { "id": "exe_1",  "workspace": "<matrix-root>/workspace/agents/exe_1" },
      { "id": "exe_2",  "workspace": "<matrix-root>/workspace/agents/exe_2" },
      { "id": "dbg_1",  "workspace": "<matrix-root>/workspace/agents/dbg_1" },
      { "id": "dbg_2",  "workspace": "<matrix-root>/workspace/agents/dbg_2" },
      { "id": "dbg_3",  "workspace": "<matrix-root>/workspace/agents/dbg_3" },
      { "id": "judge",  "workspace": "<matrix-root>/workspace/agents/judge" }
    ]
  }
}
```

> **配置边界：** 上述映射用于描述 Matrix 的角色与共享状态关系，不会仅凭文件存在就自动触发子 Agent。实际配置必须使用当前 OpenClaw 版本支持的 Agent 注册、Main 子 Agent allowlist、Session 可见性和必要的 Agent-to-Agent 权限字段。配置写入后必须通过"配置后运行时验收"，不能只检查 JSON 语法或角色目录是否存在。

### 5. 提示词 (Prompt) 策略
* **Main Agent**（`prompts/main.md`）：包含确定性分流算法、控制器有序步骤、模式边界、回执验证、重试/熔断/恢复、完成门控和自动归档逻辑。
* **Worker Agents**（`prompts/res.md`、`prompts/exe.md`、`prompts/dbg.md`、`prompts/judge.md`）：必须遵循"协议化输出"，先读总线核对 Task_ID 和依赖，仅写入授权章节，输出结构化回执。
* **权限控制**：通过 System Prompt 强制各 Agent 仅拥有对自身章节的"写"权限和对他人的"读"权限。
