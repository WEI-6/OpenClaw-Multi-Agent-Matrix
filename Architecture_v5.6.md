# 🚀 OpenClaw 多 Agent 矩阵协作系统全局架构方案 (V5.6)

---

## 一、 设计目的 (Design Purpose)

本方案以工程化方式降低复杂任务中的上下文污染、逻辑漂移、并发冲突与无效循环，并提供可移植、可恢复、可审计的多 Agent 协作协议。

* **标准化协作**：以 Main 单写状态总线、Worker 只读执行并返回结构化回执的方式通信。
* **可靠交付**：复杂任务采用 Research/Delivery 两层 DAG、隔离执行、集成回归与 Judge 门禁。
* **动态适配**：按复杂度、可用角色、模型能力、成本和用户约束动态调度，不写死本机环境。
* **全生命周期管理**：从任务识别、局部自愈到 Judge `PASS` 后长期记忆更新、归档与重置形成闭环。
* **安全可移植**：发布文档只使用 `<workspace>`、`<project-root>`、`<task_folder>`、`<selected-model-id>` 等占位符；不得包含实际用户/主机路径、私有网络或 Provider、已安装模型 ID、运行中 Task/Session/Run、凭证或已删除文件名。

---

## 二、 设计理念 (Design Philosophy)

1. **Main 单写状态中心 (Main-Only State Authority)**
   `<workspace>/state.md` 是任务的唯一主状态总线。仅 Main 可物理写入、更新、重置或归档；Researcher、Executor、Debugger、Judge、Claude Code 与 Codex 均只读总线，完成后向 Main 返回 `WORKER_COMPLETED` 回执及建议的 `State_Patch`。Main 验证后串行提交，Worker 可并行计算。协议不依赖 Worker Revision、CAS 或文件锁。
   **Main 职责边界**：Main 的专属职责是 `state.md` 初始化与更新、DAG 构建与调度、Assignment 绑定、回执验收与串行状态写入、归档与最终说明。Main 不得在自身 Session 内代替 Researcher 产出研究结论、代替 Executor 生成实现产物、代替 Debugger 做独立质检，或代替 Judge 做最终裁决。
2. **职责解耦与隔离 (Decoupling & Isolation)**
   每个 Assignment 绑定独立 Session/Run、明确文件范围和验收标准。并行 Executor 使用独立工作树、分支、补丁或产物目录，不能直接争用同一工作树。
3. **摘要与证据驱动 (Summary & Evidence Driven)**
   状态总线只存状态、摘要、证据索引和 Digest；大产物与日志放在任务文件夹，敏感信息不得进入回执或状态。
4. **渐进式两层 DAG (Progressive Two-Layer DAG)**
   FULL 模式先建立 Research DAG，再随已验证研究结果用 `DAG_Update` 增量建立 Delivery DAG；部分研究完成即可安全解锁无争议节点。
5. **动态模型适配 (Dynamic Model Allocation)**
   从部署时实际可用模型动态发现并建立价格、上下文、推理、代码、速度、稳定性与工具能力画像，不预设 Provider 或模型 ID。
6. **局部自愈与两级熔断 (Local Recovery & Two-Level Circuit Breaking)**
   失败按 `Task_ID + Subtask_ID + Stage + Root_Cause_Signature` 隔离计数，先进行最多 3 次执行链重试，再退回 Researcher 进行最多 3 次实质修订；不影响无依赖链路。
7. **运行时可验证 (Runtime Verifiability)**
   静态角色/config 不等于已启用。配置后必须真实调用非 Main Agent，以 Assignment、Session/Run、回执和 Main 写入证据证明调度有效。
8. **长期记忆最小化 (Minimal Durable Memory)**
   `<task_folder>/MAM_state.md` 只保留跨任务有效信息；每个 Task 至多更新一次，且仅在 Judge `PASS` 后、归档或重置前更新并回读验证。

---

## 三、 角色定义与分工 (Role Definition & Division)

| 角色组 | 建议实例 | 核心职责 | 状态权限 | 输出 |
| :--- | :--- | :--- | :--- | :--- |
| **调度中心 (Main)** | `main` | 分流、评分、两层 DAG、Assignment、回执验证、串行状态提交、恢复、归档 | 唯一可写 `state.md` | 状态、调度与验收记录 |
| **研究池 (Researcher)** | `res_1~3` | 需求/边界、方案/接口、风险/验收研究 | 只读 | 研究结果、`State_Patch` |
| **执行池 (Executor)** | `exe_1~3` | 隔离实现、工具执行、测试与产物生成 | 只读状态；仅改授权产物 | 产物、证据、`State_Patch` |
| **调试池 (Debugger)** | `dbg_1~3` | 逻辑、边界、性能、安全与回归检查 | 只读 | 质检回执、`State_Patch` |
| **集成角色 (Integration)** | 动态节点 | 独占集成产物、冲突仲裁、合并 | 只读状态；独占集成范围 | 集成产物与回执 |
| **仲裁员 (Judge)** | 默认 1 个 | 独立完成门控与最终裁决 | 只读 | `PASS/REJECTED` 回执 |

**职能池并发上限：** Researcher 最多 3；Executor 最多 3（`openclaw_native`、`claude_code`、`codex` 合计）；Debugger 最多 3；Integration 最多 3，但同一集成产物仅 1 个有效节点；Judge 最多 3 个实例，同一任务最终裁决默认仅激活 1 个。实际并发为 READY 节点数、可用 Agent 数、池上限三者最小值；不另设模型级并发限制。

---

## 四、 核心步骤 (Core Steps)

### 1. 任务初始化与分流 (Initialization & Triage)

Main 由 `New_Task_Flag = TRUE` 或用户新请求识别新任务，不得仅因 `Status = COMPLETED` 推断新任务。Main 确定 `<task_folder>`；**FULL 任务在评分与拆分前必须读取 `<task_folder>/MAM_state.md`（若存在）**，但 MAM 只提供长期规划上下文，不恢复运行节点、Assignment 或 Session，也不替代当前任务的研究与评分。

**硬覆盖规则优先于评分：**

| 条件 | 结果 |
|---|---|
| 用户明确要求 Matrix、多 Agent、并行或独立 Judge | `FULL` |
| 破坏性、不可逆、外部发布/消息/事务、凭证、隐私或权限边界变更 | 先 `WAITING_USER`，确认后再评分 |
| 纯闲聊或无需文件、工具、独立验证的简单只读查询 | `MAIN_ONLY` |
| 用户明确指定执行方式或禁止拆分 | 遵循用户要求，保留必要安全门禁 |

**多维加法评分（每个实际加分项及理由均写入状态，不得只写总分）：**

| 维度 | 分值 |
|---|---:|
| 安全与失败影响 | `+3` |
| 多文件或跨组件 | `+2` |
| 实现与工具执行 | `+2` |
| 独立质量验证 | `+2` |
| 独立交付物与并行价值 | `+2` |
| 集成复杂度 | `+2` |
| 研究与需求不确定性 | `+1` |
| 长时运行与恢复敏感 | `+1` |

阈值：`0–2 → MAIN_ONLY`；`3–5 → MINIMAL`；`≥6 → FULL`。

- **MAIN_ONLY**：Main 直接处理，做风险匹配的最小验证；不派发任何子 Agent，任务在 Main 自身 Session 内完成。
- **MINIMAL**：轻量 Matrix 路径。Main **跳过 Research DAG**，直接建立 Delivery DAG 并派发真实非 Main 角色（EXE-* 等）；Research DAG 在此模式下不强制，但 Delivery DAG 必须存在，Main 不得在自身 Session 内代替 EXE/DBG 产出专家产物。MINIMAL 链路为：Main 分流/模型分配 → Delivery DAG（EXE-* → DBG-*（按风险）→ Integration（按需）→ Judge（按风险））。必须记录 Assignment、模型分配与验收证据；DBG 与 Judge 按风险裁定是否激活，但不得完全省略 EXE 节点。
- **FULL**：强制 Research DAG → Delivery DAG 两层结构、Debugger 与 Judge，Main 不得在自身 Session 代做完整专家工作。**FULL 任务必须先显式建立并保留 Research DAG，待 RES 节点 PASS 后再通过 `DAG_Update` 增量建立 Delivery DAG；禁止跳过 Research DAG 直接进入 Delivery DAG。**

FULL 的 Researcher 数量：`6–7` 分为 1 个，`8–10` 分为 2 个并行，`≥11` 分为 3 个并行。跨专业、争议方案、高风险多路径或用户要求可上调但不超过 3；低于建议数量须记录原因。

每个子任务至少包含：`Subtask_ID`、名称、目标、输入与依赖、输出路径或章节、负责角色与 Agent ID、允许修改范围、禁止修改范围、验收标准、状态、重试次数、开始/完成时间、Session Key/Run ID、结果摘要与证据。

### 2. 模型发现与子 Agent 分配 (Model Discovery & Assignment)

Main 以“用户政策优先、证据优先、按 Assignment 动态匹配”为核心，先发现可用模型，再做画像与分配；不得把目录里出现的模型、展示页或历史配置误当作运行可用证明。

**发现来源仅限以下四类，并分别记录证据与 provenance：**
1. 当前 OpenClaw 配置与其允许清单（allowlist）
2. 用户显式提供的 API / 模型清单
3. 已登记的目录/元数据发现结果
4. 受控最小探测或真实运行验收结果

**allowlist 语义：**
- 若存在 allowlist，它是可分配边界；不在 allowlist 内的模型不得分配给任何 Assignment。
- 目录或 catalog 只能补充元数据，不得绕过 allowlist。
- 仅“看得见”不等于“能执行”；目录列出、auth 可见、ready-marker 存在都不构成 runtime proof。
- 未知元数据保持 `unknown`，不得猜测、补全或编造。

**可用性证据分层：**
- `configured_allowed`：已配置且在 allowlist 内
- `ready_marker`：存在可读的 auth/local/plugin 就绪标记，但仍不代表可执行
- `catalog_listed`：目录中列出，仅表示元数据可见
- `targeted_probe`：针对性最小探测成功，证明特定能力或接口可用；探测会消耗 token / rate limit，必须按需、定向、最小化
- `runtime_validated`：真实子 Agent 运行验收成功，证明该模型/执行路径在当前 Assignment 上可用

这些是并列证据维度，不是强制线性 auth 链；不存在“一步必须自动推导下一步”的全局链式假设。Main 只能在证据足够时将其提升为可分配候选。

**画像维度必须完整且可回读：**
- 价格：已知单价、未知单价、预算敏感度
- 上下文：native context、effective context、输出长度、结构化稳定性
- 能力：reasoning、coding、debugging、writing、tool use、multimodal、schema adherence、summarization、retrieval、planning
- 运行：速度、延迟、稳定性、失败率、并发占用、限流/配额、重试代价
- 合规：隐私敏感度、数据驻留/本地优先、禁用列表、用户固定模型/固定角色偏好
- 证据：来源、时间、验证方式、适用范围、失效条件

**角色需求不是静态绑定，而是按 Assignment / Subtask 计算：**
- Main：编排、验证、写入、恢复、归档、预算控制与最终门禁
- Researcher：需求澄清、边界识别、方案比较、风险与验收拆解
- Executor：实现、工具执行、修复、产物生成与局部回归
- Debugger：按领域拆分的专项质检；包括 schema/receipt、portability/security、integration/regression、performance/stability、model-allocation-audit 等
- Integration：多产物合并、冲突仲裁、回归验证、冻结接口后的集成落地
- Judge：独立裁决、完成门控、是否可归档

Main 为每个子任务创建唯一 Assignment，并允许同一角色在不同 Assignment 上动态升降级；高能力/高价模型只用于高杠杆或高难节点，轻量模型优先覆盖研究、格式校验、回归与低风险执行。

**分配原则：**
1. 用户政策先于优化：固定模型、禁用列表、allowlist、预算、隐私/数据约束、并发与回退策略优先。
2. 先过硬约束，再做评分；任何硬约束不满足直接淘汰。
3. 评分维度按 Assignment 组合：能力覆盖、可靠性、上下文匹配、工具适配、延迟、预计输入+输出 token、预计总成本。
4. 估算 token 时同时考虑该 Assignment 的输入规模、生成规模、回合数、重试预算和 probe 成本。
5. 动态 escalation / de-escalation：当任务难度、上下文消耗、失败证据或用户约束变化时，Main 可在未开始节点上升级或降级模型；已 `PASS` 结果不回改。
6. 发生能力不足、限流、工具失败、证据不足、成本超标或策略冲突时，按预设 fallback / retry / reduce-scope / switch-model / WAITING_USER 处理。

Main 为每个子任务创建唯一 Assignment：

```text
Assignment_ID = Task_ID / Subtask_ID / Attempt
```

同一 `Subtask_ID` 同时只能有一个有效 Assignment；首次 `Attempt = 1`，仅重试时递增，派发后不可更改；Agent ID、Session Key、Run ID、角色、执行器类型和目标范围必须绑定 Assignment。READY 且无有效 Assignment 才可派发；RUNNING/PASS 禁止重复派发；FAIL/BLOCKED 重试必须 `Attempt + 1`。

**推荐工作流（可审计、可重算）：**
1. 读取用户政策、allowlist、预算、固定模型/禁用列表与保密边界。
2. 汇总可用来源，形成标准化 user-available-model set。
3. 合并 provenance，标记每条元数据的来源、时间与证据等级。
4. 对每个待派发 Assignment 构建 demand vector：目标角色、任务类型、上下文需求、工具需求、结构化输出要求、容错需求、隐私等级、预算上限、时延上限、probe 许可。
5. 对候选模型先做 hard-filter，再按 capability/reliability/context/tool/latency/estimated tokens/estimated total cost scoring。
6. 计算并发占用与预算；高价模型在 premium concurrency 下优先保留给高难/高杠杆节点。
7. 对缺失或不确定信息保持 `unknown`，必要时仅执行 targeted probe，不做全量盲测。
8. 生成分配记录、警告、回退路径与运行验收要求。

**伪代码：**

```pseudo
models = discover_from_config_and_user_allowlist()
models = normalize_and_tag_provenance(models)
models = filter_by_allowlist_and_user_policy(models)
for assignment in ready_assignments:
    demand = build_demand_vector(assignment)
    candidates = hard_filter(models, demand)
    if candidates.empty:
        fallback = choose_fallback_or_wait_user(assignment)
        record_allocation(assignment, fallback, reason="no_hard_match")
        continue
    scored = score_candidates(
        candidates,
        capability=demand.capability,
        reliability=demand.reliability,
        context=demand.context,
        tool=demand.tool,
        latency=demand.latency,
        token_cost=estimate_tokens(assignment),
        total_cost=estimate_total_cost(assignment)
    )
    selected = choose_best(scored, budget, premium_concurrency, policy)
    if selected.needs_probe and probe_budget_available(selected):
        selected = run_targeted_probe(selected, demand)
    if selected.runtime_validated or selected.ready_to_assign:
        bind_assignment(assignment, selected)
    else:
        fallback = choose_fallback_or_wait_user(assignment)
        bind_assignment(assignment, fallback)
```

**配置与审计输出必须可支持：**
- `model_allocation`：每个 Assignment 的 selected model、理由、证据等级、预算消耗、回退路径
- `allocation_audit`：输入政策、allowlist、禁用项、候选集合、评分摘要、probe 记录、runtime 验收结果、失败原因、rematch 记录
- `model_profile`：归一化画像（未知项显式 `unknown`）
- `rematch_log`：模型升级/降级/回退/重新匹配的原因与时间

**配置与验收要求：**
- 仅使用 OpenClaw 支持的配置/Schema 字段，不引入未支持的顶层字段或私有扩展。
- 配置后必须做 schema 校验、readback 校验与真实 runtime acceptance。
- runtime acceptance 不能仅靠 catalog、ready-marker 或静态声明；必须以真实非 Main 子 Agent 的回执与写入证据为准。
- targeted probe 只能作为局部能力证据，不能被冒充为全面运行证明。
- probes 与 runtime 验收必须记录 token、rate limit、失败码与适用范围，避免无意义消耗。

### 3. 路由规划与显式调度 (Routing & Explicit Dispatch)

FULL 使用两层 DAG：

```text
Research DAG:  RES-1 || RES-2 || RES-3
                         ↓ Main 验证每份回执
Delivery DAG:  EXE-* → DBG-* → INTEGRATION(按需) → REGRESSION(按需) → JUDGE
```

Researcher 可按需求与边界、技术方案与接口、风险与验收分工。Main 每收到一个有效研究回执就串行写入 Research DAG，并判断是否可通过 `DAG_Update` 提前解锁 Delivery 节点，无需等待同批全部完成。节点派发必须同时满足：用户策略允许、Assignment 已绑定、能力画像通过 hard-filter、预算与并发未超限、证据等级满足该节点最低要求。

Delivery DAG 至少明确实现依赖、Executor 产物路径、`executor_type`、对应 Debugger、并行/串行关系、Integration 是否需要、集成回归范围和 Judge 条件。Main 仅调度依赖满足的 READY 节点，使用真实注册 Agent ID 和独立 Session；采用推送式完成事件，不忙轮询。**交付扇出规则**：RES PASS 后，若多个交付工作之间无依赖关系，必须拆分为独立的并行 EXE 节点同波次派发，禁止将独立交付工作串行合并到单一 EXE 节点。

`DAG_Update` 由 Main 增量维护，不整体重写，最小结构如下：

```json
{
  "schema_version": "1.0",
  "update_id": "<task-id>/DAG-UPDATE/<sequence>",
  "task_id": "<task-id>",
  "triggered_by": {"subtask_id": "<research-node>", "assignment_id": "<assignment-id>", "result_digest": "<digest>"},
  "created_at": "<timestamp>",
  "add_nodes": [], "update_nodes": [], "add_edges": [], "remove_edges": [], "cancel_nodes": [],
  "notes": "<reason>"
}
```

节点 ID 创建后不可重命名或复用；重试只增加 Attempt。允许增加节点/边、更新非身份字段、移除未开始节点间的边、取消未开始或已终止节点；禁止删除完成记录、修改 Node ID、将 RUNNING/PASS 改回 READY、移除完成证据、形成环或改变与运行 Assignment 冲突的产物范围。边类型为 `REQUIRES_PASS`、`REQUIRES_COMPLETION`、`INFORMS`。Main 应检查 ID/产物范围冲突、依赖环和运行状态后应用并重算 READY 集。

`Update_ID` 幂等：同 ID 同内容忽略；同 ID 不同内容为 `DAG_UPDATE_CONFLICT`；不同 ID 定义同 Node ID，完全相同则忽略，否则为 `NODE_DEFINITION_CONFLICT`。

### 4. 链式执行与自愈循环 (Execution & Self-Healing Loop)

所有 Worker 开始前只读同一 `state.md`，核对 Task/Subtask/Assignment、依赖和授权范围；不得直接写、重置或归档状态。完成后返回统一 JSON 回执，并附带 `State_Patch`（仅为 Main 应写入目标章节的完整替换 Markdown，不是 Worker 写操作）：

```json
{
  "schema_version": "1.0",
  "event_type": "WORKER_COMPLETED",
  "task_id": "<task-id>",
  "subtask_id": "<subtask-id>",
  "assignment_id": "<task-id>/<subtask-id>/<attempt>",
  "attempt": 1,
  "role": "executor",
  "agent_id": "<agent-id>",
  "executor_type": "openclaw_native",
  "session_key": "<session-key>",
  "run_id": "<run-id>",
  "status": "PASS",
  "target_section": "<authorized-section>",
  "completed_at": "<timestamp>",
  "result_summary": "<summary>",
  "artifacts": [{"path": "<artifact-path>", "type": "source", "sha256": "<optional-sha256>"}],
  "evidence": [{"type": "test", "summary": "<test-summary>", "path": "<evidence-path>"}],
  "errors": [],
  "next": ["<next-node>"],
  "State_Patch": "<complete replacement markdown for target_section>",
  "result_digest": "<sha256-of-canonical-result-payload>"
}
```

必填字段为以上全部字段，包括 `State_Patch` 与 `result_digest`；数组为空时必须为 `[]`。`status` 仅允许 `PASS / FAIL / BLOCKED / CANCELLED`；`executor_type` 仅允许 `openclaw_native / claude_code / codex`。Result Digest 依固定顺序覆盖 Assignment ID、Status、Result Summary、Artifacts、Evidence、Errors、Next 的标准 JSON，Main 必须重算，不信任 Worker 值。`State_Patch` 单独接受范围和内容校验。

Main 拒绝 Task/Subtask/当前 Assignment 不匹配、Agent/Session/Run 不匹配、字段缺失、枚举无效、Digest 不符、产物不存在或 Target/State_Patch 越权的回执。验证流程为：解析 → 身份/Assignment 核对 → 重算 Digest → 重复/冲突/迟到判断 → 检查状态/证据/产物/范围 → 串行写入 `State_Patch` → 记录接受标志与 Digest → 更新节点 → 重算 READY → 派发。若验证通过但能力证据不足以支持后续节点，Main 应 rematch 或降级，而不是默认继续使用同一模型。

每个 Assignment 记录 `Result_Accepted` 与 `Result_Digest`：首次有效完成才接受；同 Assignment 已接受且 Digest 相同，记录 `Duplicate_Event_Ignored`，不写入、不解锁；Digest 不同标记 `RESULT_CONFLICT`，拒绝覆盖并保留冲突证据；旧 Attempt 迟到标记 `LATE_RESULT`，默认不写入、不改变节点，仅保留诊断摘要。

**Executor 隔离与集成：** 每个 Executor 使用 `<task_folder>/artifacts/<subtask-id>/` 或独立工作树/分支/patch；并行 Executor 不得直接改同一工作树。完成回执后停止修改、退出运行并释放工具会话。**EXE 验证边界**：EXE 节点只能做本地完成性检查（如编译通过、冒烟测试、产物存在），并将结果仅作为执行证据（execution evidence）写入回执。**EXE 不得将任何自运行结果标注为 acceptance、QA、validation、PASS gate 或 final verification；此类标注无效且视为越权。** 独立的最终接受验证由 DBG 节点完成，EXE 不得替代 DBG 角色。多产物合并时建立 Integration 节点；全部相关 Executor 完成并退出后，集成者才取得 `<task_folder>/artifacts/INTEGRATION/` 或目标集成范围的独占修改权。前序 Executor 不得再修改，后续修复创建新 Assignment/节点。Integration 后运行跨模块接口、构建和整体行为回归。

实现冲突先受用户要求、安全边界和冻结接口约束；不合规方案直接淘汰。其余按 Main 在任务开始时记录的动态 `Model_Capability_Order`（Tier 1/2/3）仲裁：高能力等级优先，低等级可补充但不可覆盖；同等级交 Judge。Integration 记录双方、模型等级、采用方案和理由。

**局部重试与两级熔断：** 计数键为 `Task_ID + Subtask_ID + Stage + Root_Cause_Signature`。Executor/Debugger/Integration/Regression 失败仅暂停受影响链路，无依赖链继续，依赖失败节点者保持 `BLOCKED_BY_DEPENDENCY`。第一阶段严格为 `Attempt 1（初始）→ Attempt 2（重试1）→ Attempt 3（重试2）→ Attempt 4（重试3）`；Attempt 4 同根因仍失败则执行链熔断并退回 Researcher。第二阶段 Researcher 必须基于证据给出实质不同方案，Main 增量更新 Delivery DAG，新方案可重置对应链路重试；Researcher 最多 3 次修订。第 3 次修订后仍失败，或连续 3 次无可执行方案，进入 `WAITING_USER`，报告子任务/阶段、3 次局部重试、3 次研究修订、根因证据、选项与待决定事项。根因签名仅在机制、位置或解决路径实质变化时改变；PASS 结束当前计数但保留历史。

### 5. 恢复与重建 (Recovery & Reconciliation)

中断时进入 `RECOVERING`，以 Main 最后成功写入的 `state.md` 为准，不依赖聊天记忆：已写入结果不重复执行；已完成未写入结果从绑定 Session 重新取得；RUNNING Worker 先核对状态再等待、恢复或重试；孤立运行标记后处理；重复回执按 `Assignment_ID + Result_Accepted + Result_Digest`，DAG 更新按 `Update_ID` 幂等处理。有效迟到结果只作诊断，除非 Main 明确重新验证并建立新的有效 Assignment。Main 物理写入失败时不得声称状态已提交或解锁下游；具体恢复机制由部署实现规定，本架构不展开。

### 6. 仲裁与完成门控 (Review & Completion Gate)

Debugger 仅在对应 Executor `PASS` 后启动；失败只重开受影响链路。Judge 仅在所有必要 Executor 与 Debugger 通过、Integration/Regression 通过或明确不需要、且无未解决失败/阻塞/冲突/熔断时启动。

Judge 独立核验：交付与验收标准、必要测试、节点证据、Assignment/Session/Run/Digest、文件与安全边界、集成与回归、未解决风险。任一 CRITICAL 问题一票否决。裁决仅为 `PASS` 或 `REJECTED`；只有全部门控为真才能 `PASS`。

### 7. 自动归档 (Automatic Archiving)

Judge `PASS` 后、归档或重置前，Main 必须按以下顺序执行，且每个 Task 恰好更新一次 MAM：

1. 检查 `<task_folder>/MAM_state.md` 的 `Last_Completed_Task_ID`；相同则跳过写入（幂等保护）。
2. 读取旧 MAM（若存在），从最终状态、产物和验收证据提炼长期有效内容，去重并移除过时内容。**MAM 只记录跨任务长期有效的规划记忆；不得写入活跃节点状态、Session/Run/Assignment 标识、调度器状态或任何瞬态执行证据。**
3. 写入并回读验证 MAM；仅回读成功后设置 `MAM_Update_Status = PASS`。
4. 原子、幂等归档当前 `state.md` 至 `<workspace>/history/<task-id>/`，记录路径与校验和。
5. 仅在归档成功且 MAM 门禁通过后重置 `state.md`，再向用户报告。

MAM 写入或回读失败会阻止清空/重置状态；Main 先修复，无法修复则进入 `WAITING_USER`，说明交付已完成但长期记忆保存失败。破坏性、不可逆、外部、隐私/凭证或权限边界操作仍遵循用户审批门。

### 8. 配置后运行时验收 (Post-Configuration Runtime Acceptance)

配置后必须运行等价于 `main → res_1 → judge → main` 的最小真实链路：Main 创建占位化测试 Task/Assignment，真实调用独立 Session 的 Worker；Worker 只读总线并返回完整回执与 `State_Patch`；Main 校验身份/Digest/范围后写入；Judge 独立读取证据并返回裁决；Main 写入结论。必须同时证明：唯一 Task ID、非 Main Session/Run、Assignment 绑定、Worker 无状态写权限、Main 接受并物理写入、Judge 证据和归档路径。还应覆盖三个 Researcher 乱序完成、单份研究提前解锁、并行隔离、Integration 独占、局部失败不阻塞、重复/冲突/迟到事件、恢复与 MAM 回读门禁。

任一证据缺失时标记 `RUNTIME_NOT_READY` 或 `FAIL`，不得宣称 Matrix 已启用；修正 Agent 注册/allowlist、Session 可见性、工作区、权限、Prompt 或配置后重新验收。验收结果必须能回读到配置 schema、allowlist、模型画像、Assignment 绑定、runtime proof 与失败原因；若 targeted probe 成功但 runtime acceptance 失败，必须降级为“局部可用”而非“全面可用”。

---

## 五、 OpenClaw 实施 (Implementation)

### 1. 路径占位符定义 (Path Placeholder Definitions)

本协议使用以下占位符；发布文档不得将其替换为实际主机路径或私有标识。

| 占位符 | 含义 |
|---|---|
| `<workspace>` | OpenClaw 全局工作区根目录；`state.md` 与 `history/` 均位于此目录下。 |
| `<project-root>` | 当前项目的代码仓库或顶级目录；提示词、配置等项目级资产放于此处。 |
| `<task_folder>` | 当前任务的最小稳定工作根目录（见下方详细说明）。 |
| `<state-file>` | 任务状态总线文件，固定为 `<workspace>/state.md`。 |

**`<task_folder>` 详细说明**

`<task_folder>` 是当前任务的最小稳定工作根目录，用于承载：任务产物（`artifacts/`）、`MAM_state.md`、任务内相对路径引用，以及任务级归档上下文。

- **选择原则**：Main 必须在任务初始化时显式选择并在 `state.md` 的 `[0] Signals` 中记录 `Task_Folder`，以及选择理由（`Task_Folder_Reason`）。
- **与 `<project-root>` 的关系**：`<task_folder>` *可以*等于 `<project-root>`，但**不得默认等于 `<project-root>`**。仅当任务真正跨越整个项目时方可令二者相等，且必须写明理由。
- **典型区分示例**：`<project-root>` 可以是整个 monorepo（如 `my-repo/`），而同一任务的 `<task_folder>` 可以是其中一个子模块（如 `my-repo/services/auth/`）；前者是项目边界，后者是本次任务的实际工作范围。
- **禁止行为**：不得将 `<task_folder>` 省略、隐式继承或在任务中途静默更改；变更须通过新 Assignment 明确授权。

### 2. 目录结构

```text
<workspace>/state.md
<workspace>/history/<task-id>/
<task_folder>/MAM_state.md
<task_folder>/artifacts/<subtask-id>/
<task_folder>/artifacts/INTEGRATION/
<project-root>/prompts/
```

### 2. `state.md` 核心模版

```markdown
# 📂 OpenClaw 任务总线 (State Center)

## 📊 [0] 系统信号 (Signals)
- **Task_ID**: `<task-id>`
- **Status**: `INIT`
- **New_Task_Flag**: `TRUE`
- **Matrix_Score**: `<score>`
- **Matrix_Mode**: `MAIN_ONLY | MINIMAL | FULL`
- **Decision_Reasons**: `<each scoring dimension and override>`
- **Lifecycle_State**: `INIT`
- **Model_Capability_Order**: `<dynamic tiers>`
- **Model_Discovery_Provenance**: `<sources-and-evidence>`
- **MAM_Update_Status**: `PENDING`
- **Task_Folder**: `<task_folder>`
- **Task_Folder_Reason**: `<why this folder was chosen as task root; may equal project-root only when task truly spans the whole project>`

## 📈 [1] 调度计划 (Dispatch Plan)
### Research DAG
#### RES-1
- **Subtask_ID**: `RES-1`
- **Assignment_ID**: `<task-id>/RES-1/1`
- **Attempt**: `1`
- **Status**: `READY`
- **Depends_On**: `none`
- **Agent_ID**: `<agent-id>`
- **Session_Key**: `PENDING`
- **Run_ID**: `PENDING`
- **Target_Section**: `[2]/RES-1`
- **Allowed_Scope**: `<allowed>`
- **Forbidden_Scope**: `<forbidden>`
- **Acceptance**: `<criteria>`
- **Result_Accepted**: `FALSE`
- **Result_Digest**: `PENDING`

### Delivery DAG
#### EXE-A
- **Status**: `BLOCKED_BY_DEPENDENCY`
#### DBG-A
- **Status**: `BLOCKED_BY_DEPENDENCY`
#### INTEGRATION
- **Status**: `NOT_REQUIRED | BLOCKED_BY_DEPENDENCY`
#### REGRESSION
- **Status**: `NOT_REQUIRED | BLOCKED_BY_DEPENDENCY`
#### JUDGE
- **Status**: `BLOCKED_BY_DEPENDENCY`

### Applied DAG Updates
- `<update-id>`

## 📖 [2] 共享工作区 / 黑板 (Blackboard)
### RES-1
- **Status**: `PENDING`
- **Result**: `PENDING`
- **Evidence**: `PENDING`

## 🧪 [3] 质量中心 (Quality Center)
- **Status**: `PENDING`

## ⚖️ [4] 评估结论 (Evaluation)
- **Status**: `PENDING`

## 📦 [5] 归档 (Archive)
- **MAM_Update_Status**: `PENDING`
- **Archive_Path**: `PENDING`
- **Archive_Checksum**: `PENDING`
```

###3. 动态模型分配协议 (Dynamic Model Allocation Protocol)

本节把“动态发现 → 画像 → 约束过滤 → 打分 → 绑定 → 验证 → 复配”拆成可落地步骤；它与第二章第 2 节的控制原则一致，但此处给出实现/配置/审计所需的细化协议。

#### 3.1 输入、发现顺序与 allowlist 边界

Main 先读取用户策略，再读取 OpenClaw 当前配置/allowlist，最后汇总可发现来源；发现顺序只影响证据优先级，不表示自动继承运行可用性。

1. **用户策略**：固定模型、禁用模型、预算上限、隐私/驻留要求、角色偏好、允许回退路径。
2. **OpenClaw 配置与 allowlist**：若存在 allowlist，它就是可分配边界；不在 allowlist 内的模型不得分配给任何 Assignment。
3. **用户显式提供的 API / 模型清单**：只作为候选输入，不自动等于可执行。
4. **目录/元数据发现**：只能丰富候选元数据，不能突破 allowlist，也不能替代运行验收。
5. **受控最小探测 / 真实运行验收**：仅在证据不足时执行，且结果只证明当前能力或当前 Assignment 下可用，不能推导所有场景都可用。

发现时必须保留 provenance：来源、时间、查询方式、证据等级、适用范围、失效条件。若 allowlist 为空，则可用集合由用户策略、配置约束与显式提供清单共同决定；若 allowlist 存在，则它优先于目录、探测和展示页。

#### 3.2 可用性证据模型

可用性不是单线认证链，而是并列证据维度；Main 不能把“目录看见”“有 ready-marker”误当成“已可运行”。推荐证据维度如下：

- `configured_allowed`：已配置且在 allowlist 内
- `ready_marker`：存在可读的 auth/local/plugin 就绪标记
- `catalog_listed`：目录或 catalog 可见，仅说明元数据存在
- `targeted_probe`：针对性最小探测成功，证明某项能力/接口可用
- `runtime_validated`：真实非 Main 子 Agent 的运行验收成功

这些证据可以并列存在；Main 在做分配时按 Assignment 要求判断“证据是否足够”，而不是默认按线性链自动升级。未知字段保持 `unknown`，不得猜测、补全或编造。

#### 3.3 画像 Schema

每个候选模型都应形成归一化画像。建议最小字段如下，字段缺失时写 `unknown`：

```json
{
  "model_profile": {
    "model_id": "<selected-model-id>",
    "provider_ref": "<provider-or-unknown>",
    "source_refs": ["<provenance-ref>"],
    "evidence": {
      "configured_allowed": "true|false|unknown",
      "ready_marker": "true|false|unknown",
      "catalog_listed": "true|false|unknown",
      "targeted_probe": "true|false|unknown",
      "runtime_validated": "true|false|unknown"
    },
    "context": {
      "native": "<tokens-or-unknown>",
      "effective": "<tokens-or-unknown>",
      "output": "<tokens-or-unknown>",
      "structured_stability": "high|medium|low|unknown"
    },
    "capabilities": {
      "reasoning": "high|medium|low|unknown",
      "coding": "high|medium|low|unknown",
      "debugging": "high|medium|low|unknown",
      "writing": "high|medium|low|unknown",
      "tool_use": "high|medium|low|unknown",
      "multimodal": "high|medium|low|unknown",
      "schema_adherence": "high|medium|low|unknown",
      "summarization": "high|medium|low|unknown",
      "retrieval": "high|medium|low|unknown",
      "planning": "high|medium|low|unknown"
    },
    "runtime": {
      "latency": "<ms-or-unknown>",
      "stability": "high|medium|low|unknown",
      "failure_rate": "<rate-or-unknown>",
      "rate_limits": "<limits-or-unknown>",
      "concurrency": "<slots-or-unknown>"
    },
    "cost": {
      "input_token_price": "<known-or-unknown>",
      "output_token_price": "<known-or-unknown>",
      "probe_cost": "<known-or-unknown>",
      "budget_sensitivity": "high|medium|low|unknown"
    },
    "policy": {
      "privacy": "<level-or-unknown>",
      "data_residency": "<local-or-unknown>",
      "fixed_role": "<role-or-unknown>",
      "disabled": ["<model-or-provider>"],
      "warnings": ["<warning>"]
    }
  }
}
```

画像里的能力维度不要求每项都有明确数值，但必须可回读、可审计、可标注证据来源；高风险缺失项应直接标记 `unknown` 并触发保守分配或 probe。

#### 3.4 角色先验与 per-Assignment 需求向量

角色是先验，不是死绑定；真正分配时按 Assignment 重新计算需求向量。最少要包含：

- **角色先验**：
  - Researcher：偏重低成本、长上下文、摘要/比较/风险识别
  - Executor：偏重工具适配、代码/操作能力、产物生成、稳定输出
  - Debugger：偏重结构化输出、错误定位、回归/一致性/安全校验
  - Integration：偏重跨产物合并、冲突仲裁、变更冻结后的整合
  - Judge：偏重证据整合、门控判断、规则一致性与最终裁决

- **per-Assignment demand vector**：
  - 目标角色/子角色
  - 输入规模与上下文需求
  - 预期输出规模
  - 结构化输出要求
  - 工具需求（shell / file / browser / API / patch 等）
  - 隐私与驻留等级
  - 失败容忍度与重试预算
  - 预计 probe 次数与 probe 成本
  - 预算上限与时延上限
  - 固定/禁用模型与回退策略
  - 是否允许 premium concurrency

需求向量必须按 Assignment 构建，而不是按静态角色一次性固定。相同角色在不同 Assignment 上可以升降级；高能力/高价模型只应用于高杠杆、高难度或高失败成本节点，轻量模型优先覆盖研究、格式校验、回归和低风险执行。

#### 3.5 硬约束与打分

先硬过滤，再打分；任何硬约束不满足的候选直接淘汰。硬约束至少包括：

- 用户固定/禁用模型与显式回退策略
- allowlist 边界
- 角色/工具/结构化输出硬要求
- 上下文最低需求
- 隐私/驻留/权限边界
- 可用并发与配额
- 预算上限与 probe 上限

打分必须把估计输入 token、输出 token、重试次数、probe 次数和总成本纳入同一分配决策。推荐排序维度：

1. 能力覆盖度
2. 可靠性与稳定性
3. 上下文匹配度
4. 工具适配度
5. 预估时延
6. 预估输入 token
7. 预估输出 token
8. 预估重试/探测代价
9. 预估总成本

`premium concurrency` 的含义是：高价/高杠杆模型的并发槽位要严格保留给确有必要的高难 Assignment，不能被低价值节点占满；预算不足时应先降级、缩 scope 或回退，而不是默认扩槽。

#### 3.6 推荐工作流与伪代码

```text
1) 读取用户策略、allowlist、固定/禁用模型、预算与隐私边界
2) 汇总所有候选来源并归一化 provenance
3) 生成每个候选模型的画像，未知项保持 unknown
4) 为每个 READY Assignment 构建 demand vector
5) 先 hard-filter，再按 capability/reliability/context/tool/latency/estimated tokens/retries/probes/total cost scoring
6) 检查预算与 premium concurrency
7) 必要时执行 targeted probe；probe 只补局部证据，不冒充全面运行证明
8) 绑定 model ↔ Assignment，记录 reason/warnings/fallback
9) 做 schema 校验、readback 校验、runtime acceptance
10) 若证据变化或运行失败，则 rematch / escalate / de-escalate
```

```pseudo
sources = discover(user_policy, openclaw_allowlist, user_models, catalogs)
models = normalize_with_provenance(sources)
models = apply_hard_constraints(models, user_policy, allowlist)
for assignment in ready_assignments:
    if has_valid_current_allocation_record(assignment):
        continue
    demand = build_demand_vector(assignment)
    candidates = hard_filter(models, demand)
    if candidates.empty:
        fallback = choose_fallback_or_wait_user(assignment)
        record_allocation(assignment, fallback, reason="no_hard_match", warnings=["hard_filter_empty"])
        continue
    ranked = score(
        candidates,
        capability=demand.capability,
        reliability=demand.reliability,
        context=demand.context,
        tool=demand.tool,
        latency=demand.latency,
        input_tokens=estimate_input_tokens(assignment),
        output_tokens=estimate_output_tokens(assignment),
        retries=estimate_retries(assignment),
        probes=estimate_probe_cost(assignment),
        total_cost=estimate_total_cost(assignment)
    )
    selected = choose_best(ranked, budget, premium_concurrency, policy)
    if needs_targeted_probe(selected, demand):
        selected = run_targeted_probe(selected, demand)
    if selected.runtime_validated or selected.ready_to_assign:
        bind_assignment(assignment, selected)
    else:
        fallback = choose_fallback_or_wait_user(assignment)
        bind_assignment(assignment, fallback)
    record_allocation_audit(assignment, selected, demand)
```

#### 3.7 通用分配与审计示例

```json
{
  "model_allocation": {
    "policy_snapshot": {
      "allowlist": ["<selected-model-id>", "<fallback-model-id>"],
      "budget": "<budget-or-unknown>",
      "premium_concurrency": "<slots-or-unknown>",
      "disabled": ["<disabled-model-id>"],
      "privacy": "<policy-level>"
    },
    "assignments": {
      "<task-id>/<subtask-id>/1": {
        "role": "Executor",
        "model": "<selected-model-id>",
        "evidence": "runtime_validated",
        "reason": "tool+context+cost match",
        "warnings": ["<unknown-price>", "<probe-required-if-any>"],
        "fallback": "<fallback-model-id-or-wait_user>",
        "budget_cost": "<estimated-cost-or-unknown>"
      }
    },
    "audit": {
      "inputs": ["<policy-ref>", "<allowlist-ref>", "<catalog-ref>", "<probe-ref>"],
      "decision": "selected after hard filter and scoring",
      "rematch": ["<none-or-reason>"]
    }
  }
}
```

审计记录至少要能回答：为什么选这个模型、为什么没选其他候选、预算如何计算、是否 probe、是否 runtime validated、何时需要 rematch。

#### 3.8 配置、readback、probe、runtime acceptance 与 rematch 门禁

- **配置**：只写 OpenClaw 支持的字段，不引入未支持的顶层扩展；模型与 Agent 绑定必须可由当前 schema 表达。
- **readback**：写入后必须回读配置，核对 allowlist、模型绑定、会话可见性、只读边界与调度权限是否与预期一致。
- **targeted probe**：仅在证据不足、成本可接受且 probe 结果能显著降低不确定性时使用；probe 不得替代 runtime validation。
- **runtime acceptance**：必须由真实非 Main 子 Agent 的运行结果证明；目录可见、ready-marker、静态声明都不算。
- **rematch / escalate / de-escalate**：当证据等级变化、限流、失败、预算变化、权限变化或模型不可用时，Main 应重新匹配；必要时升级、降级或回退。

`unknown` 是合法状态，但不可长期悬置：若未知信息影响 Assignment 成败，必须通过 probe、readback 或 runtime 验收把它收敛掉；若不能收敛，则应进入回退或等待用户策略。

```json
{
  "model_allocation": {
    "source": "user_available_models",
    "assignments": {
      "<agent-id>": {
        "model": "<selected-model-id>",
        "reason": "<capability-and-cost-match>",
        "evidence": "<configured_allowed|targeted_probe|runtime_validated>",
        "fallback": "<fallback-model-id-or-wait_user>",
        "warnings": ["<unknown-metadata-warning>"]
      }
    },
    "audit": "<allocation-audit-ref>",
    "warnings": ["<unknown-metadata-warning>"]
  }
}
```

### 4. `openclaw.json` 配置映射

以下仅为概念映射；真实字段以当前 OpenClaw schema 为准。不得因为示例文件存在就假定运行时已启用。

```jsonc
{
  "agents": {
    "defaults": {"workspace": "<workspace>", "model": {"primary": "<main-model>"}},
    "list": [
      {"id": "main"},
      {"id": "res_1", "workspace": "<workspace>/agents/res_1"},
      {"id": "exe_1", "workspace": "<workspace>/agents/exe_1"},
      {"id": "dbg_1", "workspace": "<workspace>/agents/dbg_1"},
      {"id": "judge", "workspace": "<workspace>/agents/judge"}
    ]
  }
}
```

配置必须同时设置 Main 的子 Agent allowlist、Session 可见性、必要调用权限和 Worker 对状态总线的只读边界，并通过第 8 步验收。

### 5. 提示词 (Prompt) 策略

* **Main**：包含 MAM 读取、硬覆盖与完整评分、Researcher 数量、两层 DAG、渐进 `DAG_Update`、Assignment、统一回执/Digest/`State_Patch` 验证、串行状态写入、去重冲突、隔离集成、两级重试、Judge/MAM/归档门禁。
* **Worker**：只读 `state.md`，核对 Task/Subtask/Assignment/依赖/范围；不得物理写状态；仅返回完整 `WORKER_COMPLETED` JSON、证据和目标章节的 `State_Patch`。
* **Claude Code / Codex**：仅当用户明确指定本轮或具体子任务时，通过用户本地 CLI 调用；不使用 ACP，不创建特殊 OpenClaw Agent ID。用户未指定时默认 `openclaw_native`。Main 先检查 PATH、启动、认证、工作目录/写入/命令权限与非交互支持，状态为 `AVAILABLE / NOT_INSTALLED / NOT_AUTHENTICATED / NOT_CONFIGURED / UNSUPPORTED_MODE / START_FAILED`。不可用时告知原因、不安装、不保存凭证、不伪装调用，默认回退原生 Executor 并记录；若用户禁止回退则暂停。CLI 仍遵循同一隔离、验收、回执与两级熔断规则，Evidence 记录版本、退出码、测试和 diff 摘要，不记录凭证。

`<task_folder>/MAM_state.md` 固定精简 Schema：

```markdown
# MAM Project Memory

## Project Overview
- **Project_Name**: `<project-name>`
- **Project_Path**: `<task_folder>`
- **Purpose**: `<purpose>`
- **Current_Stage**: `<stage>`
- **Last_Completed_Task_ID**: `<task-id>`
- **Last_Updated_At**: `<timestamp>`

## Active Principles
- `<active-principle>`

## Active Boundaries
- **Allowed**: `<allowed>`
- **Forbidden**: `<forbidden>`
- **Safety**: `<safety-boundary>`

## Current Architecture
- `<current-architecture>`

## Key Decisions
### <decision-title>
- **Background**: `<background>`
- **Decision**: `<decision>`
- **Decision_Reason**: `<reason>`
- **Alternatives_Not_Used**: `<alternatives>`
- **Impact**: `<impact>`

## Recent Important Changes
- `<important-change>`

## Acceptance Evidence
- `<durable-evidence>`

## Risks And Technical Debt
- `<risk-or-debt>`

## Open Questions
- `<question-or-None>`

## Next Starting Point
- `<next-starting-point>`
```

MAM 不得包含完整聊天/状态副本、全量 DAG、调度与重试流水、大日志、临时 Session/Run/Assignment 标识、活跃节点状态、调度器状态、瞬态执行证据、失效重复决定或凭证。写后验证文件可读、固定章节齐全、`Last_Completed_Task_ID` 匹配、无敏感信息/运行日志，且足以回答项目现状、原则边界、近期变更、可用证据与下一步。

---

## 六、 V5.6 架构升级：Coordinator/Integration 运行时调度模型 (V5.6 Upgrade: Coordinator/Integration Runtime Orchestration)

> **版本标记**：本章为 V5.6 → V5.6 增量升级规范。所有 V5.6 条款（一至五章）继续有效；本章仅在其上新增或明确 Coordinator/Integration 运行时调度层的职责、边界与协议。

---

### 6.1 升级动因与核心目标 (Motivation & Goals)

V5.6 中 Main 既是唯一状态写入者，又需逐节点跟进 Delivery 阶段的波次派发、重试仲裁与产物汇总，导致 Main 上下文消耗过高，且调度细节与状态主权职责耦合。

V5.6 目标：
- **Main 保留状态主权**，但将 Delivery 阶段的运行时编排权有界授权给 Coordinator/Integration 角色。
- 引入 **Delegated_Runtime_Authority**（有界、可撤销的运行时编排授权），使 Coordinator 可在授权范围内独立驱动波次、处理局部重试、汇总阶段产物，无需每步回报 Main。
- 引入 **Stage Package / Final Coordinator Package**、**Coordinator Ledger Artifact** 与 **Judge Direct Verdict Channel**，使 Main 只需在关键节点（授权、包接受、归档）介入，而非参与每一节点调度。
- 保留 EXE 证据边界、MAM 读写门禁、两级熔断与 Judge 独立裁决等全部 V5.6 护栏。

---

### 6.2 Main 状态主权边界（V5.6 精化）(Main State Sovereignty)

V5.6 中 Main 的专属职责精化为以下最小必要集，不得再扩展：

| 职责 | 说明 |
|---|---|
| **`<state-file>` 初始化** | 任务开始时写入 Signals、Research DAG 骨架与契约 |
| **授权边界设定** | 为每个 Assignment 绑定 Agent ID、Scope、Acceptance；发布 `Delegated_Runtime_Authority` |
| **Research 回执验收与串行写入** | 接受 RES Worker 回执，串行写入 Blackboard，触发 DAG_Update |
| **Coordinator Package 接受** | 在 Delivery 阶段结束时接受 Final Coordinator Package，串行写入状态 |
| **Judge 裁决写入** | 接受 Judge Direct Verdict，写入 `[4] Evaluation` |
| **MAM 更新与归档** | 按 V5.6 第 7 步门禁，Judge PASS 后更新 MAM、归档、重置 |
| **熔断升级决策** | 当 Coordinator 上报熔断事件时，Main 决定退回 Researcher 还是 WAITING_USER |

**Main 在 Delivery 阶段不再逐节点派发 EXE/DBG；此职责委托给 Coordinator。** Main 只在收到 Final Coordinator Package 时做一次完整验收写入。

**Main 不得因 Coordinator 存在而放弃状态主权**：Coordinator 没有 `<state-file>` 写权限，所有状态变更仍须经 Main 串行提交。

---

### 6.3 Delegated_Runtime_Authority（有界运行时编排授权）

`Delegated_Runtime_Authority` 是 Main 在 Research PASS 且 Delivery Plan 通过验收后，向 Coordinator/Integration 发布的有界、可撤销授权记录，内容至少包含：

```markdown
## Delegated_Runtime_Authority
- **Task_ID**: `<task-id>`
- **Granted_To**: `<coordinator-agent-id>`
- **Granted_At**: `<timestamp>`
- **Scope**: `Delivery DAG 运行时编排；EXE/DBG 节点波次派发；局部重试（上限 Attempt 3）；产物汇总与 Stage Package 生成`
- **Forbidden**: `写入 <state-file>；修改 Research DAG；扩展授权范围；替代 Judge 裁决；替代 Main 的 Package 接受；超出已授权文件范围`
- **Revocation_Condition**: `熔断触发（Attempt 4 同根因失败）；Coordinator 越权行为；Main 显式撤销`
- **Expires_On**: `Final Coordinator Package 被 Main 接受，或任务终止`
```

`Delegated_Runtime_Authority` 记录在 `state.md` 的 `[1] 调度计划` 中，并在 Coordinator Ledger Artifact 中引用。Coordinator 不得自行修改或扩展该授权。

---

### 6.4 Coordinator/Integration 运行时调度职责

Coordinator（可复用 Integration 角色实例担任）在 `Delegated_Runtime_Authority` 有效期内负责：

1. **波次派发（Wave Dispatch）**：按 Delivery DAG 拓扑，将 READY EXE 节点同波次派发给对应 Executor Agent；依赖未满足的节点保持 `BLOCKED_BY_DEPENDENCY`。
2. **局部重试仲裁（Local Retry Arbitration）**：在授权范围内对 FAIL 节点执行最多 Attempt 1→3 的重试；Attempt 4 同根因失败时停止，生成熔断事件并上报 Main，不得自行扩展重试次数。
3. **产物汇总与冲突仲裁（Artifact Aggregation & Conflict Resolution）**：按 V5.6 第 4 步的 `Model_Capability_Order` 仲裁实现冲突；记录双方、模型等级、采用方案和理由。
4. **Stage Package 生成（Stage Package Reporting）**：每个 Delivery 波次结束后生成 Stage Package，记录本波次产物路径、证据摘要、节点状态与局部重试记录。
5. **Final Coordinator Package 生成**：全部 EXE/DBG/Integration/Regression 节点 PASS 后，汇总所有 Stage Package，生成 Final Coordinator Package 并提交 Main 接受。
6. **Coordinator Ledger Artifact 维护**：全程维护一份不可变的调度台账（Ledger），记录每次波次派发、重试、产物汇总与冲突决策，供 Judge 和 Main 审计。

**Coordinator 不得执行的操作：**
- 物理写入 `<state-file>`
- 扩展 `Delegated_Runtime_Authority` 的授权范围
- 替代 Main 做 Package 接受或 Assignment 绑定
- 替代 Judge 做最终裁决
- 在熔断后自行扩展重试或绕过 Main 决策

---

### 6.5 Stage Package 与 Final Coordinator Package 规范

**Stage Package**（每波次结束时生成，保存在 `<task_folder>/artifacts/COORDINATOR/stage-<n>.md`）：

```markdown
## Stage Package — Wave <n>
- **Task_ID**: `<task-id>`
- **Wave**: `<n>`
- **Nodes**: `<node-id-list>`
- **Status_Summary**: `<PASS/FAIL/PARTIAL>`
- **Artifacts**: `<path-list>`
- **Evidence_Summary**: `<brief>`
- **Retry_Log**: `<attempt-summary-per-node>`
- **Conflicts_Resolved**: `<conflict-summary-or-none>`
- **Generated_At**: `<timestamp>`
```

**Final Coordinator Package**（全部 Delivery 节点结束时生成，保存在 `<task_folder>/artifacts/COORDINATOR/final-package.md`）：

```markdown
## Final Coordinator Package
- **Task_ID**: `<task-id>`
- **Coordinator_Agent_ID**: `<agent-id>`
- **Delegated_Runtime_Authority_Ref**: `<dra-record-ref>`
- **Stage_Packages**: `<stage-package-path-list>`
- **All_Nodes_Status**: `<node-id: PASS/FAIL list>`
- **Unresolved_Failures**: `<list-or-none>`
- **Artifact_Index**: `<final-artifact-path-list>`
- **Coordinator_Ledger_Ref**: `<ledger-path>`
- **Ready_For_Judge**: `TRUE | FALSE`
- **Generated_At**: `<timestamp>`
```

Main 在接受 Final Coordinator Package 时，按 V5.6 第 4 步的回执验证流程检查：产物存在、节点状态完整、无未解决熔断、Ledger 可读。验证通过后串行写入 `state.md` 对应章节，并解锁 Judge。

---

### 6.6 Coordinator Ledger Artifact

Coordinator Ledger 是 Delivery 阶段的不可变调度台账，保存在 `<task_folder>/artifacts/COORDINATOR/ledger.md`，记录：

- 每次波次派发：时间、节点列表、分配的 Agent ID 与 Session Key
- 每次局部重试：节点 ID、Attempt、根因签名、结果
- 每次产物汇总：冲突描述、模型等级、采用方案
- 熔断事件（若有）：节点 ID、根因、上报 Main 的时间戳
- `Delegated_Runtime_Authority` 引用：授权记录位置与版本

Ledger 只允许追加，不得覆盖历史记录。Judge 和 Main 在验收时均可直接引用 Ledger 作为调度证据。

---

### 6.7 Judge Direct Verdict Channel

在 V5.6 中，Judge 在完成独立核验后，通过 **Judge Direct Verdict Channel** 直接向 Main 提交裁决回执，格式与 V5.6 `WORKER_COMPLETED` 回执兼容，但新增以下字段：

```json
{
  "coordinator_package_ref": "<final-coordinator-package-path>",
  "coordinator_ledger_ref": "<ledger-path>",
  "delegated_runtime_authority_ref": "<dra-record-ref>",
  "verdict": "PASS | REJECTED",
  "critical_issues": [],
  "gate_checks": {
    "all_exe_dbg_pass": true,
    "integration_regression_pass": true,
    "no_unresolved_failures": true,
    "coordinator_package_valid": true,
    "ledger_auditable": true,
    "artifacts_exist": true,
    "security_boundary_intact": true
  }
}
```

Main 接受 Judge Direct Verdict 后，按 V5.6 第 6 步门禁写入 `[4] Evaluation`，并触发 MAM 更新与归档流程。

**Judge 核验范围在 V5.6 中扩展为包含：**
- Final Coordinator Package 完整性与 Ready_For_Judge = TRUE
- Coordinator Ledger Artifact 可读性与关键记录完整性
- `Delegated_Runtime_Authority` 未被越权使用
- 所有 V5.6 第 6 步的原有门控条件

---

### 6.8 Main Minimal State Writes（V5.6 最小写入集）

在引入 Coordinator 后，Main 的 `state.md` 写入操作精简为以下必要集：

| 写入时机 | 写入内容 |
|---|---|
| 任务初始化 | `[0] Signals`、`[1]` Research DAG 骨架、`[2]` 契约 |
| RES Worker 回执验收后 | `[2]` Research Blackboard 对应章节；`[1]` 节点状态更新；DAG_Update |
| Delivery Plan 通过后 | `[1]` Delivery DAG 骨架；`Delegated_Runtime_Authority` 记录 |
| Final Coordinator Package 接受后 | `[2]` EXE-ARCH/EXE-PROMPT/EXE-MAM 节点状态；产物索引摘要 |
| Judge Direct Verdict 接受后 | `[4] Evaluation`；`[1] JUDGE` 节点状态 |
| MAM 更新 + 归档后 | `[5] Archive`；`[0] Status = COMPLETED`；重置 |
| 熔断上报后 | `[1]` 熔断节点状态；`[2]` 熔断事件摘要；触发 Researcher 或 WAITING_USER |

Main 不得在 Delivery 阶段写入每个 EXE/DBG 节点的逐步调度记录；此类细节由 Coordinator Ledger 承载。

---

### 6.9 V5.6 护栏的继承与保留

以下 V5.6 机制在 V5.6 中**完整保留，不受 Coordinator 委托影响**：

- **`<task_folder>` 定义与选择原则**（五章第 1 节）：Coordinator 不得修改 Task_Folder。
- **EXE 证据边界**：EXE 节点的执行证据只能作为 execution evidence 写入回执；Coordinator 汇总时不得将其升格为 acceptance 或 QA gate。
- **MAM 读写门禁**：MAM 仍仅在 Judge PASS 后、由 Main 更新一次；Coordinator 不得触发 MAM 写入。
- **两级熔断**：Coordinator 执行第一级（执行链重试 Attempt 1→3）；Attempt 4 同根因失败后上报 Main，由 Main 决定是否触发第二级（退回 Researcher）。Coordinator 不得自行启动第二级熔断。
- **Judge 独立裁决**：Coordinator 不得替代 Judge，不得在 Final Coordinator Package 中声称已完成裁决。
- **Worker 只读 `state.md`**：Coordinator 同样只读 `state.md`，所有状态变更通过回执与 State_Patch 提交 Main。
- **DAG_Update 幂等性**：Coordinator 如需建议 DAG 变更，须在 Stage Package 或 Final Package 中以 `proposed_dag_updates` 字段提交 Main 决策，Main 串行应用后方生效。
- **并行 Executor 隔离**：并行 EXE 节点仍使用独立产物目录，Coordinator 不得打破隔离边界。
- **配置后运行时验收**（五章第 8 节）：Coordinator 角色的引入不豁免运行时验收要求；Coordinator 本身的 Assignment 绑定、Session 可见性与授权范围须通过同等验收。
