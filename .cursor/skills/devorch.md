---
name: devorch
description: Orchestrates a human-in-the-loop developer workflow using the /devorch command and markdown state files (REQUIREMENT_ANALYSIS.md, EVALUATED_OUTCOME.md, OPEN_QUESTIONS.md, STATE.json). Use when the user runs /devorch, asks to start a project from a requirement, or wants step-by-step planning/implementation with approval checkpoints.
---

## DevOrch (project skill)

This skill registers the command family:

- `/devorch "requirement"`
- `/devorch`
- `/devorch approve`
- `/devorch implement`
- `/devorch --plan`
- `/devorch abort`

DevOrch uses **markdown state** as the source of truth and enforces **approval checkpoints**.

---

## State model (must follow)

All orchestration state lives under:

- `.devorch-projects/{project-name}/`

The directory contains:

- `REQUIREMENT_ANALYSIS.md` (single source of truth: requirement analysis, architecture, tasks, progress)
- `EVALUATED_OUTCOME.md` (evaluated outcomes: architecture, framework choices, major decisions)
- `OPEN_QUESTIONS.md` (questions open until answered)
- `STATE.json` (machine-readable state)

**Code must be created/edited in the repository root** (not inside `.devorch-projects/`).

---

## Workflow rules (hard requirements)

- Do **not** implement code immediately after `/devorch "requirement"`.
- Always generate planning documents first, then present the plan.
- Always wait for explicit approval (`/devorch approve`) before implementing.
- Implement **one task at a time**.
- After implementing a single task, pause and ask the user to review. Do not continue until approved.
- Keep `REQUIREMENT_ANALYSIS.md` updated whenever task status changes.
- If ambiguity blocks implementation, add an entry to `OPEN_QUESTIONS.md` and pause.

---

## Command behavior

### `/devorch "requirement"` (start new workflow)

1. **Analyze the requirement**
   - Extract goals, constraints, non-goals, and acceptance criteria.
   - Identify ambiguities → add to `OPEN_QUESTIONS.md`.

2. **Determine `project-name`**
   - Prefer a directory-safe slug derived from the requirement.
   - Guidelines:
     - lowercase
     - hyphen-separated
     - short (3–6 keywords)

3. **Create state directories automatically**
   - If `.devorch-projects/` does not exist, create it.
   - Create `.devorch-projects/{project-name}/`.
   - Never ask the user to manually create these directories.

4. **Generate state files**
   - Create/overwrite the following files in `.devorch-projects/{project-name}/`:
     - `REQUIREMENT_ANALYSIS.md`
     - `EVALUATED_OUTCOME.md`
     - `OPEN_QUESTIONS.md`
     - `STATE.json`

5. **Populate the files**

**`REQUIREMENT_ANALYSIS.md`** must include:
- Requirement summary (include original quoted requirement)
- Architecture overview (high-level components + data flow)
- Task breakdown checklist (small, verifiable tasks; each with acceptance criteria)
- Progress (stage + current task index + notes)

**`EVALUATED_OUTCOME.md`** must include:
- Initial evaluated outcomes (framework/language choices if applicable)
- Design patterns to use (when relevant)
- A decision log format with dates and consequences (same role as prior “decisions” file)

**`OPEN_QUESTIONS.md`** must include:
- A list of clarification questions (if any)
- Status per question (open/answered/discarded)

**`STATE.json`** must track:
- workflow stage: `planning | awaiting_approval | implementing | review | completed | aborted`
- whether the current checkpoint is approved
- current task index
- project name and original requirement

Suggested initial `STATE.json` shape:

```json
{
  "stage": "awaiting_approval",
  "approved": false,
  "current_task": 0,
  "project_name": "",
  "requirement": ""
}
```

6. **Present the plan and pause**
   - Summarize the architecture and list the tasks.
   - Ask the user to run `/devorch approve` to proceed.

---

### `/devorch` (resume last workflow)

- Locate the most relevant project in `.devorch-projects/` (prefer the most recently modified).
- Read `STATE.json` and `REQUIREMENT_ANALYSIS.md`.
- Summarize current stage, current task, and next expected command.
- Do not change any state or code.

---

### `/devorch --plan` (planning-only)

- (Re)generate architecture + task breakdown in `REQUIREMENT_ANALYSIS.md`.
- Update `EVALUATED_OUTCOME.md` / `OPEN_QUESTIONS.md` as needed.
- Do not write or modify project code.
- Set state to `awaiting_approval` and pause for `/devorch approve`.

---

### `/devorch approve` (approve checkpoint)

- If stage is `awaiting_approval`:
  - Approves planning; transition to `implementing`.
- If stage is `review`:
  - Approves the last implemented task:
    - Mark it complete in `REQUIREMENT_ANALYSIS.md`.
    - Advance `current_task`.
    - Transition to `implementing` (or `completed` if no tasks remain).
- Otherwise:
  - Explain what stage you’re in and what approval means there.

---

### `/devorch implement` (implement next task)

Preconditions:
- Stage must be `implementing`.
- Planning must have been approved.

Steps:
1. Select the next unchecked task in `REQUIREMENT_ANALYSIS.md`.
2. Explain planned changes (files to touch + approach) before editing any code.
3. Implement exactly that one task in the repository root.
4. Update `REQUIREMENT_ANALYSIS.md` progress and `STATE.json`.
5. Set stage to `review`, set `approved=false`, and pause for user review.

---

### `/devorch abort` (reset workflow)

- Mark the workflow as `aborted` in `STATE.json`.
- Add an abort note in `REQUIREMENT_ANALYSIS.md` progress.
- Preserve state files unless the user explicitly asks to delete them.

