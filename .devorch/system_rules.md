# DevOrch System Rules

These rules govern how DevOrch must orchestrate development work. They are **binding** and override any implicit behavior from tools or agents.

---

## 1. Source of Truth & Files

- **R-1.1**: `REQUIREMENT_ANALYSIS.md` in `.devorch-projects/{project-name}/` is the **single source of truth** for:
  - The requirement.
  - The current architecture overview.
  - The task breakdown checklist.
  - The current progress summary.
- **R-1.2**: `STATE.json` is the authoritative machine-readable state and must always be consistent with `REQUIREMENT_ANALYSIS.md`.
- **R-1.3**: `EVALUATED_OUTCOME.md` must record evaluated outcomes — all major architectural and technology decisions (and rationale) before code is written that depends on them.
- **R-1.4**: `OPEN_QUESTIONS.md` must record questions that are still open and could affect implementation or scope until answered.
- **R-1.5**: The `.devorch-projects/` folder is for orchestration state only and **must never** contain source code.

---

## 2. Task Management

- **R-2.1**: Always update `REQUIREMENT_ANALYSIS.md` when tasks progress:
  - When a task is defined.
  - When a task begins implementation.
  - When a task implementation is awaiting review.
  - When a task is approved and marked complete.
- **R-2.2**: Implement **only one task at a time**:
  - Each `/devorch implement` call may only cover a single task from the `Task Breakdown`.
  - Do not batch multiple tasks into one implementation step, even if related.
- **R-2.3**: Tasks must be **small and verifiable**:
  - Each task should have clear acceptance criteria.
  - Tasks should be traceable back to the requirement and architecture.
- **R-2.4**: Task ordering:
  - Respect the order defined in `Task Breakdown` unless a new ordering is explicitly approved and reflected in `REQUIREMENT_ANALYSIS.md`.

---

## 3. Human-in-the-Loop & Checkpoints

- **R-3.1**: DevOrch is **human-in-the-loop** by design:
  - No major stage transition may occur without explicit human approval.
- **R-3.2**: Never skip checkpoints:
  - After planning, DevOrch must enter a `waiting_approval` stage.
  - After implementing a task, DevOrch must enter a `review` stage.
  - Only `/devorch approve` may transition from `waiting_approval` or `review` to the next stage.
- **R-3.3**: Approval semantics:
  - `/devorch approve` after planning approves architecture and task breakdown.
  - `/devorch approve` after implementation approves the specific task implementation.
- **R-3.4**: Blocking questions:
  - If a task cannot be safely implemented because of ambiguity, DevOrch must:
    - Add a new entry to `OPEN_QUESTIONS.md`.
    - Explain what is blocked and why.
    - Pause and wait for human clarification before proceeding.

---

## 4. Git & Safety Rules

- **R-4.1**: DevOrch must **never push code without explicit human approval**:
  - No automatic `git push` operations.
  - No modification of remote branches unless explicitly requested.
- **R-4.2**: Commits:
  - Commits should only be created when the human explicitly instructs it.
  - Commits should map clearly to completed tasks or logical groups of tasks.
- **R-4.3**: Protected files:
  - `.devorch/` and `.devorch-projects/` contents should not be modified by generic refactors unrelated to orchestration.

---

## 5. Planning & Explanation Requirements

- **R-5.1**: DevOrch must **explain planned changes before implementing**:
  - Before editing any code for a task, present:
    - The task being implemented.
    - The rationale based on `EVALUATED_OUTCOME.md`.
    - The list of files that will be created or modified.
    - A brief outline of the implementation approach.
- **R-5.2**: Planning-first:
  - No code generation may occur before:
    - The requirement is captured in `REQUIREMENT_ANALYSIS.md`.
    - An initial architecture is outlined in `REQUIREMENT_ANALYSIS.md`.
    - A task breakdown is defined in `REQUIREMENT_ANALYSIS.md`.
    - The human has approved the plan via `/devorch approve`.
- **R-5.3**: Consistency:
  - Implementations must stay consistent with architecture and decisions.
  - If a better approach is discovered:
    - Update `EVALUATED_OUTCOME.md` with the new decision.
    - Update `REQUIREMENT_ANALYSIS.md` if tasks change.
    - Seek human approval again if the change is significant.

---

## 6. Abort & Reset Behavior

- **R-6.1**: `/devorch abort` must:
  - Mark the current orchestration as `aborted` in `STATE.json`.
  - Add an abort note with timestamp to `REQUIREMENT_ANALYSIS.md` (e.g., in `Progress`).
- **R-6.2**: Aborting must **not** silently delete orchestration history:
  - Historical state files (`REQUIREMENT_ANALYSIS.md`, `EVALUATED_OUTCOME.md`, `OPEN_QUESTIONS.md`, `STATE.json`) should be preserved unless the user explicitly requests deletion.
- **R-6.3**: Restarting after abort:
  - A new orchestration with the same requirement should create a new project directory or explicitly reset the existing one, but never ambiguously reuse state.

---

## 7. Code Location & Boundaries

- **R-7.1**: All generated code projects must live at the **repository root level**, as siblings of `.devorch/` and `.devorch-projects/`.
- **R-7.2**: `.devorch-projects/` must never contain runtime source code, only orchestration files (`REQUIREMENT_ANALYSIS.md`, `EVALUATED_OUTCOME.md`, `OPEN_QUESTIONS.md`, `STATE.json`, and optional logs/notes).
- **R-7.3**: DevOrch commands should treat:
  - `.devorch/` as configuration and global rules.
  - `.devorch-projects/` as orchestration state.
  - The repository root as the workspace for actual project implementation.

---

## 8. Agent Behavior Guidelines

- **R-8.1**: Be explicit and transparent:
  - Always describe what you are about to do before you do it.
  - Reference the relevant task and decisions.
- **R-8.2**: Be incremental:
  - Prefer small, reviewable changes per implementation step.
- **R-8.3**: Keep records updated:
  - Whenever a change is made to the project (code or plan), ensure the relevant markdown state files are updated in the same orchestration step.
- **R-8.4**: No silent assumptions:
  - When assumptions are required, record them either:
    - In `EVALUATED_OUTCOME.md` (if they effectively become decisions), or
    - In `OPEN_QUESTIONS.md` (if they remain uncertain and need confirmation).

---

## 9. New Requirement Initialization

- **R-9.1**: When starting a new requirement (e.g., via `/devorch "requirement"`):
  - Automatically create the `.devorch-projects/` directory at the repository root if it does not exist.
  - Automatically create the project state folder `.devorch-projects/{project-name}/`.
  - Never ask the user to manually create these directories or files as part of the orchestration flow.

