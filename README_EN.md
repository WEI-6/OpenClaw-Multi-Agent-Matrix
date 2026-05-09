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
4. **Lifecycle Automation**: From new task detection (via `New_Task_Flag`) to user-confirmed archiving, the system manages the entire task lifecycle autonomously.

---

## 🏗️ Matrix Role Definitions

| Role Group | ID | Responsibility | Key Output |
| :--- | :--- | :--- | :--- |
| **Controller** | `Main` | Global routing, DAG scheduling, and User interaction. | Dispatching & State Updates |
| **Research Pool** | `Res_1~3` | Tech research, architecture design, and manual writing. | **Execution Manual** |
| **Execution Pool** | `Exe_1~3` | Code implementation, environment setup, and deployment. | **Final Artifacts** |
| **Debug Pool** | `Dbg_1~3` | Logic, boundary, and performance validation. | **Debug Feed / Logs** |
| **Arbitrator** | `Judge` | Aggregating debug reports and final evaluation. | **Evaluation / Approval** |

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
- Inject the specialized Prompts for each role from the `/prompts` folder.
- Generate the `state.md` template in your workspace.
- Monitor for the `New_Task_Flag` to begin execution.

---

## 📂 Directory Structure

- `/prompts`: Contains the specialized protocols (RES, EXE, DBG, JUDGE) in `.txt` format.
- `/workspace`: The active `state.md` bus and execution environment.
- `/workspace/history`: Automatic archives of completed tasks.
- `Architecture_v5.0.md`: The "Source of Truth" document used for self-configuration.
- `openclaw.json.example`: Template for session mapping.

---

## 🔄 Self-Healing Loop Logic

When an error occurs in the `Artifacts` section:
1. **Detection**: `Debugger` identifies the failure and writes a `FAIL` report to the Quality Center.
2. **Routing**: `Main` reads the report and re-assigns the `Executor` with specific error logs.
3. **Escalation**: If the loop repeats 3 times without success, `Main` triggers a "Plan Reset," sending the task back to the `Researcher` to redesign the approach.

---

## 🤝 Contributing

We welcome contributions to the Matrix Protocol! Whether it's optimizing Prompt efficiency, refining the `state.md` structure, or adding new specialized roles, feel free to open a PR.

---

## 📄 License

This project is licensed under the **MIT License**.

---
**Engineered for the future of Autonomous Collaboration.**
