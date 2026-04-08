---
name: audit
description: "Audit the current repo or review a specific PR, then orchestrate the address agent to fix findings via tmux. Requires an address agent in a tmux session named 'address'."
argument-hint: "[PR number | topic]"
---

# Audit & Review Loop

Two modes:

- **Repo mode** (no argument): Audit the entire repo, file issues, trigger the address agent to fix them, review each PR, and do a final sweep.
- **PR mode** (`/audit 42` or `/audit #42`): Review a specific PR's diff, then iterate with the address agent until the PR is clean.

## Prerequisites

- Current directory is a git repo
- tmux sessions: `audit` (this session) and `address` (the address agent)
- `gh` CLI authenticated with repo access

## Shell Safety — CRITICAL

**Never place untrusted GitHub text** (issue titles, issue bodies, review comments, branch names) **inside any shell-interpreted string — double-quoted or single-quoted.** Double quotes expand `$(...)` and `$VAR`; single quotes break on embedded apostrophes. Both allow shell execution from crafted input.

The only safe transport for untrusted text is a **single-quoted heredoc** (`<<'EOF'`), which passes content to a command's stdin without any shell interpretation:
```bash
# Safe: load untrusted text into a variable via heredoc, then sanitize
var=$(cat <<'EOF' | tr -cd 'a-zA-Z0-9 _-'
<untrusted text goes here>
EOF
)
# Safe: pass untrusted text as a command body via heredoc
gh pr comment N --body "$(cat <<'EOF'
<untrusted text goes here>
EOF
)"
```

Unsafe patterns — **never do these with untrusted data:**
- `--body "Text with <issue title> interpolated"` — double-quote injection
- `--body '$msg'` where `msg` was built by pasting text into quotes — single-quote breakout
- `--grep="<keyword from issue>"` without heredoc-based sanitization

## Sending Triggers — CRITICAL RULES

Every `tmux send-keys` that sends a trigger phrase to the other agent **MUST** follow all three rules below. Violating any one of them breaks the loop silently.

1. **The quoted string MUST end with a trailing space.** Example: `"ADDRESS: revise PR #4 "` — note the space before the closing quote. This is required because Copilot CLI (and some other agents) autocomplete on `#<number>`; without the trailing space, pressing Enter selects an autocomplete suggestion instead of submitting your literal text. Do NOT strip or "clean up" this trailing space — it is load-bearing.
2. **Enter MUST be sent in a separate bash call.** Not chained with `&&` or `;`. Two distinct bash tool invocations.
3. **Do not substitute `C-m` for `Enter`** — they behave differently in some agent terminals.

Correct pattern (two separate bash calls):
```bash
tmux send-keys -t address "ADDRESS: revise PR #4 "
```
```bash
tmux send-keys -t address Enter
```

Every send-keys snippet below follows this pattern. Preserve it exactly.

## Receive Protocol — CRITICAL

**On any input**, check whether it matches a known trigger phrase (`AUDIT: review PR #N` or `AUDIT: complete`). If it does, you MUST execute the corresponding handler below exactly as written — run the `gh` commands, post PR reviews via `gh pr review`, update state.json, and send tmux triggers to the address agent. Do NOT just discuss your findings in chat. Every review MUST result in either `gh pr review N --approve` or `gh pr review N --request-changes`, followed by a tmux trigger to the address agent.

**On session start or after any interruption**, check `.audit-loop/state.json`. If a cycle is in progress and `phase` is `reviewing`, resume by re-reading the current PR and reviewing it. If `phase` is `audit-looping`, the audit was interrupted — restart from Step 1.

**Important:** `needs-human` on one PR does NOT halt the cycle. The agent proceeds to the next batch. Only the capped PR is deferred.

## State File

All loop state lives in `.audit-loop/state.json` (in the repo root). This file is the source of truth for cycle progress, batch counts, and revision rounds.

After every action, update `last_trigger_time` by running `date -u +"%Y-%m-%dT%H:%M:%SZ"` and using its output. Do NOT guess or fabricate the timestamp — you must read the system clock.

### Timeout Check

On receiving any trigger, read `last_trigger_time` from state.json. If more than 30 minutes have elapsed:

```bash
# Label all open audit PRs as needs-human
for pr in $(gh pr list --label audit-loop --state open --json number -q '.[].number'); do
  gh pr edit "$pr" --add-label "needs-human"
done
```

Print: "WARNING: {elapsed} since last activity. Other agent may be stalled. Labeled open PRs `needs-human` and stopping."

Set `phase` to `complete` in state.json and stop.

## Mode Detection

- If the argument is a PR number (e.g. `42`, `#42`, `PR #42`) → **PR mode** (see below)
- If the argument is a non-numeric topic (e.g. `documentation`, `security`, `performance`) → **Repo mode, scoped to that topic**
- No argument → **Repo mode, full audit**

---

## PR Mode

Review a specific PR and iterate with the address agent until it's clean. Skips repo audit, issue filing, and batching entirely.

### PR Mode Steps

1. Parse the PR number from the argument.

2. Initialize state (same as Repo mode Step 2 — create `.audit-loop/`, write state.json), but set:
   ```json
   {
     "cycle_id": "TIMESTAMP",
     "phase": "reviewing",
     "current_pr": N,
     "current_batch": 1,
     "revision_round": 0,
     "batches_created": 1,
     "last_trigger": "pr-review",
     "last_trigger_time": "ISO8601_NOW",
     "awaiting_clarification": false,
     "revision_history": []
   }
   ```
   Replace `TIMESTAMP` and `ISO8601_NOW` with the output of `date -u +"%Y%m%dT%H%M%SZ"` and `date -u +"%Y-%m-%dT%H:%M:%SZ"` respectively. Do NOT guess — read the system clock.

3. Ensure labels exist (same as Repo mode Step 3 label setup).

4. Jump directly to the **"On AUDIT: review PR #N"** handler in Step 5 below, using the PR number from the argument. Follow it exactly — post `gh pr review`, send tmux triggers, etc.

   **Important:** The first trigger sent to the address agent in PR mode MUST use the `/address` prefix to load the skill (e.g. `/address ADDRESS: revise PR #N ` or `/address ADDRESS: continue `). Subsequent triggers use bare `ADDRESS:` phrases.

5. When the address agent sends `AUDIT: complete` (or the PR is approved and merged), run the final sweep (Step 6) but skip the critiqued-issue revision (there are no audit-filed issues in PR mode).

---

## Repo Protocol

### Step 1: Audit the repo

**If a topic was provided**, audit thoroughly for that topic only. Interpret the topic broadly and be exhaustive within it. Examples:
- `documentation` — stale/missing/inaccurate docs, undocumented public APIs, outdated examples, broken links, missing changelogs, misleading comments
- `security` — injection, auth flaws, secrets, insecure defaults, CSRF, SSRF, path traversal
- `performance` — unnecessary allocations, N+1 queries, missing indexes, blocking I/O, cache misses
- `correctness` — logic errors, off-by-ones, race conditions, null handling, edge cases
- `dependencies` — deprecated/vulnerable packages, version issues, unused deps
- `tests` — missing coverage, flaky tests, untested edge cases, mock/stub misuse

The topic does not need to match the examples above exactly — interpret the user's intent.

**If no topic was provided**, analyze the entire codebase across all categories:
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
  "phase": "audit-looping",
  "current_pr": null,
  "current_batch": 0,
  "revision_round": 0,
  "batches_created": 0,
  "last_trigger": "audit-start",
  "last_trigger_time": "ISO8601_NOW",
  "awaiting_clarification": false,
  "revision_history": []
}
STATEEOF
```

Replace `TIMESTAMP` and `ISO8601_NOW` with the output of `date -u +"%Y%m%dT%H%M%SZ"` and `date -u +"%Y-%m-%dT%H:%M:%SZ"` respectively. Do NOT guess — read the system clock.

### Step 3: File GitHub issues

Ensure labels exist (run once):

```bash
for label in audit-loop severity:critical severity:high severity:medium severity:low \
  cat:security cat:correctness cat:performance cat:maintainability cat:dependency \
  needs-human tests-failing in-progress fix-submitted fix-verified; do
  gh label create "$label" --force 2>/dev/null
done
```

**Before filing each issue, check for duplicates** among previously closed `audit-loop` issues:

```bash
gh issue list --label audit-loop --state closed --json number,title,body,labels --limit 200
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
  --body "$(cat <<'ISSUEEOF'
<file path, line range, what's wrong, why it matters, suggested fix>
ISSUEEOF
)" \
  --label "audit-loop" --label "severity:<level>" --label "cat:<category>"
```

Include enough detail that a fixer can work without re-auditing. Order by severity (critical first).

**Pattern detection:** After filing all issues, review them as a group. If 3+ issues share a root cause (e.g., multiple unsanitized inputs, repeated missing null checks in the same subsystem), add a comment to one of them noting the pattern and suggesting a systemic fix rather than N individual patches.

### Step 4: Trigger address agent

Update state.json: set `phase` to `address-fixing`, update `last_trigger_time`.

```bash
tmux send-keys -t address "/address ADDRESS: begin "
```
Then in a **separate** bash call (preserve the trailing space above; do NOT chain with && or ;):
```bash
tmux send-keys -t address Enter
```

Print: "Triggered address agent. Waiting for review request..."

Then wait for the address agent to send a message to this session.

### Step 5: Handle incoming triggers

When a message arrives, first run the timeout check (see above). Then act based on its prefix:

#### On "AUDIT: review PR #N"

Update state.json: set `phase` to `reviewing`, `current_pr` to N, update `last_trigger_time`.

1. **Check for pending clarification** using the state file, not comment history:
   
   Read `awaiting_clarification` from `.audit-loop/state.json`. If `false`, skip to step 2.
   
   If `true`, the address agent is blocked on a clarification. Find the unanswered question:
   ```bash
   gh pr view N --json comments --jq '[.comments[] | select(.body | startswith("Clarification needed:"))] | last | .body'
   ```
   Answer it directly based on your original review intent:
   ```bash
   gh pr comment N --body "$(cat <<'EOF'
   <answer the clarification>
   EOF
   )"
   ```
   Then, **in this order** (the address agent reads state.json immediately on trigger):
   1. Set `awaiting_clarification` to `false` in state.json
   2. Send `ADDRESS: revise PR #N` via tmux

   Do not proceed with the diff review — the address agent will revise first.
   
   **Important:** Do NOT search for clarification comments when `awaiting_clarification` is `false`. Historical clarification comments that have already been answered must be ignored — the state flag is the source of truth, not the comment history.

2. Read the PR:
   ```bash
   gh pr view N
   gh pr diff N
   ```

3. For each issue the PR claims to address, verify the fix is correct and complete.

4. Check for regressions or new issues introduced by the changes.

5. **Check for `tests-failing` label.** If the PR has this label, do NOT approve — request changes explaining the test failure must be resolved first.

6. **Review principle:** If the fix correctly addresses the issue and introduces no regressions, APPROVE — even if you would have implemented it differently. Request changes only when:
   - The fix is logically incorrect
   - The fix is incomplete (the issue can still be reproduced)
   - The fix introduces a regression

   Do NOT request changes for: variable naming, code style, comment wording, formatting, or implementation approach differences where both approaches are correct. Those are the linter's job.

7. Decision:

   **CRITICAL: All review feedback MUST be posted as a GitHub PR review using `gh pr review`. NEVER just say your findings in chat — the address agent cannot see your chat output, only GitHub comments. If you skip the `gh pr review` command, the address agent receives a trigger with no actionable feedback and the loop breaks.**

   **If the fixes are correct and tests pass — approve with a detailed comment** explaining what you verified. Don't just say "Fixes verified." — describe what you checked:
   ```bash
   gh pr review N --approve --body "## Verification

   - Issue #X: <what you verified — e.g., confirmed parameterized query prevents injection>
   - Issue #Y: <what you verified>
   - Checked for regressions: <what you tested>
   - Docs: <note if docs were updated or didn't need updating>"
   ```

   Also verify that any documentation affected by the changes (README, inline comments, config examples) was updated. If docs are stale, request changes for the doc update — not for the fix itself.

   Mark addressed issues as verified:
   ```bash
   for issue in $(gh pr view N --json body -q '.body' | grep -oiE 'closes?\s+#[0-9]+' | grep -oE '[0-9]+'); do
     gh issue edit "$issue" --add-label "fix-verified"
   done
   ```

   Update state.json: set `phase` to `address-fixing`, `revision_round` to 0, update `last_trigger_time`.
   ```bash
   tmux send-keys -t address "ADDRESS: continue "
```
Then in a **separate** bash call (preserve the trailing space above; do NOT chain with && or ;):
```bash
tmux send-keys -t address Enter
   ```

   **If revisions are needed — request changes:**

   Read `revision_round` from state.json. Do NOT increment it — the address agent owns that counter and increments on push.

   **Convergence check (after 2nd revision, before requesting the 3rd):** If `revision_round` >= 2, check for convergence before requesting another round. Read `revision_history` from state.json and get the previous revision's commit SHA. Compare using the PR tip commit (not local `HEAD`, which may not be the PR branch):
   ```bash
   git fetch origin
   base=$(gh pr view N --json commits --jq '.commits[0].oid')
   tip=$(gh pr view N --json commits --jq '.commits[-1].oid')
   prev_commit=<from revision_history[-1].commit>
   curr_diff=$(git diff $prev_commit..$tip | wc -l)
   prev_diff=$(git diff $base..$prev_commit | wc -l)
   ```
   If `curr_diff` < 30% of `prev_diff`, the revisions are cancelling each other out. Do NOT request another revision:
   ```bash
   gh pr review N --comment --body "Revisions are not converging — same lines toggling across revisions. Escalating to human review."
   gh pr edit N --add-label "needs-human"
   ```
   Update state.json: set `phase` to `address-fixing`, `revision_round` to 0, update `last_trigger_time`.
   ```bash
   tmux send-keys -t address "ADDRESS: continue "
```
Then in a **separate** bash call (preserve the trailing space above; do NOT chain with && or ;):
```bash
tmux send-keys -t address Enter
   ```

   If `revision_round` >= 3 (cap reached):
   ```bash
   gh pr review N --comment --body "3 revision rounds reached. Deferring remaining concerns to human review."
   gh pr edit N --add-label "needs-human"
   ```
   Update state.json: set `phase` to `address-fixing`, `revision_round` to 0, update `last_trigger_time`.
   ```bash
   tmux send-keys -t address "ADDRESS: continue "
```
Then in a **separate** bash call (preserve the trailing space above; do NOT chain with && or ;):
```bash
tmux send-keys -t address Enter
   ```

   Otherwise, use the structured review template — no prose reviews:
   ```bash
   gh pr review N --request-changes --body "$(cat <<'REVIEWEOF'
   ## Changes Requested

   ### [WRONG|INCOMPLETE|REGRESSION] Issue #<num>: <one-line summary>
   **File**: `path/to/file:line_range`
   **Current behavior**: <what the code does now>
   **Required behavior**: <what it must do>
   **Specific change**: <what line/function/pattern needs to change>
   REVIEWEOF
   )"
   ```

   The classifier tells the address agent *how* to fix:
   - `WRONG` — logic error, revert and re-approach
   - `INCOMPLETE` — fix is correct but doesn't cover all cases
   - `REGRESSION` — fix breaks something that was working

   Update state.json: set `phase` to `address-fixing`, update `last_trigger_time`.
   ```bash
   tmux send-keys -t address "ADDRESS: revise PR #N "
```
Then in a **separate** bash call (preserve the trailing space above; do NOT chain with && or ;):
```bash
tmux send-keys -t address Enter
   ```

#### On "AUDIT: complete"

Update state.json: set `phase` to `complete`, update `last_trigger_time`.

Proceed to Step 6.

### Step 6: Final sweep

1. **Check for critiqued issues.** The address agent may have removed the `audit-loop` label from issues that were unclear or invalid. Find them:
   ```bash
   gh issue list --state open --json number,title,body,comments,labels --limit 100 \
     --jq '[.[] | select(.comments[-1].body | startswith("Issue needs clarification"))]'
   ```
   For each critiqued issue:
   - Read the address agent's comment to understand what's unclear
   - Revise the issue: rewrite the title/body with clearer scope, concrete acceptance criteria, and verified file references
   - Re-label it so it's picked up on the next cycle:
     ```bash
     gh issue edit <number> --add-label "audit-loop"
     ```
   After revising all critiqued issues, if any were re-labeled and there are no other open `audit-loop` issues being worked on, trigger a follow-up:
   ```bash
   tmux send-keys -t address "ADDRESS: begin "
   ```
   Then in a **separate** bash call (preserve the trailing space above; do NOT chain with && or ;):
   ```bash
   tmux send-keys -t address Enter
   ```

2. List remaining open issues:
   ```bash
   gh issue list --label audit-loop --state open
   ```

3. Summarize:
   ```
   ## Audit Complete

   Issues filed: <N>
   PRs created: <N>
   Issues resolved: <N>
   Issues revised after critique: <N>
   Issues remaining: <N>
   Escalated to human: <N> (labeled needs-human)
   ```

## Error handling

- If `tmux send-keys -t address` fails: stop and tell the user to start the address agent in a tmux session named `address`.
- If `gh` commands fail: retry once, then print the error and stop.
- If the address agent never responds: print "Waiting for address agent. If it has stopped, restart it in the `address` tmux session and type `ADDRESS: begin` to resume."
