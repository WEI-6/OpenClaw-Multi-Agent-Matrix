[中文版](README.md) | **English**

---

# 🚀 OpenClaw-Matrix: Autonomous Multi-Agent Matrix Engine

> **A self-driven, state-centralized, and self-bootstrapping Multi-Agent Collaboration Protocol built on OpenClaw.**

[![OpenClaw](https://img.shields.io/badge/Powered%20by-OpenClaw-blueviolet)](https://github.com/OpenClaw/OpenClaw)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Status](https://img.shields.io/badge/Status-V5.6--Stable-orange)](https://github.com/)

## 🌟 Overview

**OpenClaw-Matrix** is a high-performance orchestration framework designed for complex AI tasks. By implementing a **"Centralized State Bus"** and **"Strict Role Decoupling,"** this system eliminates the common pitfalls of multi-agent workflows — context contamination, logic drifting, and infinite loops.

The core of the system is the `state.md` file, which acts as a "Shared Blackboard" for all agents. **The standout feature is its Self-Bootstrapping capability: simply provide the architecture documentation, and OpenClaw will read, parse, and configure the entire matrix by itself.**

---

## 🧠 Core Philosophy (The V5.6 Protocol)

1. **Main-Only State Authority**: `state.md` is the sole master state bus. Only Main may physically write, update, reset, or archive it. All Workers (Researcher, Executor, Debugger, Judge) are read-only; upon completion they return a `WORKER_COMPLETED` receipt and a proposed `State_Patch` to Main. Main validates and commits serially; Workers may compute in parallel.
2. **Strict Sandbox Isolation**: Each Assignment is bound to an independent Session/Run with a defined file scope and acceptance criteria. Parallel Executors use separate work trees, branches, or artifact directories and must not contend over the same work tree.
3. **Summary & Evidence Driven**: The state bus stores only status, summaries, evidence indexes, and Digests. Large artifacts and logs go in the task folder; sensitive data must not enter receipts or state.
4. **Progressive Two-Layer DAG**: FULL mode first builds a Research DAG, then incrementally builds the Delivery DAG via `DAG_Update` as validated research results arrive. Uncontested nodes can be safely unlocked before the full research batch completes.
5. **Dynamic Model Discovery & Per-Assignment Allocation**: Available models are discovered dynamically at deployment time and profiled for capability, cost, runtime, and compliance. Each Assignment gets a dedicated demand vector match — high-capability/high-cost models are reserved for high-leverage nodes; lightweight models cover research, format checks, and low-risk execution, saving tokens and cost.
6. **Two-Level Retry & Circuit Breaking**: Failures first trigger up to 3 execution-chain retries, then escalate to Researcher for up to 3 substantive revisions. Independent chains are unaffected.
7. **Lifecycle Automation**: From `New_Task_Flag` detection through Judge `PASS`, automatic archiving, and reset, the system manages the full lifecycle autonomously. User confirmation is only required for destructive/irreversible/external/privacy-sensitive actions.
8. **Runtime-Verifiable Dispatch**: Static configuration does not equal an operational system. After configuration, Main must actually invoke a non-Main sub-Agent and prove dispatch with Assignment, Session/Run, receipt, and write-back evidence.

---

## 🏗️ Matrix Role Definitions

| Role Group | ID | Responsibility | State Permission | Key Output |
| :--- | :--- | :--- | :--- | :--- |
| **Controller (Main)** | `main` | Triage, scoring, two-layer DAG, Assignment, receipt validation, serial state writes, archiving | Sole writer of `state.md` | State, dispatch & acceptance records |
| **Research Pool** | `res_1~3` | Requirements/boundaries, design/interfaces, risk/acceptance research | Read-only | Research results, `State_Patch` |
| **Execution Pool** | `exe_1~3` | Isolated implementation, tool execution, testing, artifact generation | Read-only state; authorized artifacts only | Artifacts, evidence, `State_Patch` |
| **Debug Pool** | `dbg_1~3` | Logic, boundary, performance, security, and regression checks | Read-only | QA receipts, `State_Patch` |
| **Integration** | Dynamic node | Exclusive artifact merge, conflict arbitration, regression | Read-only state; exclusive integration scope | Integrated artifacts & receipt |
| **Arbitrator (Judge)** | `judge` | Independent completion gate and final verdict | Read-only | `PASS/REJECTED` receipt |

Pool concurrency limits: Researcher max 3; Executor max 3; Debugger max 3; only 1 effective Integration node per artifact; Judge defaults to 1 instance. Actual concurrency is the minimum of READY nodes, available Agents, and pool limits.

---

## 🚀 Quick Start & Self-Configuration

### 1. Prerequisites

- Ensure [OpenClaw](https://github.com/OpenClaw/OpenClaw) is installed and your LLM API is configured.
- Clone the repository:

```bash
git clone https://github.com/WEI-6/OpenClaw-Multi-Agent-Matrix.git
cd OpenClaw-Multi-Agent-Matrix
```

### 2. The Self-Bootstrapping Command

Send the following instruction to the **Main Agent** so Main reads the architecture document and automatically completes session mapping, bootstrap configuration, workspace initialization, and one verifiable bootstrap flow under the V5.6 protocol:

> *"Please read and parse the `Architecture_V5.6.md` file in the repository root (full relative path: `./Architecture_V5.6.md`). Treat `Architecture_V5.6.md` as the sole architecture source for this Matrix system build and bootstrap, and strictly construct the system according to the roles, state bus, DAG, receipt, dispatch, acceptance, and archiving protocols defined there. Using that file as the base directory, check my OpenClaw Matrix session configuration, initialize the `state.md` bus in the same directory, and strictly apply the role prompts in the sibling `./prompts/` directory before entering the INIT state for a new task."*

**Main should complete or verify the following protocol steps:**
- Verify the configured Agent IDs, Session mappings, and available models, then bind model discovery results to concrete Assignments.
- Confirm that each role is configured in OpenClaw with the corresponding prompt from `/prompts` (`main.md`, `res.md`, `exe.md`, `dbg.md`, `judge.md`).
- Initialize or inspect the `state.md` task bus template in the workspace.
- When Main reads `state.md` during a turn, it handles `New_Task_Flag` according to the protocol; OpenClaw itself does not auto-poll files or enable Matrix just because a file exists.
- Verify that Main has configured restricted sub-Agent dispatch permissions; permissions and allowlists must come from OpenClaw configuration and cannot be granted automatically by reading the architecture document.
- After configuration, Main should run a runtime acceptance check proving independent sub-Agent Sessions, Main-only state writes, and structured receipts.

### 3. Runtime Dispatch Acceptance

> **Important:** `Architecture_V5.6.md` is the sole architecture source for building and bootstrapping the Matrix system, and its protocol definitions must be followed strictly. `Architecture_V5.6.md` and `prompts/` are both anchored at the repository root. Do not interpret `prompts` as `/prompts` at the Linux filesystem root; it means `./prompts/` next to `Architecture_V5.6.md`.


Simply generating a role roster, prompts, and config files **does not mean the Matrix is operational**. Without real sub-Agent Sessions and shared blackboard write-back, the system remains a static deployment only.

After configuration, Main must complete a minimal acceptance task:

1. Create a unique `Task_ID` in `state.md` and write a clear `[Dispatch Plan]`.
2. Actually invoke at least one non-Main sub-Agent per the plan — do not have Main simulate that role's output.
3. Confirm that the role is running in an independent Session and record the actual Agent ID, Session ID, or Run ID.
4. The sub-Agent reads `state.md` (read-only), then returns a `WORKER_COMPLETED` receipt and `State_Patch` only to its authorized section — it does not write directly to the bus.
5. Main validates the receipt, recomputes `result_digest`, and serially commits the `State_Patch` upon successful validation.
6. Judge independently reads the evidence and returns a `PASS / FAIL` verdict. Any missing evidence is treated as runtime dispatch not enabled.

Acceptance requires all of the following simultaneously:
- A unique `Task_ID` and recent update time exist in `state.md`.
- At least one assigned non-Main Agent has a real Session / Run.
- `READY` / `current active` only means the node may be scheduled, not that it has executed; a Delivery/EXE node enters execution only after a real Agent ID, Session Key, and Run ID have been recorded.
- The `Task_ID` in the sub-Agent's receipt matches the shared blackboard, and the result has been written to `state.md` by Main.
- Main or Judge has recorded the final acceptance verdict.

### 4. First-Run Initialization

When a sub-agent starts for the first time, OpenClaw initializes `MEMORY.md` and the `memory/` directory in its workspace. If a sub-agent has never been started, its memory index simply does not exist yet — start the corresponding session once and it initializes automatically. Live runtime files (e.g., `workspace/state.md`) should stay outside the committed repository.

---

## 📂 Directory Structure

```
<project-root>/
├── prompts/              # Role-specific protocol prompts (main.md, res.md, exe.md, dbg.md, judge.md)
├── Architecture_V5.6.md  # V5.6 global architecture spec (the "Source of Truth" for self-configuration)
├── LICENSE
└── README.md / README_EN.md
```

> Runtime files such as `workspace/state.md`, `workspace/history/`, and an optional task-level `MAM_state.md` are generated or maintained in the operator's workspace and should not be treated as shipped source files.

---

## 🔄 Protocol Core Mechanics

### Task Triage & Scoring Modes

After receiving a task, Main applies hard overrides first, then additive scoring to determine the mode, recording each scoring dimension and reason in `state.md`:

| Mode | Score | Use Case |
|---|---|---|
| `MAIN_ONLY` | 0–2 | Pure conversation, read-only lookup, tiny reversible operations; Main handles directly, no sub-Agent dispatched |
| `MINIMAL` | 3–5 | Lightweight Matrix path: Main skips Research DAG, directly creates a Delivery DAG and dispatches real non-Main roles (EXE-* etc.). Chain: Main triage/model allocation → Delivery DAG → EXE-* → DBG-* (risk-gated) → Integration (if needed) → Judge (risk-gated). Main must not produce expert artifacts in its own session. |
| `FULL` | ≥6 | Full two-layer DAG: Research DAG → Delivery DAG → EXE-* → DBG-* → Integration → Judge |

Hard overrides (take precedence over score): user explicitly requests Matrix/independent QA → `FULL`; safety confirmation required → `WAITING_USER`; pure conversation/read-only → `MAIN_ONLY`.

Scoring dimensions (additive): safety & failure impact `+3`; multi-file or cross-component `+2`; implementation & tool execution `+2`; independent quality verification `+2`; independent deliverables & parallel value `+2`; integration complexity `+2`; research & requirement uncertainty `+1`; long-running & recovery-sensitive `+1`.

### Two-Layer DAG

FULL mode uses a Research DAG → Delivery DAG two-layer structure:

```
Research DAG:   RES-1 || RES-2 || RES-3
                         ↓ Main validates each receipt, commits serially
Delivery DAG:   EXE-* → DBG-* → INTEGRATION (if needed) → REGRESSION (if needed) → JUDGE
```

Main incrementally unlocks Delivery nodes via `DAG_Update` as each valid research receipt arrives, without waiting for the full research batch to complete. **Unlocking is not execution**: a Delivery node only enters `RUNNING` after a real non-Main Agent has been dispatched and a real `Agent ID` / `Session Key` / `Run ID` exists. Main must never relabel Main-only edits as EXE work, and must not unlock DBG/Judge nodes that depend on an EXE PASS it did not actually obtain.

### Assignment & WORKER_COMPLETED Receipt

Main creates a unique Assignment (`Task_ID/Subtask_ID/Attempt`) for each subtask. Workers return a standard `WORKER_COMPLETED` JSON receipt containing: `task_id`, `subtask_id`, `assignment_id`, `attempt`, `role`, `agent_id`, `session_key`, `run_id`, `status`, `target_section`, `result_summary`, `artifacts`, `evidence`, `errors`, `next`, `State_Patch` (complete replacement Markdown for the target section that Main should write), and `result_digest` (Main must recompute and verify; the Worker's value is not trusted).

Workers do not write `state.md` directly. Main validates the receipt and serially commits the `State_Patch`.

### Parallel Isolation & Integration

Parallel Executors each use `<task_folder>/artifacts/<subtask-id>/` or an independent work tree; they must not contend over the same tree. Only after all relevant Executors have completed and exited does the Integration node acquire exclusive modification rights over the integration scope, then runs cross-module interface, build, and regression checks.

### Dynamic Model Discovery & Per-Assignment Allocation

Main dynamically discovers available models from the OpenClaw configuration and allowlist, user-supplied model lists, catalog/metadata discovery, and targeted minimal probes. Each model receives a capability, cost, runtime, and compliance profile.

Allocation follows a per-Assignment demand vector (role, context, tooling, budget, privacy, etc.) — hard-filter first, then score. High-capability/high-cost models are reserved for high-leverage or high-difficulty nodes; lightweight models prioritize research, format validation, and low-risk execution, saving tokens and cost. Evidence tiers: `configured_allowed` > `targeted_probe` > `catalog_listed`. Being "visible" does not mean "executable" — runtime acceptance must be proven by a real sub-Agent run.

### Two-Level Retry & Circuit Breaking

The counter key is `Task_ID + Subtask_ID + Stage + Root_Cause_Signature`. Level 1: up to 3 execution-chain retries (Attempt 1→4); if the 4th attempt still fails with the same root cause, the chain trips (`CIRCUIT_OPEN`) and the task is returned to Researcher. Level 2: Researcher provides up to 3 substantively different plans; after the 3rd revision still fails, the system enters `WAITING_USER`. Independent chains are never blocked.

### Judge & Completion Gate

Judge starts only after all required Executors and Debuggers pass, with no unresolved failures, blockers, or open circuits. Judge independently verifies deliverables, evidence, Assignment/Digest, and security boundaries. The verdict is either `PASS` or `REJECTED`.

After Judge `PASS`, Main atomically and idempotently archives `state.md` to `<workspace>/history/<task-id>/`, updates `MAM_state.md` long-term memory, resets active state, and reports completion — no routine user confirmation required. Destructive/irreversible/external/privacy-sensitive operations still require user consent.

### MAM (Cross-Task Long-Term Memory)

`<task_folder>/MAM_state.md` retains only cross-task-valid information: project overview, active principles, boundaries, key decisions, recent changes, acceptance evidence, risks, and next starting point. It is updated at most once per Task, only after Judge `PASS` and before archiving, with a mandatory readback verification. MAM must not contain full DAG copies, scheduling logs, session data, or credentials.

### Lifecycle States

```
INIT → TRIAGED → DISPATCHING → RUNNING → VERIFYING → JUDGING → ARCHIVING → COMPLETED
```

Side states: `WAITING_USER`, `RETRYING`, `RECOVERING`, `CIRCUIT_OPEN`, `FAIL`

---

## 🤝 Contributing

We welcome contributions to the Matrix Protocol! Whether it's optimizing prompt efficiency, refining the `state.md` structure, or adding new specialized roles, feel free to open a PR.

---

## 📄 License

This project is licensed under the **MIT License**.

---
**Engineered for the future of Autonomous Collaboration.**
