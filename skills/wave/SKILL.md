---
name: wave
description: "Host-side iterative wave orchestration policy for code-dispatcher execution. Use only when the user explicitly invokes /wave, mentions wave, or requests iterative wave-based dispatch in successive rounds. Apply only after the host agent already understands the task and can split the remaining work into roughly balanced, independent tasks with mutually exclusive file scopes. Dispatch each wave via code-dispatcher --parallel, review the results, then plan the next wave. Do not use for ambiguous tasks, overlapping same-wave file scopes, or work that is better handled by a single DAG run."
---

# /wave - Iterative Wave Orchestration

## Purpose

Use `/wave` when the host agent already understands the task well enough to orchestrate execution in rounds:

1. split the remaining work into a frontier of independent tasks
2. dispatch that frontier in parallel via `code-dispatcher`
3. review results
4. plan the next frontier from what actually happened

`code-dispatcher` is the execution layer. `/wave` is the host-side orchestration policy.

## Responsibility Split

- `code-dispatcher` skill explains how to invoke the CLI, use `--parallel`, use per-task backends, and use `session_id` resume.
- `/wave` explains when tasks may share a wave, what constraints must hold, how the host reviews a wave, and when to open the next wave.

Do not treat `/wave` as a separate runtime. It is a workflow contract for the current host agent.

## Core Model

ASCII model:

```text
wave 1: [task A] [task B] [task C]
           |        |        |
           +--------+--------+
                    review
                      |
wave 2:        [task D] [task E]
                  |        |
                  +--------+
                    review
                      |
wave 3:           [task F]
```

Interpretation:

- Same wave = tasks are independent before dispatch.
- Next wave = tasks depend on the code, decisions, or findings produced by the previous wave.
- The host agent owns decomposition, routing, review, and replanning.
- Worker backends own execution inside the file scope they were assigned.

## Hard Rules

1. A task belongs in the current wave only if all prerequisites are already satisfied before dispatch.
2. Tasks in the same wave MUST have mutually exclusive file scopes.
3. Mutual exclusion is mandatory at the file-scope level:
   - no overlapping files
   - no parent/child directory overlap
   - no shared ownership of a generated file unless moved to a later merge wave
4. If mutual exclusion cannot be guaranteed in one working tree, use separate worktrees/clones or move one task to the next wave.
5. Every task MUST declare:
   - `id`
   - `backend`
   - `workdir`
   - `file scope`
   - `objective`
   - `validation command`
   - `success criteria`
6. Analysis/planning tasks and implementation tasks MUST NOT be mixed in the same wave.
7. The host agent MUST NOT directly implement production code while a wave is running. Keep the host focused on orchestration, review, and next-wave planning.
8. After every wave, the host agent MUST review outputs before opening the next wave.
9. Prefer per-task backend routing. The command-line `--backend` is only the global fallback.
10. If a follow-up task benefits from worker memory, use `session_id` resume. If not, start a new task.
11. Retry a failed task at most 2 rounds. After that, re-split the plan or escalate the issue to the user.

## Wave Admission Checklist

Before dispatching a wave, the host agent MUST verify all of the following:

1. Each task is implementation-ready.
2. Each task has a precise file scope.
3. Same-wave file scopes are mutually exclusive.
4. Each task has an explicit validation command.
5. Each task has a backend chosen for the task type.
6. Tasks in this wave are roughly comparable in expected effort.
7. Nothing in this wave depends on output from another task in the same wave.

If any check fails, do not dispatch the wave yet.

## Wave Artifact

Before each dispatch, the host agent SHOULD write out the wave plan in this structure:

```text
Wave N Goal:
- what this wave is meant to unlock

Wave N Tasks:
- ID:
  Backend:
  File Scope:
  Objective:
  Validation:
  Success:

Wave N Safety Check:
- file scopes mutually exclusive: yes/no
- shared worktree safe: yes/no
- session resume needed: yes/no
```

This artifact can be shown to the user, kept in scratch notes, or embedded directly into the dispatched prompts.

## Backend Routing

Choose the backend per task, not per workflow slogan.

Recommended defaults:

- `codex`: complex logic, refactoring, bug fixing, deep code changes
- `claude`: smaller fixes, cleanup, docs, prompt-heavy transformations
- `gemini`: UI, UX, styling, component polish

If all tasks truly need the same backend, reusing one backend for the whole wave is acceptable. Otherwise, assign `backend` per task explicitly.

## Dispatch Template

Use `code-dispatcher --parallel` as the execution engine.

```bash
code-dispatcher --parallel --backend codex <<'EOF'
---TASK---
id: w1-auth-logic
backend: codex
workdir: .
---CONTENT---
Wave: 1
Task ID: w1-auth-logic
Objective: implement auth token validation flow
File Scope: code-dispatcher/auth.go, code-dispatcher/auth_test.go
Constraints:
- Modify only files in the declared file scope.
- Do not edit files owned by other tasks in this wave.
Validation: cd code-dispatcher && go test ./... -run TestAuth
Success:
- token validation implemented
- tests pass
- no edits outside declared file scope

---TASK---
id: w1-docs
backend: claude
workdir: .
---CONTENT---
Wave: 1
Task ID: w1-docs
Objective: update auth usage documentation
File Scope: docs/runtime-config.md
Constraints:
- Modify only files in the declared file scope.
- Do not edit files owned by other tasks in this wave.
Validation: test -f docs/runtime-config.md
Success:
- auth documentation updated
- no edits outside declared file scope
EOF
```

Notes:

- The command-line `--backend codex` above is only the fallback backend.
- Use per-task `backend` whenever tasks differ by type.
- `workdir: .` is acceptable only if same-wave file scopes are mutually exclusive.

## Resume Across Waves

If a task in wave `N+1` is a continuation of a worker result from wave `N`, prefer `session_id` resume instead of starting from scratch.

Example:

```bash
code-dispatcher --parallel --backend codex <<'EOF'
---TASK---
id: w2-auth-followup
backend: codex
session_id: <dispatcher-session-id-from-wave-1>
---CONTENT---
Wave: 2
Task ID: w2-auth-followup
Objective: apply the requested follow-up changes after wave 1 review
File Scope: code-dispatcher/auth.go, code-dispatcher/auth_test.go
Validation: cd code-dispatcher && go test ./... -run TestAuth
Success:
- requested follow-up implemented
- tests pass
EOF
```

Use resume only when continuity helps. Do not carry stale context forward blindly.

## Review Gate After Each Wave

After each wave, the host agent MUST review results and classify every task:

- `pass`: task met the objective and respected file scope
- `fail`: task did not meet the objective or failed validation
- `conflict`: task edited outside its scope or collided with another task's ownership
- `follow-up`: task succeeded but revealed new work for a later wave

Wave review checklist:

1. Did each task stay inside its declared file scope?
2. Did each task pass its validation command?
3. Did any task create overlap that was not planned?
4. Did results unlock a new frontier of independent tasks?
5. Should any follow-up reuse `session_id`?

## Conflict Handling

When a same-wave conflict occurs:

1. Do not silently merge and move on.
2. Record which task crossed its file scope.
3. Create a dedicated merge/fix task in the next wave, or re-run the conflicting tasks with corrected scopes.
4. If the conflict came from unsafe shared workspace assumptions, switch to separate worktrees/clones before retrying.

## Completion Rule

Stop the loop only when:

- all user-visible work is complete
- all required validations have passed
- no unresolved same-wave conflicts remain
- no known follow-up tasks remain for another wave

Then run the final verification and report the complete result to the user.

## Minimal Execution Loop

```text
repeat:
  1. identify the next frontier of implementation-ready tasks
  2. verify same-wave file scopes are mutually exclusive
  3. choose backend per task
  4. dispatch via code-dispatcher --parallel
  5. review results and classify pass/fail/conflict/follow-up
  6. if done: final verification and exit
  7. else: build the next wave from the reviewed results
```
