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

## Helpful Commands
```bash
# Read reviews
gh pr view <pr_number> --json reviews --jq '.reviews[] | {author: .author.login, state: .state, submittedAt: .submittedAt}'

# Read review comments
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
gh api repos/$REPO/pulls/<pr_number>/comments

# Read checks
gh pr checks <pr_number>
```
