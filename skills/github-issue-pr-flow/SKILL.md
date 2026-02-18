---
name: github-issue-pr-flow
description: "Autonomous, long-running GitHub delivery workflow. Issue decomposition, PR creation, bot review polling (Gemini/CodeRabbit), autonomous rebut-or-fix, CI gating, squash merge — all via gh CLI. code-dispatcher (codex-only) available as strong backup for critical/high-priority findings. Triggers on end-to-end issue-driven delivery, PR lifecycle automation, or bot review triage."
---

# GitHub Issue → PR Delivery Flow

## Design Principles
- **High autonomy**: agent makes all decomposition, branching, review-triage decisions independently. Only escalate to user on genuine ambiguity or destructive actions.
- **Long-running**: the full cycle (issue → merge) may span minutes to hours. Each phase completes before advancing; no user polling between phases.
- **Traceable**: every code change links back to an issue via `Closes #N`.

## Context Detection
```bash
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
BASE=$(gh repo view --json defaultBranchRef -q .defaultBranchRef.name)
```
Never ask the user for repo or base branch.

## code-dispatcher (Strong Backup)
The host agent does all normal work directly. code-dispatcher (`--backend codex`) is available as a strong backup — call it when bot reviews flag something as `critical` or `high priority` and you need heavier firepower.

```bash
code-dispatcher --backend codex - <<'EOF'
<goal + relevant context (CI errors, review excerpts, issue number)>
EOF
```

## Autonomous Workflow
The agent executes phases sequentially. No user confirmation gates between phases.

### Phase 0: Sync Base Branch
Pull latest base branch before planning and implementation:
```bash
git fetch origin
git switch $BASE
git pull --ff-only
```
If `git pull --ff-only` fails, escalate (do not hard reset automatically).

### Phase 1: Decompose → Issues
Agent decomposes **by task and deliverables**, not by file counts.

Principles:
- Prefer **one issue = one user-verifiable outcome** (with acceptance criteria + test evidence)
- Prefer **one PR closes one primary issue** (`Closes #N`)
- Split when work can be done/merged independently, or when review risk is high
- Create an **epic** only when there are multiple child issues that benefit from a parent umbrella
- Use sub-issues (checklist `- [ ] #N`) when tasks are parallelizable or have clear dependency edges
- If scope grows mid-flight, split into new issue(s) and keep current PR focused

Create issue:
```bash
gh issue create --title "<title>" --body "<body_with_acceptance_criteria>" --label "<labels>"
```

### Phase 2: Implement → Branch + Commits
Branch: `feat/issue-<number>-<slug>` or `fix/issue-<number>-<slug>`

Create branch from updated base:
```bash
git switch -c feat/issue-<number>-<slug>
```

One branch per issue chain. Commits scoped to linked issue(s).

### Phase 3: PR → Link Issues
```bash
gh pr create --base $BASE --head <branch> --title "<title>" --body "Closes #<N>

<change summary>"
```
Body must contain `Closes #<primary>` and optionally `Relates #<secondary>`.

### Phase 4: Poll Bot Reviews
Wait for bot review window, then collect:
```bash
sleep 300 && gh pr view <pr_number> --json reviews \
  --jq '[.reviews[] | {author: .author.login, state: .state, body: .body[:600]}]'
```
Also poll CI:
```bash
gh pr checks <pr_number>
```
If no reviews after wait, proceed — do not block indefinitely.

### Phase 5: Triage (Autonomous)
For each review finding, agent decides independently.
If a finding is `critical/high priority` → consider calling code-dispatcher for backup.
```text
finding
 ├─ real bug / violation     → fix, commit, push
 ├─ misunderstanding         → reply with evidence, dismiss
 └─ style disagreement       → follow repo convention, explain once
```
See `references/review-playbook.md` for reply templates.

After **every** fix or rebuttal:
1. **Reply under the specific review comment** explaining what was fixed or why it was rebutted
2. **Resolve the conversation thread** once addressed
3. Re-request review and re-poll CI

Never skip replying or resolving — every review item must have a visible response.

Loop: PR → review → fix/rebut → re-review → fix/rebut → end.
Max 2 triage rounds. After round 2, proceed to merge (or escalate if still blocked).

**CI awareness**: after each push, poll `gh pr checks <pr_number>` before proceeding. If CI fails, read logs, attempt fix, and push again. CI must be green before entering Phase 6.

### Phase 6: Squash Merge
```bash
gh pr merge <pr_number> --squash --delete-branch
```
Verify linked issues auto-closed. Close residual issues manually if needed.

## Escalation Policy
Only interrupt user when:
1. Genuine ambiguity that cannot be resolved from codebase context
2. Destructive action (force push, closing someone else's issue)
3. 2 triage rounds exhausted with unresolved blocking reviews
4. CI failure that requires credentials or infra access

Everything else: decide and proceed.

## Failure Handling
- Bot reviews absent after wait → proceed with CI gating alone
- Conflicting bot feedback → prioritize reproducible evidence and tests
- Scope creep detected → split into new issue, keep current PR focused
- `gh` CLI failure → retry once, then report error and stop
