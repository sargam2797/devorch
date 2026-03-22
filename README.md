## DevOrch – Markdown-Driven AI Dev Orchestration

*AI developer orchestrator — structured workflows with human-in-the-loop checkpoints.*

DevOrch is a small orchestration framework designed to run inside Cursor. It lets AI agents and humans collaborate on software projects using **markdown state files** as the source of truth, enforcing a **human-in-the-loop, checkpointed workflow** from requirement to implementation.

DevOrch never skips approval checkpoints and never pushes code without explicit instruction.

---

## Why we built DevOrch

DevOrch is an experiment to solve a problem that goes **beyond generating code**: following a **structured development workflow**. Typical AI code generation skips planning, misses context, and jumps straight into implementation.

DevOrch introduces a simple flow:

**Requirement → Plan → Decisions → Tasks → Approval → Implementation**

The goal is to make AI behave less like a one-shot code generator and more like a developer who follows a clear, repeatable process—with humans in the loop at the right moments.

---

## What DevOrch Does

- **Captures requirements** and turns them into a structured project.
- **Generates planning artifacts** under `.devorch-projects/{project-name}/`:
  - `BREAKDOWN.md` – requirement, architecture, task checklist, and progress.
  - `DECISIONS.md` – important architectural and technology decisions.
  - `QUESTIONS.md` – open questions that need clarification.
  - `STATE.json` – machine-readable orchestration state (stage, approvals, current task, etc.).
- **Enforces a staged workflow**:
  - Requirement → Planning → Human approval → Stepwise implementation → Human approval → Completion.
- **Keeps code and orchestration separate**:
  - Code lives in the repo root (e.g., `billing-api/`).
  - Orchestration state lives in `.devorch-projects/` (gitignored).

---

## High-Level Workflow

### 1. Start a New Project

- **Command**: `/devorch "your requirement here"`
- **What happens**:
  - A new project folder is created in `.devorch-projects/{project-name}/`.
  - DevOrch drafts:
    - An architecture overview.
    - A task breakdown checklist.
  - `BREAKDOWN.md`, `DECISIONS.md`, `QUESTIONS.md`, and `STATE.json` are initialized.
  - The project enters the `waiting_approval` stage.

- **Your role**:
  - Open the generated `BREAKDOWN.md` and `DECISIONS.md`.
  - Review the requirement, architecture, and tasks.

### 2. Approve the Plan

- **Command**: `/devorch approve`
- **What happens**:
  - The planning checkpoint is approved.
  - `STATE.json` moves from `waiting_approval` to `implementing`.
  - DevOrch is now allowed to start implementing tasks.

### 3. Implement Tasks Step-by-Step

- **Command**: `/devorch implement`
- **What happens**:
  - DevOrch selects the **next unchecked task** in `BREAKDOWN.md`.
  - It **explains the planned changes first**:
    - Which task is being implemented.
    - Which files in the repo root will be added or modified.
    - How the implementation aligns with recorded decisions.
  - It then implements **only that single task** in the codebase.
  - Progress is updated in `BREAKDOWN.md` and `STATE.json`.
  - The project enters the `review` stage, awaiting your approval.

- **Your role**:
  - Review the code changes and the updated markdown state.

### 4. Approve or Abort Each Implementation

- **Approve implementation**:
  - **Command**: `/devorch approve`
  - **Effect**:
    - Marks the current task as completed in `BREAKDOWN.md`.
    - Advances `STATE.json.current_task` to the next task.
    - If tasks remain, stage goes back to `implementing`.
    - If all tasks are done, stage moves to `completed`.

- **Abort orchestration**:
  - **Command**: `/devorch abort`
  - **Effect**:
    - Marks the project as `aborted` in `STATE.json`.
    - Adds an abort note to `BREAKDOWN.md`.
    - Leaves all state files in place as history.

### 5. Resume at Any Time

- **Command**: `/devorch`
- **What happens**:
  - Reads `STATE.json` for the active project.
  - Summarizes:
    - Current stage (`planning`, `waiting_approval`, `implementing`, `review`, `completed`, or `aborted`).
    - Current task and overall progress.
  - Suggests the appropriate next command (e.g., `/devorch approve` or `/devorch implement`).
  - Does **not** mutate any state or code.

---

## Commands Cheat Sheet

- **Start new project**: `/devorch "requirement"`
- **Show current status**: `/devorch`
- **Approve current checkpoint**: `/devorch approve`
- **Plan only (no code)**: `/devorch --plan`
- **Implement next task**: `/devorch implement`
- **Abort current orchestration**: `/devorch abort`

All command semantics and low-level rules are defined in:

- `./.devorch/workflow.md`
- `./.devorch/commands.md`
- `./.devorch/system_rules.md`

---

## Using DevOrch

This repository includes a **project Cursor skill** at `.cursor/skills/devorch.md` that teaches the agent to respond to the `/devorch` command family.

### Run DevOrch inside your repo

In Cursor, run one of:

- `/devorch "requirement"` to start a new orchestrated workflow.
- `/devorch` to resume the last workflow from saved state.
- `/devorch approve` to approve the current checkpoint (planning or review).
- `/devorch implement` to implement the next planned task (one task only).
- `/devorch --plan` to generate planning documents only (no code changes).
- `/devorch abort` to abort the current workflow.

### What gets created

DevOrch automatically creates orchestration state under:

- `.devorch-projects/{project-name}/`
  - `BREAKDOWN.md` (source of truth: requirement, architecture, tasks, progress)
  - `DECISIONS.md` (key architecture decisions)
  - `QUESTIONS.md` (clarifications needed)
  - `STATE.json` (workflow stage and task index)

The actual project code is created/edited in the **repository root** (not inside `.devorch-projects/`).
