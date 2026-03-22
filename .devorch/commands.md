# DevOrch Commands

This document defines how DevOrch interprets high-level commands and how each command must interact with the project state files.

All commands are intended to be run inside Cursor (for example via slash commands or similar integrations).

---

## `/devorch "requirement"`

**Purpose**: Start a new DevOrch project from a natural language requirement.

### Behavior

1. **Ensure `.devorch-projects` exists**
   - If the `.devorch-projects` directory does not exist at the repository root:
     - Automatically create `.devorch-projects/`.
     - Never ask the user to manually create this directory.

2. **Derive `project-name`**
   - Convert the quoted requirement into a directory-safe identifier, e.g.:
     - Lowercase.
     - Replace spaces and punctuation with hyphens.
     - Trim to a reasonable length (e.g. 3–6 keywords).
   - Example:
     - Input: `/devorch "Build a billing API with Stripe"`
     - Project name: `billing-api-with-stripe`

3. **Create orchestration directory**
   - Path: `.devorch-projects/{project-name}/`
   - Automatically create this project folder if it does not exist.
   - Never require the user to manually create the project folder.

4. **Create / overwrite project state files**
   - `BREAKDOWN.md`
     - Fill with the template from `workflow.md`:
       - `Requirement` section: embed the original requirement.
       - `Architecture` section: draft a high-level architecture (no code).
       - `Task Breakdown` section: list a sequence of concrete tasks.
         - Each task should be a small, verifiable unit of work.
         - Order tasks logically (e.g., scaffolding → core features → polish).
       - `Progress` section: set:
         - `Overall stage: planning`
         - `Last updated: {now}`
         - `Current task index: 0`
         - `Notes: Initial planning created. Awaiting approval.`
   - `DECISIONS.md`
     - Initialize with an empty decision log plus at least one initial decision if applicable (e.g., choice of framework, language, or stack).
   - `QUESTIONS.md`
     - Initialize with an empty `Open Questions` list.
     - Optionally add questions when the requirement is ambiguous.
   - `STATE.json`
     - Create or reset to:
       ```json
       {
         "stage": "planning",
         "approved": false,
         "current_task": 0,
         "project_name": "{project-name}",
         "requirement": "{requirement}",
         "last_command": "/devorch \"{requirement}\"",
         "history": [
           "Initialized project from requirement."
         ]
       }
       ```

5. **Draft architecture and task breakdown**
   - Use the requirement to propose:
     - A high-level architecture in `BREAKDOWN.md > Architecture`.
     - A linear `Task Breakdown` checklist (ideally 5–20 tasks).

6. **Pause for human approval**
   - Set:
     - `stage = "waiting_approval"`
     - `approved = false`
   - Present a summary to the human:
     - Project name.
     - Architecture summary.
     - Task Breakdown list.
   - Instruct the human to run `/devorch approve` to continue or `/devorch abort` to restart.

---

## `/devorch`

**Purpose**: Resume the current project from saved state.

### Behavior

1. **Locate current project**
   - Determine the "active" project:
     - If only one project exists in `.devorch-projects`, use that.
     - If multiple exist, use a heuristic (e.g., most recently modified) or require explicit selection (future extension).

2. **Read `STATE.json`**
   - Use `stage` to decide what to do:
     - `planning` or `waiting_approval`:
       - Re-present the current architecture, tasks, and questions.
       - Prompt for `/devorch approve` or `/devorch abort`.
     - `implementing`:
       - Summarize the current task (from `BREAKDOWN.md`).
       - Propose next steps and wait for `/devorch implement` to actually modify code.
     - `review`:
       - Remind the user that a task implementation is awaiting approval.
       - Prompt for `/devorch approve` or `/devorch abort`.
     - `completed`:
       - Summarize the finished project and remind the user to handle Git commit/push as desired.
     - `aborted`:
       - Inform the user that orchestration was aborted and suggest `/devorch "requirement"` to start over.

3. **Never mutate state on a plain `/devorch`**
   - This command is for **inspection and guidance only**.

---

## `/devorch approve`

**Purpose**: Approve the current checkpoint and advance the workflow.

### Behavior

1. **Read `STATE.json`** and inspect `stage`.

2. **If `stage = "waiting_approval"` (after planning)**
   - Interpretation: Approve the architecture and task breakdown.
   - Actions:
     - Set:
       - `approved = true`
       - `stage = "implementing"`
     - Append a history entry:
       - `"Planning approved by human."`
     - Optionally add a note in `BREAKDOWN.md > Progress` indicating planning approval and timestamp.
   - Next step:
     - Instruct user to run `/devorch implement` to start implementing the first task.

3. **If `stage = "review"` (after a task implementation)**
   - Interpretation: Approve the implemented task.
   - Actions:
     - Set `approved = true`.
     - Mark the current task checkbox as completed in `BREAKDOWN.md > Task Breakdown`.
     - Update `BREAKDOWN.md > Progress`:
       - `Current task index` to the next task.
       - Add a short note about approval and timestamp.
     - In `STATE.json`:
       - Increment `current_task` by 1.
       - Determine if more tasks remain:
         - If yes: `stage = "implementing"`.
         - If no: `stage = "completed"`.
       - Append history entry:
         - `"Task {index} approved by human."`
   - Next step:
     - If more tasks: suggest `/devorch implement`.
     - If none: summarize completion and suggest Git flows.

4. **Other stages**
   - If approval is called in unsupported stages (e.g., `completed`, `aborted`), respond with a non-mutating explanation.

---

## `/devorch --plan`

**Purpose**: Planning mode only – generate or regenerate architecture and task breakdown without changing any project code.

### Behavior

1. **If no active project exists**
   - Require a requirement string (prefer `/devorch "requirement"`).

2. **If an active project exists**
   - Read `STATE.json` and `BREAKDOWN.md`.
   - Recompute or refine:
     - `Architecture` section.
     - `Task Breakdown` section.
   - Update `Progress`:
     - Keep or reset `stage` to:
       - `planning` or `waiting_approval` as appropriate.
   - Never touch source files in the codebase.
   - Set `approved = false`, `stage = "waiting_approval"` and add a history entry like:
     - `"Plan updated in planning-only mode."`

3. **Always pause for approval**
   - After recomputing, the next step is `/devorch approve` or `/devorch abort`.

---

## `/devorch implement`

**Purpose**: Implement the **next pending task** from `BREAKDOWN.md`.

### Behavior

1. **Preconditions**
   - `STATE.json.stage` must be `"implementing"`.
   - `approved` should be `true` from the last checkpoint (planning or previous task).

2. **Select next task**
   - Use `STATE.json.current_task` as the index into the `Task Breakdown` list.
   - Identify the first **unchecked** task.
   - If no remaining tasks:
     - Set `stage = "completed"` and exit with a completion summary.

3. **Explain planned changes**
   - Before editing any files:
     - Present:
       - The selected task text.
       - Relevant decisions from `DECISIONS.md`.
       - Any open questions from `QUESTIONS.md` that may impact the task.
     - Describe:
       - Files expected to be added or changed.
       - Rough outline of the implementation.

4. **Implement exactly one task**
   - Modify project code in the repository root according to the planned task.
   - Do not modify other unrelated tasks, even if they appear trivial.
   - Keep implementation consistent with `DECISIONS.md`.

5. **Update state and pause for review**
   - Update `BREAKDOWN.md > Progress`:
     - Record that task `{index}` has been implemented and is awaiting review.
   - Update `STATE.json`:
     - `stage = "review"`
     - `approved = false`
     - `last_command = "/devorch implement"`
     - Append a history entry:
       - `"Implemented task {index}, awaiting review."`
   - Do **not** mark the task checkbox as completed yet.
   - Instruct the human to run `/devorch approve` to accept the implementation or `/devorch abort` to reset.

---

## `/devorch abort`

**Purpose**: Abort the current orchestration and restart planning.

### Behavior

1. **Read `STATE.json`**.

2. **Mark orchestration aborted**
   - Set:
     - `stage = "aborted"`
     - `approved = false`
     - Append history entry:
       - `"Orchestration aborted by human."`

3. **Handle state files**
   - Option A (recommended default): **Preserve** existing `BREAKDOWN.md`, `DECISIONS.md`, and `QUESTIONS.md` as a historical artifact.
     - Optionally append notes at the top of `BREAKDOWN.md`:
       - Indicate that the project has been aborted, with timestamp.
   - Option B (explicit future extension): Delete and reinitialize state.

4. **Restarting**
   - To start a fresh orchestration, the human should run:
     - `/devorch "requirement"` again (can be the same or a new requirement).

---

## Behavioral Guarantees

All commands must respect the rules defined in `.devorch/system_rules.md`, including:

- Always keep `BREAKDOWN.md` and `STATE.json` in sync.
- Implement at most one task per `/devorch implement`.
- Never push commits or run `git push` without explicit human instruction.
- Never skip approval checkpoints (`waiting_approval` or `review` stages).

