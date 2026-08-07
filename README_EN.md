[中文版](README.md) | **English**

---

# 🚀 OpenClaw-Matrix: Autonomous Multi-Agent Matrix Engine

> **A self-driven, state-centralized, and self-bootstrapping Multi-Agent Collaboration Protocol built on OpenClaw.**

[![OpenClaw](https://img.shields.io/badge/Powered%20by-OpenClaw-blueviolet)](https://github.com/OpenClaw/OpenClaw)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Status](https://img.shields.io/badge/Status-v5.0--Stable-orange)](https://github.com/)

## 🌟 Overview

**OpenClaw-Matrix** is a high-performance orchestration framework designed for complex AI tasks. By implementing a **"Centralized State Bus"** and **"Strict Role Decoupling,"** this system eliminates the common pitfalls of multi-agent workflows, such as context contamination, logic drifting, and infinite loops.

The core of the system is the `state.md` file, which acts as a "Shared Blackboard" for all agents. **The standout feature is its Self-Bootstrapping capability: simply provide the architecture documentation, and OpenClaw will read, parse, and configure the entire matrix by itself.**

---

## 🧠 Core Philosophy (The v5.0 Protocol)

1. **State Centralization (`state.md`)**: No direct agent-to-agent whispering. All communication flows through a single, structured Markdown file to ensure a "Single Source of Truth."
2. **Strict Sandbox Isolation**: Each agent (Researcher, Executor, Debugger) operates in its own OpenClaw Session, maintaining a "clean" context focused only on its specific duty.
3. **Summary-Driven Execution**: Agents are required to output structured summaries. The Main Controller makes routing decisions based solely on these summaries, significantly saving tokens and reducing hallucinations.
4. **Lifecycle Automation**: From new task detection (via `New_Task_Flag`) to automatic internal archiving after Judge PASS, the system manages the entire task lifecycle autonomously. User confirmation is only required for destructive/irreversible/external/privacy-sensitive actions.
5. **Runtime-Verifiable Dispatch**: Agent registration, role prompts (`.md` format), and workspace files only indicate that static configuration is complete. Main must actually invoke the assigned sub-Agents per the `Dispatch Plan`, create independent Sessions for them, and verify that results have been written back to the same `state.md`.

---

## 🏗️ Matrix Role Definitions

| Role Group | ID | Responsibility | Key Output |
| :--- | :--- | :--- | :--- |
| **Controller** | `Main` | Global routing, DAG scheduling, and User interaction. | Dispatching & State Updates |
| **Research Pool** | `Res_1~3` | Tech research, architecture design, and manual writing. | **Execution Manual** |
| **Execution Pool** | `Exe_1~3` | Code implementation, environment setup, and deployment. | **Final Artifacts** |
| **Debug Pool** | `Dbg_1~3` | Logic, boundary, and performance validation. | **Debug Feed / Logs** |
| **Arbitrator** | `Judge` | Aggregating debug reports and final evaluation. | **Evaluation / Verdict** |

---

## 🚀 Quick Start & Self-Configuration

The most powerful aspect of this project is its **Auto-Config** capability. Follow these steps to let the system build itself:
### 1. Prerequisites
- Ensure [OpenClaw](https://github.com/OpenClaw/OpenClaw) is installed and your LLM API is configured.
- Prepare the project directory:
```bash
git clone https://github.com/YourUsername/OpenClaw-Matrix.git
cd OpenClaw-Matrix
mkdir -p workspace/history
```

### 2. The "Self-Bootstrapping" Command
Start your OpenClaw session and send the following instruction to the **Main Agent**:

> *"Please read and parse the `Architecture_v5.0.md` file in the root directory. Based on this document, autonomously configure my OpenClaw Matrix sessions, initialize the `state.md` bus, inject the role-specific prompts, and enter the INIT state for a new task."*

**The system will automatically:**
- Register all Agent IDs and Session IDs.
- Inject the specialized prompts for each role from the `/prompts` folder (`.md` format).
- Generate the `state.md` template in your workspace.
- Monitor for the `New_Task_Flag` to begin execution.
- Grant Main restricted sub-Agent dispatch permissions, allowing only registered Matrix roles to be invoked.
- Perform a runtime acceptance check upon completion to confirm that sub-Agent independent Sessions, shared blackboard read/write, and structured receipts are all genuinely active.

### 3. Runtime Dispatch Acceptance

Simply generating a role roster, prompts, workspace, and config files **does not mean the Matrix is operational**. Without real sub-Agent Sessions and shared blackboard write-back, the system remains a static deployment with no runtime scheduler enabled.

After configuration, the Main Agent must complete a minimal acceptance task:

1. Create a unique `Task_ID` in `state.md` and write a clear `[Dispatch Plan]`.
2. Actually invoke at least one non-Main sub-Agent per the plan — do not have Main simulate that role's output.
3. Confirm that the role is running in an independent Session and record the actual Agent ID, Session ID, or Run ID.
4. The sub-Agent must first read `state.md` from the same absolute path, then write structured results only to its authorized section.
5. Main reads `state.md` again to confirm the `Task_ID` matches, the file modification time has updated, and the role's result genuinely exists.
6. Judge or Main issues a `PASS / FAIL` verdict based on the above evidence; any missing evidence is treated as runtime dispatch not enabled.

Acceptance requires all of the following to be true simultaneously:

- A unique `Task_ID` and recent update time for the current task exist in `state.md`.
- At least one assigned non-Main Agent has a real Session / Run.
- The `Task_ID` in the sub-Agent's receipt matches the shared blackboard.
- The sub-Agent's result has been written to the same shared blackboard, not just returned in a chat message.
- Main or Judge has recorded the final acceptance verdict.

> **Important:** Role mappings in `openclaw.json.example` and descriptor files like `matrix-config` do not automatically become a scheduler. Actual deployment still requires configuring Main's sub-Agent invocation permissions, Session visibility, and necessary Agent-to-Agent permissions per the current OpenClaw version, and completing acceptance through real invocations.

---

## 📂 Directory Structure

- `/prompts`: Contains the specialized protocol prompts for each role (`main.md`, `res.md`, `exe.md`, `dbg.md`, `judge.md`).
- `/workspace`: The active `state.md` bus and execution environment.
- `/workspace/history`: Automatic archives of completed tasks.
- `Architecture_v5.0.md`: The "Source of Truth" document used for self-configuration.
- `openclaw.json.example`: Template for session mapping.

---

## 🔄 Self-Healing Loop Logic

When an error occurs in the `Artifacts` section:
1. **Detection**: `Debugger` identifies the failure and writes a `FAIL` report to `## [3] Quality Center` (including root-cause signature and severity level).
2. **Routing**: `Main` reads the report and re-assigns the `Executor` with specific error logs, preserving already-passed nodes.
3. **Escalation**: If the same stage/root-cause signature fails **3 consecutive times**, the circuit opens (`CIRCUIT_OPEN`): dispatch stops, the task is returned to `Researcher` for a materially revised plan, or the system enters `WAITING_USER` for human intervention. The counter resets only after verifiable progress or a materially changed plan.

### Task Triage Rules

After receiving a task, Main applies hard overrides first, then additive scoring to determine the mode, and records it in `state.md`:

| Mode | Score | Use Case |
|---|---|---|
| `MAIN_ONLY` | 0–2 | Pure conversation, read-only lookup, tiny reversible operations |
| `MINIMAL` | 3–5 | Single specialist + proportionate verification |
| `FULL` | ≥6 | Full Researcher → Executor → Debugger → Judge DAG |

Hard overrides (take precedence over score): user explicitly requests Matrix/independent QA → `FULL`; safety confirmation required → `WAITING_USER`; pure conversation/read-only → `MAIN_ONLY`.

### Lifecycle States

`INIT → TRIAGED → DISPATCHING → RUNNING → VERIFYING → JUDGING → ARCHIVING → COMPLETED`

Side states: `WAITING_USER`, `RETRYING`, `RECOVERING`, `CIRCUIT_OPEN`, `FAIL`

### Node Ledger

Each dispatched node is tracked in `[1] Dispatch Plan` with: role, dependency, status (`BLOCKED_BY_DEPENDENCY / READY / RUNNING / PASS / FAIL / RETRYING / CIRCUIT_OPEN`), authorized section, Agent ID, Session Key/Run ID, retry count, timestamps, receipt, and write-back evidence. Main only dispatches `READY` nodes; downstream nodes unlock only after prerequisite nodes reach `PASS`.

### Completion Gate & Auto-Archive

The completion gate requires: deliverables satisfy acceptance criteria; all required checks pass; every required ledger node is `PASS`; no unresolved blockers or open circuits; all receipts/Task_IDs/write-back evidence validate; Judge returns `PASS`; structure and safety constraints hold.

After Judge `PASS`, Main immediately performs an internal atomic idempotent archive (named by `Task_ID`), records path and checksum evidence, resets active state, and reports completion — **no routine user confirmation required**. Destructive/irreversible operations, external publication, credential or privacy-sensitive access, and permission/security-boundary changes **still require** user consent.

---

## 🤝 Contributing

We welcome contributions to the Matrix Protocol! Whether it's optimizing Prompt efficiency, refining the `state.md` structure, or adding new specialized roles, feel free to open a PR.

---

## 📄 License

This project is licensed under the **MIT License**.

---
**Engineered for the future of Autonomous Collaboration.**
