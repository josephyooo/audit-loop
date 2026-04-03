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

## Receive Protocol

**On any input**, check whether it matches a known trigger phrase (`AUDIT: review PR #N` or `AUDIT: complete`). If it does, handle it per the protocol below.

**On session start or after any interruption**, check `.audit-loop/state.json`. If a cycle is in progress and `phase` is `codex-reviewing`, resume by re-reading the current PR and reviewing it. If `phase` is `codex-auditing`, the audit was interrupted — restart from Step 1.

**Important:** `needs-human` on one PR does NOT halt the cycle. The agent proceeds to the next batch. Only the capped PR is deferred.

## State File

All loop state lives in `.audit-loop/state.json` (in the repo root). This file is the source of truth for cycle progress, batch counts, and revision rounds.

After every action, update `last_trigger_time` to the current ISO 8601 timestamp.

### Timeout Check

On receiving any trigger, read `last_trigger_time` from state.json. If more than 30 minutes have elapsed:

```bash
# Label all open audit PRs as needs-human
for pr in $(gh pr list --label codex-audit --state open --json number -q '.[].number'); do
  gh pr edit "$pr" --add-label "needs-human"
done
```

Print: "WARNING: {elapsed} since last activity. Other agent may be stalled. Labeled open PRs `needs-human` and stopping."

Set `phase` to `complete` in state.json and stop.

## Protocol

### Step 1: Audit the repo

Analyze the entire codebase for substantive issues across these categories:
- **security** — injection, auth flaws, secrets, insecure defaults
- **correctness** — logic errors, off-by-ones, race conditions, null handling
- **performance** — unnecessary allocations, N+1 queries, missing indexes
- **maintainability** — dead code, unclear control flow, missing error handling
- **dependency** — deprecated/vulnerable packages, version issues

Skip trivial style nits unless they mask bugs.

### Step 2: Initialize state

Create the `.audit-loop/` directory and add it to `.gitignore`:

```bash
mkdir -p .audit-loop
grep -qxF '.audit-loop/' .gitignore 2>/dev/null || echo '.audit-loop/' >> .gitignore
```

Write initial state:

```bash
cat > .audit-loop/state.json << 'STATEEOF'
{
  "cycle_id": "TIMESTAMP",
  "phase": "codex-auditing",
  "current_pr": null,
  "current_batch": 0,
  "revision_round": 0,
  "batches_created": 0,
  "last_trigger": "audit-start",
  "last_trigger_time": "ISO8601_NOW"
}
STATEEOF
```

Replace `TIMESTAMP` and `ISO8601_NOW` with actual values.

### Step 3: File GitHub issues

Ensure labels exist (run once):

```bash
for label in codex-audit severity:critical severity:high severity:medium severity:low \
  cat:security cat:correctness cat:performance cat:maintainability cat:dependency \
  needs-human tests-failing in-progress fix-submitted fix-verified; do
  gh label create "$label" --force 2>/dev/null
done
```

**Before filing each issue, check for duplicates** among previously closed `codex-audit` issues:

```bash
gh issue list --label codex-audit --state closed --json number,title,body,labels --limit 200
```

For each finding, search the closed issues for matches on the same file path and category. If a match is found:
- **Reopen the original issue** instead of filing a new one:
  ```bash
  gh issue reopen <number>
  gh issue comment <number> --body "Recurrence detected. This issue was previously fixed but has reappeared. Escalating severity — a deeper/systemic fix is needed."
  ```
- **Escalate severity** by one level (low→medium, medium→high, high→critical):
  ```bash
  gh issue edit <number> --remove-label "severity:<old>" --add-label "severity:<new>"
  ```
- Remove any stale state labels (`fix-submitted`, `fix-verified`) from the reopened issue.

If no duplicate is found, create a new issue:

```bash
gh issue create --title "<concise title>" \
  --body "<file path, line range, what's wrong, why it matters, suggested fix>" \
  --label "codex-audit" --label "severity:<level>" --label "cat:<category>"
```

Include enough detail that a fixer can work without re-auditing. Order by severity (critical first).

**Pattern detection:** After filing all issues, review them as a group. If 3+ issues share a root cause (e.g., multiple unsanitized inputs, repeated missing null checks in the same subsystem), add a comment to one of them noting the pattern and suggesting a systemic fix rather than N individual patches.

### Step 4: Trigger Claude

Update state.json: set `phase` to `claude-fixing`, update `last_trigger_time`.

```bash
tmux send-keys -t claude "ADDRESS: begin" Enter
```

Print: "Triggered Claude. Waiting for review request..."

Then wait for Claude to send a message to this session.

### Step 5: Handle incoming triggers

When a message arrives, first run the timeout check (see above). Then act based on its prefix:

#### On "AUDIT: review PR #N"

Update state.json: set `phase` to `codex-reviewing`, `current_pr` to N, update `last_trigger_time`.

1. Read the PR:
   ```bash
   gh pr view N
   gh pr diff N
   ```

2. For each issue the PR claims to address, verify the fix is correct and complete.

3. Check for regressions or new issues introduced by the changes.

4. **Check for `tests-failing` label.** If the PR has this label, do NOT approve — request changes explaining the test failure must be resolved first.

5. Decision:

   **If the fixes are correct and tests pass — approve:**
   ```bash
   gh pr review N --approve --body "Fixes verified."
   ```

   Mark addressed issues as verified:
   ```bash
   for issue in $(gh pr view N --json body -q '.body' | grep -oiE 'closes?\s+#[0-9]+' | grep -oE '[0-9]+'); do
     gh issue edit "$issue" --add-label "fix-verified"
   done
   ```

   Update state.json: set `phase` to `claude-fixing`, `revision_round` to 0, update `last_trigger_time`.
   ```bash
   tmux send-keys -t claude "ADDRESS: continue" Enter
   ```

   **If revisions are needed — request changes:**
   Read `revision_round` from state.json. Increment it by 1 and write it back.

   If `revision_round` >= 3:
   ```bash
   gh pr review N --comment --body "3 revision rounds reached. Deferring remaining concerns to human review."
   gh pr edit N --add-label "needs-human"
   ```
   Update state.json: set `phase` to `claude-fixing`, `revision_round` to 0, update `last_trigger_time`.
   ```bash
   tmux send-keys -t claude "ADDRESS: continue" Enter
   ```

   Otherwise:
   ```bash
   gh pr review N --request-changes --body "<specific feedback per file/issue>"
   ```
   Update state.json: set `phase` to `claude-fixing`, update `last_trigger_time`.
   ```bash
   tmux send-keys -t claude "ADDRESS: revise PR #N" Enter
   ```

#### On "AUDIT: complete"

Update state.json: set `phase` to `complete`, update `last_trigger_time`.

Proceed to Step 6.

### Step 6: Final sweep

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
- If Claude never responds: print "Waiting for Claude. If it has stopped, restart it in the `claude` tmux session and type `ADDRESS: begin` to resume."
