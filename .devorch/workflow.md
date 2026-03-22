# DevOrch Workflow

DevOrch orchestrates a human‑in‑the‑loop development workflow from requirement to implementation using structured markdown state files.

This document defines the **canonical workflow** and how each command should interpret and update project state.

---

## 1. Core Concepts

- **Requirement**: A natural language description of what should be built.
- **Project**: A single DevOrch orchestration instance created from a requirement.
- **State Directory**: `.devorch-projects/{project-name}/`
- **Source of Truth**:
  - `REQUIREMENT_ANALYSIS.md` – single source of truth: requirement summary, architecture, task breakdown, and progress.
  - `EVALUATED_OUTCOME.md` – evaluated outcomes: architecture decisions, framework choices, and major design trade-offs.
  - `OPEN_QUESTIONS.md` – open questions that block or affect work until answered.
  - `STATE.json` – machine-readable orchestration state.

---

## 2. Project State Files

Each project created under `.devorch-projects/{project-name}/` MUST contain:

### 2.1 REQUIREMENT_ANALYSIS.md

**Purpose**: Single source of truth for the project — requirement analysis, architecture, tasks, and progress.

**Required sections (in order):**

```markdown
# REQUIREMENT_ANALYSIS – {project-name}

## Requirement
- Original requirement:
  - "{requirement}"

## Architecture
- High-level overview:
  - [ ] Define main components
  - [ ] Define data flow
  - [ ] Define external dependencies

## Task Breakdown

- [ ] Task 1 – Short description
  - Summary:
  - Acceptance criteria:
- [ ] Task 2 – Short description
  - Summary:
  - Acceptance criteria:

## Progress

- Overall stage: `{planning|implementation|review|completed|aborted}`
- Last updated: {ISO timestamp}
- Current task index: {integer}
- Notes:
  - Initialised.
```

**Rules:**
- Always update `Task Breakdown` checkboxes and `Progress` when any task starts, is completed, or is re-scoped.
- Treat `REQUIREMENT_ANALYSIS.md` as the **authoritative list of tasks**; implementation must follow this list in order.

### 2.2 EVALUATED_OUTCOME.md

**Purpose**: Record evaluated outcomes — decisions and rationale that affect architecture, technology, and scope.

**Template:**

```markdown
# EVALUATED_OUTCOME – {project-name}

## Summary
- This file tracks architectural and other major decisions made during the project.

## Decision Log

### D-001 – {short title}
- Date: {ISO timestamp}
- Context:
  - Why was this decision needed?
- Decision:
  - What did we decide?
- Alternatives considered:
  - Option A – Pros / Cons
  - Option B – Pros / Cons
- Consequences:
  - Immediate:
  - Long term:
```

### 2.3 OPEN_QUESTIONS.md

**Purpose**: Track questions that are not yet resolved and require human clarification or decisions.

**Template:**

```markdown
# OPEN_QUESTIONS – {project-name}

## Open Questions

- Q-001 – {question title}
  - Asked: {ISO timestamp}
  - Details:
  - Status: open|answered|discarded
  - Answer (if any):

## Notes
- Use this file to pause progress when clarification is needed.
```

### 2.4 STATE.json

**Purpose**: Machine-readable orchestration state.

**Baseline shape:**

```json
{
  "stage": "planning",
  "approved": false,
  "current_task": 0,
  "project_name": "",
  "requirement": "",
  "last_command": "",
  "history": []
}
```

**Field semantics:**
- `stage`: One of `"planning" | "waiting_approval" | "implementing" | "review" | "completed" | "aborted"`.
- `approved`: Whether the current checkpoint is approved.
- `current_task`: Zero-based index into the `Task Breakdown` list in `REQUIREMENT_ANALYSIS.md`.
- `project_name`: Directory-safe project identifier.
- `requirement`: Original requirement string.
- `last_command`: The last DevOrch command that mutated state.
- `history`: Optional array of short log entries summarizing transitions.

---

## 3. Lifecycle & Stages

DevOrch follows these stages:

1. **planning**
   - Triggered by `/devorch "requirement"` or `/devorch --plan`.
   - Create project state directory and all state files.
   - Draft architecture and full task breakdown in `REQUIREMENT_ANALYSIS.md`.
   - Record initial decisions in `EVALUATED_OUTCOME.md` where applicable.
   - Set `stage = "waiting_approval"` and `approved = false`.
   - **Always pause for human approval**.

2. **waiting_approval**
   - Await `/devorch approve` or `/devorch abort`.
   - No code changes are allowed in this stage.

3. **implementing**
   - Entered after approval when executing `/devorch implement`.
   - Select the **next unchecked task** in `REQUIREMENT_ANALYSIS.md`.
   - Explain the planned change referencing the task and decisions.
   - Implement **only that single task** in the codebase.
   - Update `REQUIREMENT_ANALYSIS.md` progress and `STATE.json.current_task`.
   - Move to `stage = "review"` and `approved = false`.

4. **review**
   - Await `/devorch approve` to mark the current task as accepted or `/devorch abort` to reset.
   - After approval:
     - Mark the task checkbox as completed in `REQUIREMENT_ANALYSIS.md`.
     - If more tasks are pending, set `stage = "implementing"` (ready for next `/devorch implement`).
     - If no tasks remain, move to `stage = "completed"`.

5. **completed**
   - All tasks in `Task Breakdown` are completed and approved.
   - The human may perform or instruct Git commit / push.
   - DevOrch must **never push** without explicit instruction.

6. **aborted**
   - Entered via `/devorch abort`.
   - Orchestration state may be deleted or reset as per command definition.

---

## 4. Checkpoints & Human-in-the-Loop

- Every major transition requires an explicit checkpoint.
- The agent:
  - **Must never change `stage` from a waiting state without a matching command.**
  - **Must never skip the `review` stage for implemented tasks.**
  - **Must not implement multiple tasks per `/devorch implement` invocation.**
- Approved checkpoints are recorded in:
  - `STATE.json` (`approved: true` after approval).
  - Optional note in `REQUIREMENT_ANALYSIS.md` under `Progress`.

---

## 5. Mapping Commands to Workflow

See `.devorch/commands.md` for detailed command semantics.

At a high level:

- `/devorch "requirement"` → Initialize project, enter `planning`, then `waiting_approval`.
- `/devorch` → Resume based on `STATE.json.stage`.
- `/devorch approve` → Approve current checkpoint, advance to next stage.
- `/devorch --plan` → Only (re)generate architecture + tasks; no code.
- `/devorch implement` → Implement next task, then move to `review`.
- `/devorch abort` → Abort and reset orchestration for the project.

---

## 6. Code Location Rules

- All **code** is generated in the **repository root**, as sibling directories/files to `.devorch` and `.devorch-projects`.
- `.devorch-projects/` is reserved only for orchestration metadata and is always gitignored.

