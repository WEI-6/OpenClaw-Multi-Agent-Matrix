# OpenClaw 多 Agent 矩阵系统 v5.1 实施规范

> 基于版本：`Architecture_v5.0.md`
> 用途：v5.0 → v5.1 变更的具体规则、Schema 和实施要求。

---

## 一、改进目标

1. Main 依据复杂度评分可靠选择执行模式。
2. FULL 模式任务强制拆分子任务并采用两层 DAG。
3. 可并行子任务在职能池容量内并行派发。
4. 每个子任务具备明确输入、输出、依赖和验收标准。
5. 并行结果经集成、回归检查和 Judge 门禁后完成。
6. 复杂任务开始前读取项目长期记忆，完成后写入更新。

---

## 二、复杂度调度

### 2.1 硬覆盖规则（优先于分数）

| 条件 | 结果 |
|---|---|
| 用户明确要求 Matrix / 多 Agent / 并行 / 独立 Judge | `FULL` |
| 涉及破坏性、不可逆、外部发布/消息/事务、凭证、隐私或权限边界变更 | 先 `WAITING_USER`，确认后再评分调度 |
| 纯闲聊、简单只读查询，不需要文件/工具/独立验证 | `MAIN_ONLY` |
| 用户明确指定执行方式或禁止拆分 | 遵循用户要求，保留必要安全门禁 |

### 2.2 多维加法评分

| 维度 | 分值 |
|---|---:|
| 安全与失败影响（破坏性、隐私、权限、生产、高失败成本） | `+3` |
| 多文件或跨组件 | `+2` |
| 实现与工具执行 | `+2` |
| 独立质量验证 | `+2` |
| 独立交付物与并行价值 | `+2` |
| 集成复杂度 | `+2` |
| 研究与需求不确定性 | `+1` |
| 长时运行与恢复敏感 | `+1` |

Main 必须在 `state.md` 中记录每个实际加分项，不能只写总分。

### 2.3 模式阈值

```
0–2  → MAIN_ONLY
3–5  → MINIMAL
≥6   → FULL
```

**MAIN_ONLY**：Main 直接处理，不强制建立 DAG，执行与任务风险匹配的最小验证。

**MINIMAL**：至少分配 1 个子 Agent；是否增加 Debugger/Judge 由 Main 根据风险决定；不强制两层 DAG，但必须记录 Assignment 和验收结果。

**FULL**：
- 强制子任务拆分，强制两层 DAG。
- 第一层 Research DAG，第二层根据研究结果增量建立 Delivery DAG。
- 可并行节点在职能池容量内并行派发。
- 必须包含 Debugger 检查和 Judge 最终门禁。
- Main 不得在自己的 Session 中串行完成完整复杂任务。

### 2.4 Researcher 数量裁定（FULL 模式）

```
6–7 分   → 1 个 Researcher
8–10 分  → 2 个 Researcher 并行
≥11 分   → 3 个 Researcher 并行
```

以下情况可上调 Researcher 数量（上限 3 个）：跨多专业领域、方案存在明显争议、风险与实现路径需独立分析、用户明确要求多角度研究。Main 低于建议数量时须在 `state.md` 说明原因。

### 2.5 评分示例

```
示例一：修改一个配置字段
  实现与工具执行 +2 → 总分 2 → MAIN_ONLY

示例二：修改三个文件并运行测试
  多文件或跨组件 +2，实现与工具执行 +2 → 总分 4 → MINIMAL

示例三：跨模块重构，需编码、独立测试、集成
  多文件 +2，实现 +2，独立质量验证 +2，并行价值 +2，集成复杂度 +2 → 总分 10 → FULL，2 个 Researcher

示例四：生产系统迁移
  安全与失败影响 +3，多文件 +2，实现 +2，独立质量验证 +2，集成复杂度 +2，长时运行 +1 → 总分 12 → FULL，3 个 Researcher
```

### 2.6 职能池并发上限

```
Researcher：最多 3 个并行
Executor：最多 3 个并行（openclaw_native / claude_code / codex 合计）
Debugger：最多 3 个并行
Integration：最多 3 个并行，同一集成产物只能有 1 个有效节点
Judge：最多 3 个并行实例，同一任务最终裁决默认只激活 1 个
```

实际并发数取 READY 节点数、可用 Agent 数、职能池上限三者的最小值。不设模型级并发限制；限流/超时按任务失败和重试规则处理。子任务最小粒度由 Main 动态决定。

### 2.7 子任务最小字段

每个子任务必须包含：`Subtask_ID`、名称、目标、输入与依赖、输出路径或章节、负责角色与 Agent ID、允许修改的文件范围、禁止修改的文件范围、验收标准、当前状态、重试次数、开始/完成时间、Session Key / Run ID、结果摘要与证据。

### 2.8 局部重试与两级熔断

失败计数键：`Task_ID + Subtask_ID + Stage + Root_Cause_Signature`，不同子任务/阶段/根因互不累计。

**第一阶段：执行/调试链路重试**

Executor、Debugger、Integration 或集成回归失败时：
1. 记录失败阶段、错误证据和 `Root_Cause_Signature`。
2. 只暂停并重试受影响链路；其他无依赖链路继续运行。
3. 每次重试增加 Assignment Attempt。

```
Attempt 1（初始）→ Attempt 2（重试 1）→ Attempt 3（重试 2）→ Attempt 4（重试 3）
```

Attempt 4 仍因同一根因失败：执行阶段熔断，退回 Researcher。

**第二阶段：退回 Researcher 修订**

1. Main 将失败证据、现有产物和测试结果交给 Researcher。
2. Researcher 必须提供实质性修改方案，不得重复原方案。
3. Main 根据新方案更新第二层 DAG；新方案可重置对应链路的执行重试计数。
4. Researcher 修订最多 3 次。

第 3 次修订后执行链路仍失败，或 Researcher 连续 3 次未提出可执行方案：进入 `WAITING_USER`。

Main 在 `WAITING_USER` 时须说明：失败子任务与阶段、3 次局部重试记录、3 次 Researcher 修订记录、每次根因与证据、当前可选方案与需用户决定的事项。

**根因变化规则**：新签名仅在错误机制、失败位置或解决路径发生实质变化时才允许使用；Main 必须防止频繁更名逃避熔断。节点 PASS 后当前根因计数结束，历史记录保留。

**局部隔离**：

```
EXE-A / DBG-A 失败
→ 只重试 A 链路
→ EXE-B / DBG-B 继续
→ 与 A 无依赖的节点继续
→ 依赖 A 的节点保持 BLOCKED_BY_DEPENDENCY
```

Integration / Judge 全部必要依赖恢复 PASS 后才能启动。

---

## 三、`state.md` 与两层 DAG

### 3.1 Main 单写入规则

- `state.md` 是唯一主状态总线，仅 Main 可写入、更新、重置或归档。
- 所有 Worker（Researcher、Executor、Debugger、Judge、Claude Code、Codex）只读 `state.md`，执行完毕后向 Main 返回结构化回执。
- Main 验证回执后串行写入；Worker 并行计算，状态提交串行。
- 不采用 Revision、CAS 或文件锁。

### 3.2 Worker 统一回执 Schema

#### Assignment ID

```
Assignment_ID = Task_ID / Subtask_ID / Attempt
例：MATRIX-001/EXE-A/1
```

规则：同一 Subtask_ID 同一时间只能有一个有效 Assignment；首次 Attempt = 1；仅重试时递增；派发后不可更改；Session Key 和 Run ID 必须绑定到 Assignment。

#### JSON Schema

```json
{
  "schema_version": "1.0",
  "event_type": "WORKER_COMPLETED",
  "task_id": "MATRIX-001",
  "subtask_id": "EXE-A",
  "assignment_id": "MATRIX-001/EXE-A/1",
  "attempt": 1,
  "role": "executor",
  "agent_id": "exe_1",
  "executor_type": "openclaw_native",
  "session_key": "agent:exe_1:subagent:...",
  "run_id": "...",
  "status": "PASS",
  "target_section": "Delivery DAG/EXE-A",
  "completed_at": "2026-08-17T14:00:00+08:00",
  "result_summary": "完成模块 A，通过单元测试。",
  "artifacts": [
    { "path": "src/module_a.py", "type": "source", "sha256": "optional" }
  ],
  "evidence": [
    { "type": "test", "summary": "12 tests passed", "path": "reports/exe-a-tests.txt" }
  ],
  "errors": [],
  "next": ["DBG-A"],
  "result_digest": "sha256-of-canonical-result-payload"
}
```

**必填字段**：`schema_version`、`event_type`、`task_id`、`subtask_id`、`assignment_id`、`attempt`、`role`、`agent_id`、`executor_type`、`session_key`、`run_id`、`status`、`target_section`、`completed_at`、`result_summary`、`artifacts`、`evidence`、`errors`、`next`、`result_digest`。

`status` 允许值：`PASS` / `FAIL` / `BLOCKED` / `CANCELLED`。

`executor_type` 允许值：`openclaw_native` / `claude_code` / `codex`。

`artifacts`、`evidence`、`errors`、`next` 无内容时用空数组，不可省略。

#### Result Digest

计算范围：Assignment ID、Status、Result Summary、Artifacts、Evidence、Errors、Next。按固定字段顺序标准 JSON 序列化后计算 SHA-256。Main 必须重新计算，不得直接信任 Worker 提供的值。

#### 大产物处理

`state.md` 只保存摘要、状态和产物索引；代码、报告、日志存放在任务文件夹；`artifacts.path` 必须指向实际存在的文件；大段日志不得写入 `result_summary` 或 `evidence.summary`；敏感信息不得写入回执或状态总线。

#### 回执拒绝条件

Main 遇以下情况必须拒绝：Task ID 或 Subtask ID 不匹配；Assignment ID 不是当前有效 Assignment；Agent ID / Session Key / Run ID 与分配记录不一致；缺少必填字段；Status 不在允许范围内；Result Digest 重算不一致；Artifacts 声称存在但文件不存在；Worker 建议写入未授权 Target Section。

### 3.3 重复完成事件处理

每个 Assignment 在 `state.md` 记录：

```markdown
- **Assignment_ID**: `MATRIX-001/EXE-A/1`
- **Status**: `RUNNING`
- **Result_Accepted**: `FALSE`
- **Result_Digest**: `PENDING`
```

| 情况 | 处理 |
|---|---|
| 正常完成（Result_Accepted=FALSE，验证通过） | 写入结果，Result_Accepted=TRUE，记录 Digest，更新节点状态，重新计算就绪集 |
| 重复事件（Assignment_ID 相同，Result_Accepted=TRUE，Digest 相同） | 不写入，不解锁下游，记录 `Duplicate_Event_Ignored` |
| 同一 Assignment 返回不同 Digest | 标记 `RESULT_CONFLICT`，拒绝自动覆盖，保留冲突回执，由 Main/Debugger 检查 |
| 旧 Attempt 迟到 | 标记 `LATE_RESULT`，默认不写入，不改节点状态，保留摘要作诊断证据 |

分配阶段防重复：READY 且无有效 Assignment → 允许派发；RUNNING/PASS → 禁止派发；FAIL/BLOCKED 需重试 → Attempt+1 后派发。

### 3.4 Main 串行写入流程

```
收到 Worker 完成事件
→ 解析并校验 Worker Receipt JSON
→ 核对 Task_ID / Subtask_ID / Assignment_ID
→ 核对 Agent_ID / Session_Key / Run_ID
→ 重新计算 Result_Digest
→ 执行重复/冲突/迟到判断
→ 检查 Status、Evidence、Artifacts
→ 将摘要写入 state.md 对应章节
→ 设置 Result_Accepted 和 Result_Digest
→ 更新节点状态
→ 重新计算 READY 节点
→ 派发新解锁的子任务
```

每个 Worker 完成后立即处理，无需等待同批次全部完成。

### 3.5 两层 DAG 结构

```
第一层：Research DAG
  多个 Researcher 并行
    ↓ Main 逐个接收、验证、写入
  Main 形成第二层 DAG（可在部分研究完成后提前建立）

第二层：Delivery DAG
  Executor 并行执行
    ↓
  Debugger 检查与局部修复
    ↓
  集成与集成回归
    ↓
  Judge 最终验收
```

Researcher 池：`res_1` / `res_2` / `res_3`，按维度分工（需求与边界 / 技术方案与接口 / 风险与验收）。每个 Researcher 完成后，Main 立即校验回执、写入结果、判断是否可提前解锁第二层节点。

### 3.6 第二层 DAG 内容要求

第二层 DAG 至少明确：实现子任务及依赖；每个 Executor 的产物路径；使用的执行器类型；对应 Debugger；可并行节点；需串行等待的节点；是否需要 Integration 节点；集成回归范围；Judge 验收条件。

### 3.7 DAG_Update Schema（增量更新）

第二层 Delivery DAG 由 Main 维护，使用结构化 `DAG_Update` 增量更新，不整体重写。

```json
{
  "schema_version": "1.0",
  "update_id": "MATRIX-001/DAG-UPDATE/3",
  "task_id": "MATRIX-001",
  "triggered_by": {
    "subtask_id": "RES-2",
    "assignment_id": "MATRIX-001/RES-2/1",
    "result_digest": "..."
  },
  "created_at": "2026-08-17T14:00:00+08:00",
  "add_nodes": [
    {
      "node_id": "EXE-A",
      "node_type": "EXECUTOR",
      "title": "实现模块 A",
      "depends_on": ["RES-2"],
      "status": "READY",
      "preferred_executor": "claude_code",
      "owner_agent": "PENDING",
      "target_section": "Delivery DAG/EXE-A",
      "artifact_scope": {
        "allow": ["src/module_a/**"],
        "deny": ["src/module_b/**"]
      },
      "acceptance": ["unit tests pass", "no unauthorized files changed"]
    }
  ],
  "update_nodes": [],
  "add_edges": [],
  "remove_edges": [],
  "cancel_nodes": [],
  "notes": "RES-2 已确定模块 A 接口，可提前执行。"
}
```

**稳定 Node ID 规则**：推荐命名 `EXE-A`、`EXE-B`、`DBG-A`、`DBG-B`、`INT-1`、`REG-1`、`JUDGE`。Node ID 创建后不得重命名；重试只增加 Attempt，不建新节点；更换执行器保持 Node ID 不变；取消节点保留记录、不复用 ID。

**允许的增量操作**：`add_nodes`、`update_nodes`（非身份字段）、`add_edges`、`remove_edges`（仅未开始节点间）、`cancel_nodes`（仅未开始或已终止节点）。

**禁止操作**：删除已完成节点记录；修改已有 Node ID；将 RUNNING/PASS 节点改回 READY；移除已完成节点的证据；添加形成环的依赖边；修改与运行中 Assignment 冲突的产物范围。

**update_nodes 格式**：

```json
{
  "node_id": "EXE-A",
  "set": {
    "status": "BLOCKED_BY_DEPENDENCY",
    "owner_agent": "exe_1",
    "preferred_executor": "claude_code",
    "acceptance": ["unit tests pass", "integration contract satisfied"]
  },
  "reason": "RES-3 补充了共享接口依赖，需等待 EXE-B。"
}
```

**边格式**：

```json
{ "from": "EXE-A", "to": "DBG-A", "type": "REQUIRES_PASS" }
```

边类型：`REQUIRES_PASS`（前置必须 PASS）、`REQUIRES_COMPLETION`（前置结束即可）、`INFORMS`（仅提供上下文，不阻塞调度）。

**Main 应用 DAG_Update 流程**：

```
收到 Researcher 结果
→ 验证并写入 Research DAG
→ 生成 DAG_Update
→ 检查 Node ID 冲突
→ 检查输出目录独立性
→ 检查依赖环
→ 检查是否违反已运行节点状态
→ 合并进 Delivery DAG
→ 记录 update_id 和 triggered_by
→ 重新计算 READY 节点
→ 并行派发新解锁节点
```

**幂等性**：`Update_ID = Task_ID/DAG-UPDATE/Sequence`，Main 维护已应用列表。相同 ID 相同内容：忽略；相同 ID 不同内容：标记 `DAG_UPDATE_CONFLICT`；不同 ID 相同 Node ID：定义完全相同则忽略，否则标记 `NODE_DEFINITION_CONFLICT`。

### 3.8 Executor 产物隔离

每个 Executor 在 Main 指定的独立目录内创建产物，不修改其他 Executor 的文件：

```
<task_folder>/artifacts/EXE-A/
<task_folder>/artifacts/EXE-B/
<task_folder>/artifacts/EXE-C/
```

若任务必须修改现有项目文件，禁止多个并行 Executor 直接操作同一工作树。须采用：独立工作树/分支；独立补丁文件；独立生成目录；或 Executor 只输出 patch，由 Integration 统一应用。

Executor 完成并返回有效回执后必须：停止修改文件；退出运行状态；释放工作目录和工具会话；节点状态变为 `PASS` / `FAIL` / `BLOCKED` / `CANCELLED`。进入 Integration 前，所有被集成的 Executor 必须已退出。

### 3.9 集成节点与冲突仲裁

多个子任务需合并时，Main 创建独立 Integration 节点。Integration 启动前所有相关 Executor 必须完成并退出；集成者取得集成产物的独占修改权；集成完成后前序 Executor 不得再修改已交付文件；后续修复须由 Main 创建新节点。

建议集成输出目录：`<task_folder>/artifacts/INTEGRATION/`

**冲突仲裁规则（按模型能力排序）**：

Main 在任务开始时维护 `Model_Capability_Order`（Tier 1 最强 / Tier 2 主力 / Tier 3 轻量）。

1. 无法同时保留的实现冲突，按模型能力等级排序。
2. 优先采用能力等级更高的模型方案；低等级方案不得覆盖高等级方案，但可作补充参考。
3. 同等级模型冲突交给 Judge 仲裁。
4. 用户明确要求、安全边界或已冻结接口是所有候选方案的前置约束；不满足约束的方案直接淘汰，不进入能力排序。
5. Integration 必须在结果中记录冲突双方、各自模型等级、最终采用方案和仲裁理由。

### 3.10 节点类型与启动条件

**Debugger**：在对应 Executor PASS 后解锁；执行逻辑/边界/性能/安全检查；只返回检查结果，不直接写 `state.md`；失败时只重开受影响链路。

**集成回归节点**：集成完成后执行；必须检查跨模块接口、构建结果和整体行为。

**Judge**：仅在以下全部满足后启动：所有必要 Executor 完成；所有必要 Debugger 通过；集成节点通过或明确标记不需要；集成回归通过或明确标记不需要；无未解决失败或阻塞。

### 3.11 `state.md` 结构建议

```markdown
## Research DAG
### RES-1
### RES-2
### RES-3

## Delivery DAG
### EXE-A
### EXE-B
### DBG-A
### DBG-B
### INTEGRATION
### REGRESSION
### JUDGE
```

每个节点记录：状态、依赖、Agent ID、Session/Run、输出摘要、证据路径、错误和下一步。所有字段由 Main 根据 Worker 回执更新。

### 3.12 恢复原则

恢复时以 `state.md` 中 Main 最后成功写入的状态为准：已写入结果不重复执行；已完成但未写入的结果从对应 Session 重新获取；运行中 Worker 先核对状态再决定等待/恢复/重试；重复回执按 `Assignment_ID + Result_Accepted + Result_Digest` 处理；重复 DAG_Update 按 `Update_ID` 处理。不引入 Revision、CAS 或文件锁。

### 3.13 验收要求

实施前必须通过：

- 三个 Researcher 同时运行，不同顺序完成时 Main 均能正确写入。
- 单个 Researcher 完成后可提前解锁第二层任务。
- 多个 Executor 并行并使用独立目录/工作树/patch，不修改彼此文件。
- 进入 Integration 前所有相关 Executor 已退出。
- Integration 获得集成产物独占修改权。
- 冲突仲裁遵循模型能力排序；用户要求/安全边界/冻结接口是所有候选方案的前置约束。
- 单个执行链失败时，其他链路不受阻塞。
- Worker 无法直接写 `state.md`。
- Main 可根据 Worker 回执连续更新主总线。
- Debugger、集成回归和 Judge 门禁正常执行。
- Main/Gateway 中断后可根据当前总线和 Session 状态恢复。

---

## 四、CLI 调用规则（Claude Code / Codex）

### 4.1 触发条件

仅当用户明确指定本轮任务或某个子任务使用 Claude Code 或 Codex 时才调用对应 CLI。用户未指定时，默认使用 OpenClaw 自身 Executor 子 Agent。

### 4.2 调用方式

- Claude Code → 本地 Claude Code CLI
- Codex → 本地 Codex CLI
- 不使用 ACP，不为 CLI 创建特殊 OpenClaw Agent ID。

Main 启动 CLI 时须指定：任务工作目录、任务说明、允许/禁止修改的范围、产物输出目录、测试与验收要求。CLI 参数以本机实际安装版本的帮助信息为准，不在架构文档中写死参数。

### 4.3 调用前检查

Main 必须先检查：CLI 是否已安装且在 PATH 中；是否能正常启动；是否完成认证配置；工作目录访问权限；文件写入和命令执行权限；是否支持非交互/自动化调用方式。

检查结果：`AVAILABLE` / `NOT_INSTALLED` / `NOT_AUTHENTICATED` / `NOT_CONFIGURED` / `UNSUPPORTED_MODE` / `START_FAILED`。

### 4.4 回退规则

CLI 未安装/未认证/未配置/无法启动时，Main 必须：告知用户检测结果和失败原因；不自行安装 CLI；不自行保存凭证；不伪装已调用；将子任务回退到 OpenClaw 自身 Executor；在节点和报告中记录回退。

```
Requested_Executor: claude_code
Executor_Check: NOT_AUTHENTICATED
Fallback_Executor: openclaw_native
Fallback_Reason: Claude Code CLI 未完成本地认证，已告知用户并回退到 exe_1。
```

若用户明确说明 CLI 不可用时暂停任务，则不执行回退。

### 4.5 用户指定范围映射

Main 将用户指定映射到明确的 `Subtask_ID` 和 `executor_type`：

```json
{
  "subtask_id": "EXE-A",
  "requested_executor": "claude_code",
  "resolved_executor": "claude_code",
  "fallback_allowed": true
}
```

用户只说"这次使用 Claude Code"时，默认指本轮 Delivery DAG 中所有 Executor 和 Integration 节点；Researcher/Debugger/Judge 仍使用 OpenClaw Agent，除非另有明确指定。

### 4.6 产物边界与统一回执

Claude Code/Codex 遵守与普通 Executor 相同的文件隔离规则，结果须包装为第三章的统一 Worker JSON 回执：`executor_type` 填 `claude_code` 或 `codex`；`agent_id` 填负责监督 CLI 的 OpenClaw Executor；CLI 自身 Session 标识放入 Evidence；`artifacts` 列出 CLI 实际生成的文件或 patch；`evidence` 记录 CLI 版本、退出码、测试结果和 diff 摘要。

建议在节点记录：

```
Executor_Type: claude_code | codex
CLI_Version: <版本>
Working_Directory: <路径>
Exit_Code: <退出码>
```

不得将密码、API Key、OAuth Token 写入 `state.md` 或回执。

### 4.7 失败与重试

与普通 Executor 使用同一两级熔断规则。若失败原因是 CLI 不可用或未配置，优先执行 4.4 节回退，不对同一未配置状态重复启动 3 次。
---

## 五、项目长期记忆（`MAM_state.md`）

### 5.1 目的

`MAM_state.md` 是任务文件夹中唯一的项目长期记忆文件，仅供下一次复杂任务启动和规划使用，不保存完整 `state.md` 副本，不记录运行流水。

文件位置：`<task_folder>/MAM_state.md`（唯一，不按 Task ID 建多个文件）。

### 5.2 读取时机

FULL 任务开始前，Main 必须：

```
确定任务文件夹
→ 检查 MAM_state.md
→ 存在则读取，提取项目摘要、原则、边界、有效决定、最近修改、风险和开放问题
→ 与用户本轮需求结合
→ 再进行复杂度评分、Research DAG 和任务拆分
```

MAM 只作规划上下文，不恢复 RUNNING/PASS/FAIL 节点，不恢复 Session/Run/Assignment。MAM 与用户当前指令冲突时以用户指令为准；与磁盘事实冲突时 Main 应先核验实际状态。

### 5.3 固定精简 Schema

```markdown
# MAM Project Memory

## Project Overview
- **Project_Name**: <项目名称>
- **Project_Path**: <任务文件夹绝对路径>
- **Purpose**: <项目目标，1–3 句话>
- **Current_Stage**: <当前阶段>
- **Last_Completed_Task_ID**: <最后完成并写入 MAM 的 Task ID>
- **Last_Updated_At**: <时间>

## Active Principles
- <当前仍需遵守的原则>

## Active Boundaries
- **Allowed**: <允许范围>
- **Forbidden**: <禁止范围>
- **Safety**: <安全与权限边界>

## Current Architecture
- <当前生效的架构、关键组件、数据流或接口>

## Key Decisions
### <决定标题>
- **Background**: <为什么需要决定>
- **Decision**: <当前采用方案>
- **Decision_Reason**: <采用该方案的原因>
- **Alternatives_Not_Used**: <未采用方案及原因>
- **Impact**: <获得什么、放弃什么>

## Recent Important Changes
- <最近完成且后续任务需要知道的关键修改>

## Acceptance Evidence
- <证明当前状态可用的测试、指标或产物路径>

## Risks And Technical Debt
- <仍然存在的风险、限制和技术债>

## Open Questions
- <后续仍需决定的问题；无则写 None>

## Next Starting Point
- <下次任务建议从哪里开始>
```

### 5.4 不写入 MAM 的内容

完整聊天记录；完整 `state.md` 副本；Research/Delivery DAG 全量节点；Agent 调度过程；重试流水和失败日志；大段测试输出；临时 Session Key / Run ID / Assignment ID；重复的历史决定；已失效且不影响后续的细节；密码、Token、API Key 或其他凭证。

### 5.5 唯一更新时间点

每个 Task ID 只更新一次 MAM，更新时机必须同时满足：所有必要子任务完成；Debugger/Integration/Regression 通过；Judge PASS；Main 已确认任务完成；Main 即将重置或归档 `state.md`。

MAM 更新在任务完成之后、`state.md` 重置之前。

任务执行期间（Researcher 完成、单个 Executor 完成、Debugger 中间结果、局部重试、临时失败、DAG 更新等）禁止更新 MAM。

### 5.6 更新流程

```
Judge PASS
→ 检查 Last_Completed_Task_ID 是否等于当前 Task_ID（相同则跳过）
→ 读取旧 MAM（若存在）
→ 从最终 state.md、产物和验收结果提取长期有效信息
→ 删除过时或重复内容
→ 合并新的有效原则、边界、决定、修改、证据、风险和下一步
→ 更新 Last_Completed_Task_ID 和 Last_Updated_At
→ 写入 MAM_state.md
→ 重新读取并检查章节完整性与敏感信息
→ 标记 MAM_Update_Status = PASS
→ 允许归档或重置 state.md
```

更新是整理当前有效摘要，不是追加任务日志。旧决定被替代时只需在新决定中注明 `Supersedes: <旧决定标题>`。

### 5.7 更新失败处理

MAM 写入或回读检查失败时：不得清空 `state.md`；不得将任务标记为已完成重置；Main 先修复写入问题；无法修复则进入 `WAITING_USER` 并说明任务已完成但长期记忆保存失败。

### 5.8 写入后最小验证

- 文件存在且可读。
- 所有固定一级章节存在。
- `Last_Completed_Task_ID` 等于当前 Task ID。
- 不包含密码、Token、API Key 等明显敏感信息。
- 不包含完整 `state.md` 章节或大量运行日志。
- 内容足以回答：项目是什么、当前怎么做、不能做什么、最近改了什么、如何证明可用、下一步是什么。

验收证据写法示例：

```markdown
## Acceptance Evidence
- `pytest`: 128 passed；报告：`reports/tests.txt`
- Integration regression: PASS；报告：`reports/integration.md`
- Judge: PASS；Task ID：`MATRIX-001`
```

---

## 六、总体任务流程

```
用户提交任务
→ Main 确定任务文件夹
→ 读取 MAM_state.md（若存在）
→ 评估任务复杂度（硬覆盖 → 加法评分 → 模式阈值）
→ FULL：强制拆分子任务，建立 Research DAG
→ 并行派发就绪 Researcher
→ 逐个接收研究结果，增量建立 Delivery DAG
→ 并行派发就绪 Executor
→ Debugger 检查各链路
→ 失败链路局部重试/熔断，其他链路继续
→ Integration 合并产物（需要时）
→ 集成回归
→ Judge 最终门禁
→ Judge PASS → 更新 MAM_state.md
→ 归档或重置 state.md
→ 向用户报告结果
```
