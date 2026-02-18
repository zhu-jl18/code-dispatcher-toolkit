---
name: pr-review-reply
description: "Autonomous bot review triage skill. Tracks the current PR, fetches Gemini and CodeRabbit review findings, verifies each finding against code, decides fix-or-rebut, replies under the corresponding GitHub review thread, and resolves the thread. Triggers when user asks to handle, triage, respond to, or clear bot reviews on a PR."
---

# PR Review Reply

## Purpose
Autonomously handle all bot review findings on the current PR end-to-end:
fetch → verify → fix or rebut → reply under thread → resolve thread.

## Context Detection
```bash
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
PR=$(gh pr view --json number -q .number)
```

## Workflow

### Step 1: Fetch Review Signals
Collect both review-level summaries and line-level comment threads:

```bash
# Review summaries
gh pr view $PR --json reviews \
  --jq '[.reviews[] | {author: .author.login, state: .state, body: .body[:600]}]'

# Line-level comment threads (with IDs for reply/resolve)
gh api repos/$REPO/pulls/$PR/comments \
  --jq '[.[] | {id: .id, author: .user.login, path: .path, line: .original_line, body: .body[:400], in_reply_to: .in_reply_to_id}]'

# CI status
gh pr checks $PR
```

Filter for bot authors: `gemini-code-assist`, `coderabbitai`.
Skip comments that already have an owner reply or are marked resolved.

Get unresolved thread IDs via GraphQL (needed for resolving):
```bash
# Parse owner/repo from $REPO (e.g., "owner/repo" → owner="owner", name="repo")
OWNER="${REPO%/*}"
NAME="${REPO#*/}"

gh api graphql -f query='
query($owner: String!, $name: String!, $pr: Int!) {
  repository(owner: $owner, name: $name) {
    pullRequest(number: $pr) {
      reviewThreads(first: 50) {
        nodes { id isResolved comments(first: 1) { nodes { databaseId author { login } body } } }
      }
    }
  }
}' -F owner="$OWNER" -F name="$NAME" -F pr="$PR"
```

### Step 2: Verify Each Finding
For every unresolved bot finding, before deciding fix or rebut:
- Read the referenced file and line(s) from the local repo
- Check if the concern is reproducible or observable in the current code
- Cross-check with existing tests and CI output

Do not accept or reject a finding based on reviewer wording alone — code evidence is the deciding factor.

### Step 3: Fix or Rebut
See `references/triage-guide.md` for decision criteria and reply templates.

**Fix path:**
1. Make code change, ensure CI passes locally if possible
2. Commit and push

**Rebut path:**
1. Collect concrete evidence (file path, line, test result, invariant)
2. Draft reply with evidence

### Step 4: Reply Under Thread
Reply under the exact comment that raised the finding (not as a top-level PR comment):

```bash
gh api repos/$REPO/pulls/$PR/comments \
  -f body="<reply text>" \
  -F in_reply_to=<comment_id>
```

The root comment ID of a thread is the one **without** `in_reply_to_id`.

### Step 5: Resolve Thread
After replying, resolve the thread via GraphQL using the thread node ID from Step 1:

```bash
gh api graphql -f query='
mutation($threadId: ID!) {
  resolveReviewThread(input: { threadId: $threadId }) {
    thread { isResolved }
  }
}' -F threadId="<thread_node_id>"
```

### Step 6: Re-request Review
After all threads are handled, re-request review to trigger the next bot pass:

```bash
gh pr edit $PR --add-reviewer gemini-code-assist[bot],coderabbitai[bot]
```

Or use the GitHub UI if reviewer re-request via CLI is not available for bots.

## Loop Limit
Max 2 rounds of fix/rebut → re-review. After round 2, stop and summarize outstanding issues to user.

## Escalation
Stop and report to user when:
- A finding is `critical`/`high priority` and fix requires substantial rework → consider calling `code-dispatcher --backend codex`
- CI fails after fix and root cause is unclear
- A thread cannot be resolved due to permissions

## Hard Rules
- Every handled finding **must** have a reply in its thread before resolving
- Never resolve a thread without replying — silent resolves are not allowed
- Never post a top-level PR comment as a substitute for replying in the thread

## Error Handling

### Local Repo State Checks
Before making changes:
- Verify current branch matches PR head branch: `gh pr view $PR --json headRefName -q .headRefName`
- If on wrong branch: stop and report to user — do not auto-checkout
- If uncommitted changes exist: stash or commit before proceeding

### Missing File in Finding
If a bot comment references a file that doesn't exist locally:
- Check PR diff: `gh pr diff $PR --name-only`
- If file was deleted/renamed in PR: skip the finding, reply noting file no longer exists
- If file never existed: rebut with file listing evidence

### Comment Without Line Number
Some bot comments may lack line context:
- Use PR diff to locate relevant code: `gh pr diff $PR -- <path>`
- If still unclear: treat as review-level comment — reply at PR level, do not resolve

### Thread Resolution Fails
If GraphQL mutation to resolve thread fails:
- Check if thread was already resolved by someone else
- If permission issue: skip resolution and note in summary
- If ID invalid: re-fetch thread IDs and retry
