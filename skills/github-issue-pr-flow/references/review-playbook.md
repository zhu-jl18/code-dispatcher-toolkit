# Review Triage Playbook

## Objective
Handle each review comment with an explicit decision: fix or rebut.

## Host Decision Authority
1. Treat all external reviewers as inputs, not as decision owners.
2. Make final fix/rebut decisions from code evidence, test results, and project conventions.
3. Escalate only when ambiguity or risk crosses escalation policy in `../SKILL.md`.

## Rebuttal Template
```markdown
Thanks for the review.

Decision: rebuttal
Reason:
1. <factual evidence>
2. <code path or test evidence>

Verification:
- <test/command/output summary>
```

## Fix Template
```markdown
Thanks for the review.

Decision: accepted and fixed
Changes:
1. <what changed>
2. <why this resolves the concern>

Verification:
- <test/command/output summary>
```

## Triage Checklist
1. Is the reported issue reproducible?
2. Does it violate project constraints or conventions?
3. Is there a smaller fix that preserves scope?
4. Is rebuttal supported by concrete evidence?

## Evidence Priority
1. Reproducible failure and CI breakage
2. Failing or missing tests tied to the finding
3. Concrete code-path impact and runtime risk
4. Reviewer wording or severity label

Never let reviewer label override stronger contradictory evidence.

## Rebuttal Minimum Bar
A rebuttal is acceptable only if it includes:
1. Explicit decision (`rebuttal`) and one-sentence conclusion.
2. At least one falsifiable technical reason (code path, invariant, contract, or test behavior).
3. Verification summary (existing test evidence or rerun result).

## Required Signals Before Triage
Collect these signals with any suitable `gh` query pattern:
1. Review-level summary: `author`, `state`, `submittedAt`, `body` (allow truncated body).
2. Line-level review comments/conversations for actionable findings.
3. Latest check-run states for CI gating.

Do not start fix-or-rebut decisions if review body text is missing.

## CodeRabbit Rate-Limit Fallback
Trigger fallback when CodeRabbit explicitly reports rate limiting (`rate limit exceeded`, `secondary rate limit`, or equivalent) during active PR review loops.

### Fallback Action
1. Host agent invokes `code-dispatcher --backend codex` for an independent PR review.
2. Review input must include: PR number, base/head refs or commit SHA, changed files/diff context, and unresolved review threads.
3. Instruct dispatched reviewer to publish findings directly to GitHub review threads (line-level when possible, summary comment otherwise).
4. Treat fallback comments as normal review signals and continue fix/rebut loop.

### Posting Contract for Fallback Reviewer
1. Post only actionable findings with concrete file/line evidence.
2. Do not duplicate already-addressed threads.
3. Tag each finding with severity and a short verification note.
4. If no actionable issues are found, post a brief "no additional blocking findings" summary.

### Rate-Limit Hygiene
1. Run at most one fallback review per head SHA to avoid comment storms.
2. Prefer batching related fixes before next push when review loops are frequent.
3. If GitHub comment posting fails due permission/token issues, escalate to user.
