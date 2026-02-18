# Bot Review Playbook

## Objective
Handle each bot comment with an explicit decision: fix or rebut.

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

## Required Signals Before Triage
Collect these signals with any suitable `gh` query pattern:
1. Review-level summary: `author`, `state`, `submittedAt`, `body` (allow truncated body).
2. Line-level review comments/conversations for actionable findings.
3. Latest check-run states for CI gating.

Do not start fix-or-rebut decisions if review body text is missing.
