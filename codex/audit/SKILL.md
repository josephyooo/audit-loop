---
name: audit
description: Audit the current repo, open GitHub issues for findings, then orchestrate Claude Code to fix them in batched PRs via tmux. Reviews each PR Claude opens and iterates until clean. Requires a Claude Code instance in a tmux session named "claude".
---

# Audit & Review Loop (Codex Side)

Audit the repo, file issues, trigger Claude to fix them, review each PR, and do a final sweep.

## Prerequisites

- Current directory is a git repo
- tmux sessions: `codex` (this session) and `claude` (Claude Code)
- `gh` CLI authenticated with repo access

## Protocol

### Step 1: Audit the repo

Analyze the entire codebase for substantive issues across these categories:
- **security** — injection, auth flaws, secrets, insecure defaults
- **correctness** — logic errors, off-by-ones, race conditions, null handling
- **performance** — unnecessary allocations, N+1 queries, missing indexes
- **maintainability** — dead code, unclear control flow, missing error handling
- **dependency** — deprecated/vulnerable packages, version issues

Skip trivial style nits unless they mask bugs.

### Step 2: File GitHub issues

Ensure labels exist (run once):

```bash
for label in codex-audit severity:critical severity:high severity:medium severity:low \
  cat:security cat:correctness cat:performance cat:maintainability cat:dependency; do
  gh label create "$label" --force 2>/dev/null
done
```

For each finding, create an issue:

```bash
gh issue create --title "<concise title>" \
  --body "<file path, line range, what's wrong, why it matters, suggested fix>" \
  --label "codex-audit" --label "severity:<level>" --label "cat:<category>"
```

Include enough detail that a fixer can work without re-auditing. Order by severity (critical first).

### Step 3: Trigger Claude

```bash
tmux send-keys -t claude "ADDRESS: begin" Enter
```

Print: "Triggered Claude. Waiting for review request..."

Then wait for Claude to send a message to this session.

### Step 4: Handle incoming triggers

When a message arrives, act based on its prefix:

#### On "AUDIT: review PR #N"

1. Read the PR:
   ```bash
   gh pr view N
   gh pr diff N
   ```

2. For each issue the PR claims to address, verify the fix is correct and complete.

3. Check for regressions or new issues introduced by the changes.

4. Decision:

   **If the fixes are correct — approve:**
   ```bash
   gh pr review N --approve --body "Fixes verified."
   tmux send-keys -t claude "ADDRESS: continue" Enter
   ```

   **If revisions are needed — request changes:**
   ```bash
   gh pr review N --request-changes --body "<specific feedback per file/issue>"
   tmux send-keys -t claude "ADDRESS: revise PR #N" Enter
   ```

   **If this is the 3rd round of changes-requested on this PR — stop iterating:**
   Check revision count:
   ```bash
   gh pr view N --json reviews --jq '[.reviews[] | select(.state=="CHANGES_REQUESTED")] | length'
   ```
   If >= 3:
   ```bash
   gh pr review N --comment --body "3 revision rounds reached. Deferring remaining concerns to human review."
   gh pr edit N --add-label "needs-human"
   tmux send-keys -t claude "ADDRESS: continue" Enter
   ```

#### On "AUDIT: complete"

Proceed to Step 5.

### Step 5: Final sweep

1. List remaining open issues:
   ```bash
   gh issue list --label codex-audit --state open
   ```

2. Summarize:
   ```
   ## Audit Complete

   Issues filed: <N>
   PRs created: <N>
   Issues resolved: <N>
   Issues remaining: <N>
   Escalated to human: <N> (labeled needs-human)
   ```

## Error handling

- If `tmux send-keys -t claude` fails: stop and tell the user to start Claude Code in a tmux session named `claude`.
- If `gh` commands fail: retry once, then print the error and stop.
- If Claude never responds: print "Waiting for Claude. If it has stopped, restart it in the `claude` tmux session."
