---
name: address
description: "Address external audit findings. Works in two modes: (1) manual — paste findings or point to a file, (2) loop — respond to audit agent triggers (ADDRESS: begin/continue/revise) to fix GitHub issues in batched PRs with automated review."
argument-hint: "[filepath | ADDRESS: begin | ADDRESS: continue | ADDRESS: revise PR #N]"
---

# Address Audit Findings

Fix code issues identified by an external audit. Operates in two modes:

- **Manual mode**: user pastes findings or provides a file path. Parse, triage, fix, report.
- **Loop mode**: activated by `ADDRESS:` trigger phrases from the audit agent via tmux. Reads GitHub issues, fixes in batched PRs, coordinates review with the audit agent.

---

## Mode Detection

- If input starts with `ADDRESS:` → **loop mode** (see Loop Protocol below)
- Otherwise → **manual mode** (see Manual Protocol below)

---

## Loop Protocol

Triggered by the audit agent via tmux send-keys. All GitHub operations use `gh` CLI.

### Sending Triggers — CRITICAL RULES

Every `tmux send-keys` that sends a trigger phrase to the other agent **MUST** follow all three rules below. Violating any one of them breaks the loop silently.

1. **The quoted string MUST end with a trailing space.** Example: `"AUDIT: review PR #4 "` — note the space before the closing quote. This is required because Copilot CLI (and some other agents) autocomplete on `#<number>`; without the trailing space, pressing Enter selects an autocomplete suggestion instead of submitting your literal text. Do NOT strip or "clean up" this trailing space — it is load-bearing.
2. **Enter MUST be sent in a separate bash call.** Not chained with `&&` or `;`. Two distinct bash tool invocations.
3. **Do not substitute `C-m` for `Enter`** — they behave differently in some agent terminals.

Correct pattern (two separate bash calls):
```bash
tmux send-keys -t audit "AUDIT: review PR #4 "
```
```bash
tmux send-keys -t audit Enter
```

Every send-keys snippet below follows this pattern. Preserve it exactly.

### Receive Protocol — CRITICAL

**On any input**, check whether it matches a known trigger phrase (`ADDRESS: begin`, `ADDRESS: continue`, `ADDRESS: revise PR #N`). If it does, you MUST execute the corresponding handler below exactly as written — run the `gh` commands, create branches, commit code, open PRs, update state.json, and send tmux triggers to the audit agent. Do NOT just discuss your plan in chat. Every handler MUST end with a tmux trigger to the audit agent.

**On session start or after any interruption**, check `.audit-loop/state.json`. If a cycle is in progress and `phase` is `address-fixing`, resume:
- If `current_pr` is set, check whether that PR has been reviewed. If not, re-send `AUDIT: review PR #N` to the audit agent. If it has reviews, process the latest review as if receiving the appropriate trigger.
- If `current_pr` is null, treat it as `ADDRESS: begin` (start/continue grouping issues).

**Important:** `needs-human` on one PR does NOT halt the cycle. The agent proceeds to the next batch. Only the capped PR is deferred.

### State File

All loop state lives in `.audit-loop/state.json` (in the repo root, created by the `/audit` skill). This file is the source of truth for cycle progress, batch counts, and revision rounds.

After every action, update `last_trigger_time` by running `date -u +"%Y-%m-%dT%H:%M:%SZ"` and using its output. Do NOT guess or fabricate the timestamp — you must read the system clock.

### Additional State Fields

Beyond the base fields (`cycle_id`, `phase`, `current_pr`, `current_batch`, `revision_round`, `batches_created`, `last_trigger`, `last_trigger_time`), state.json also tracks:

- `awaiting_clarification`: boolean — `true` when the address agent posted a clarifying question and is blocked waiting for a response. Default `false`.
- `revision_history`: array of `{round, pr, commit, summary}` entries — The address agent writes one after each revision push. The `commit` SHA enables the audit agent's convergence detection.

Both skills read and write state.json. Always read-modify-write (don't overwrite the whole file).

### Timeout Check

On receiving any trigger, read `last_trigger_time` and `awaiting_clarification` from state.json.

**If `awaiting_clarification` is true:**
- If `last_trigger_time` < 2 hours ago: skip normal timeout. Print: "Cycle is blocked on clarification — waiting for response on PR #N."
- If `last_trigger_time` >= 2 hours ago: clarification was never answered.
  ```bash
  gh pr edit <current_pr> --add-label "needs-human"
  gh pr comment <current_pr> --body "Clarification not answered within 2 hours. Deferring to human review."
  ```
  Set `awaiting_clarification` to `false`, `current_pr` to null. Move on to the next batch.

**If `awaiting_clarification` is false** and more than 30 minutes have elapsed:

```bash
for pr in $(gh pr list --label audit-loop --state open --json number -q '.[].number'); do
  gh pr edit "$pr" --add-label "needs-human"
done
```

Print: "WARNING: {elapsed} since last activity. Other agent may be stalled. Labeled open PRs `needs-human` and stopping."

Update state.json: set `phase` to `complete`. Do NOT send a tmux trigger (the other agent may be dead). Stop.

### Trigger phrases

- `ADDRESS: begin` — Read all open `audit-loop` issues, plan batches, fix batch 1
- `ADDRESS: continue` — Current PR was approved (or escalated). Merge it, move to next batch.
- `ADDRESS: revise PR #N` — Audit agent requested changes. Read review comments and revise.

### Shell Safety — CRITICAL

**Never place untrusted GitHub text** (issue titles, issue bodies, review comments, branch names) **inside a double-quoted shell string.** Command substitution (`$(...)`, backticks) and variable expansion (`$VAR`) are live inside double quotes and will execute on the operator's machine.

Safe patterns for `--body` / `--title` / commit messages:
- `--body "$(cat <<'EOF' ... EOF)"` — single-quoted heredoc delimiter prevents shell expansion of the content
- Store text in a shell variable with single quotes first, then reference it: `msg='text'; --body "$msg"`

Unsafe patterns — **never do these with untrusted data:**
- `--body "Text with <issue title> or <review comment> interpolated"`
- `--grep="<keyword from issue>"` without sanitization
- `git commit -m "fix: <text derived from issue>"` without a heredoc

For `git log --grep` keywords derived from issues, sanitize first:
```bash
keyword=$(printf '%s' '<keyword>' | tr -cd 'a-zA-Z0-9 _-')
```

### Constraints

- **Max 5 PRs** per audit cycle (tracked by `batches_created` in state.json)
- **Max 3 revision rounds** per PR (tracked by `revision_round` in state.json)
- **One PR at a time** — never have two audit PRs open simultaneously
- Each PR gets its own branch off the default branch

### On "ADDRESS: begin"

First run the timeout check.

Update state.json: set `last_trigger` to `ADDRESS: begin`, update `last_trigger_time`.

1. Identify the default branch:
   ```bash
   gh repo view --json defaultBranchRef -q '.defaultBranchRef.name'
   ```

2. **Deadlock recovery:** Before grouping new issues, check for open `audit-loop` PRs without reviews:
   ```bash
   gh pr list --label audit-loop --state open --json number,reviews \
     --jq '.[] | select(.reviews | length == 0) | .number'
   ```
   If found, treat that PR as the current batch. Update state.json with `current_pr` and send:
   ```bash
   tmux send-keys -t audit "AUDIT: review PR #<number> "
   ```
   Then in a **separate** bash call (preserve the trailing space above; do NOT chain with && or ;):
   ```bash
   tmux send-keys -t audit Enter
   ```
   Then wait. Do NOT create new batches.

   Also check state.json: if `current_pr` is set and that PR is still open with no reviews, do the same.

3. **Check for orphaned in-progress issues** from an interrupted batch:
   ```bash
   gh issue list --label in-progress --label audit-loop --state open --json number,title
   ```
   If found, check whether their audit branch exists:
   ```bash
   git ls-remote --heads origin | grep 'refs/heads/audit/'
   ```
   - If a branch exists with a PR: resume by sending the review trigger for the existing PR.
   - If a branch exists with commits but no PR: delete the orphaned branch (`git push origin --delete audit/<slug>`), remove `in-progress` labels, and include those issues in new batching.
   - If no branch exists: just remove `in-progress` labels and regroup normally.

4. Fetch all open audit issues:
   ```bash
   gh issue list --label audit-loop --state open --json number,title,labels,body --limit 100
   ```

4. If no issues found, signal completion immediately (see "Signal completion" below).

5. **Validate each issue before planning work.** For each issue, check:
   - Is the scope clear enough to implement? (not vague like "improve performance")
   - Are the acceptance criteria concrete and verifiable?
   - Does it conflict with or duplicate another open issue?
   - Are the referenced files/APIs real? (quick check — `ls` or `grep`)

   If an issue fails validation, comment explaining what's unclear and remove the `audit-loop` label so it drops out of the batch:
   ```bash
   gh issue comment <number> --body "$(cat <<'EOF'
   Issue needs clarification before work can begin: <what's unclear>
   EOF
   )"
   gh issue edit <number> --remove-label "audit-loop"
   ```
   Continue with the remaining valid issues.

6. Group valid issues into batches by relatedness:
   - Same file or subsystem/directory → same batch
   - Same category (e.g., all `cat:security` input validation) → same batch
   - Unrelated issues → separate batches
   - Order batches by severity (critical/high first)
   - Target 2-5 issues per batch. Single-issue batches are fine for complex fixes.

7. Print the batch plan:
   ```
   ## Batch Plan
   Batch 1: #12, #13, #15 (auth input validation)
   Batch 2: #14, #17 (cache race conditions)
   Batch 3: #16 (deprecated dependency)
   ```

8. Fix batch 1 (see "Fix a batch" below).

### On "ADDRESS: continue"

First run the timeout check.

Update state.json: set `last_trigger` to `ADDRESS: continue`, `revision_round` to 0, update `last_trigger_time`.

1. Merge the current PR if one is open and approved. **Refuse to merge PRs labeled `tests-failing`** — if the label is present, print a warning and skip the merge (label `needs-human` instead).

   ```bash
   # Check for tests-failing label
   gh pr view <number> --json labels --jq '.labels[].name' | grep -q 'tests-failing'
   ```

   If no blocking label:
   ```bash
   gh pr merge <number> --squash --delete-branch
   ```
   If merge fails due to conflicts, rebase:
   ```bash
   git fetch origin && git rebase origin/<default-branch>
   git push --force-with-lease
   ```
   Then retry merge. If still failing, label `needs-human` and move on.

2. Close the issues fixed by this PR (if using `Closes #N` in commit, they auto-close on merge — verify):
   ```bash
   gh issue close <number> --reason completed
   ```

3. Check the batch cap — read `batches_created` from state.json. If >= 5, signal completion.

4. Check for remaining open issues:
   ```bash
   gh issue list --label audit-loop --state open --json number,title,labels,body --limit 100
   ```
   If none remain, signal completion. Otherwise, fix the next batch.

### On "ADDRESS: revise PR #N"

First run the timeout check.

Update state.json: set `last_trigger` to `ADDRESS: revise PR #N`, update `last_trigger_time`.

1. Read `revision_round` from state.json. If >= 3, do NOT revise. Instead:
   ```bash
   gh pr edit N --add-label "needs-human"
   ```
   Print: "3 revision rounds reached. Labeled `needs-human`, waiting for next batch."
   Update state.json: set `current_pr` to null, `revision_round` to 0.
   Then wait for `ADDRESS: continue`.

2. Read review comments:
   ```bash
   gh pr view N --json reviews --jq '.reviews[-1].body'
   gh api repos/{owner}/{repo}/pulls/N/comments --jq '.[] | select(.pull_request_review_id) | {path, body, line}'
   ```

3. Check out the PR branch:
   ```bash
   gh pr checkout N
   ```

4. **Parse review comments into explicit tasks before editing.**

   For each review comment, write:
   ```
   File: <path:line>
   Requested change: <one sentence, in your own words>
   My plan: <what you'll do>
   ```

   If you cannot confidently interpret a comment (ambiguous feedback), do NOT guess. Post a clarifying comment on the PR:
   ```bash
   gh pr comment N --body "$(cat <<'CLAREOF'
   Clarification needed: [quote the ambiguous feedback]. Do you mean [interpretation A] or [interpretation B]?
   CLAREOF
   )"
   ```
   Set `awaiting_clarification` to `true` in state.json (keep `phase` as `reviewing`). Do not make any code changes. Still send the review trigger so the audit agent can answer:
   ```bash
   tmux send-keys -t audit "AUDIT: review PR #N "
   ```
   Then in a **separate** bash call (preserve the trailing space above; do NOT chain with && or ;):
   ```bash
   tmux send-keys -t audit Enter
   ```
   Then stop and wait.

   **Resuming from clarification:** On any input while `awaiting_clarification` is true, check the PR for new comments after yours:
   ```bash
   gh pr view N --json comments --jq '.comments[-1].body'
   ```
   If a response exists, set `awaiting_clarification` to `false` and proceed with the revision using the clarified feedback.

   **Scope constraint:** Only modify files and lines cited in the review comments. If you notice an unrelated issue during revision, file a new GitHub issue — do not fix it in this PR.

5. Address each parsed review task with targeted fixes.

6. **Discover and run the project's linter/formatter** (same discovery as "Fix a batch" step 5). Fix auto-fixable errors before committing.

7. If tests/lint fail:
   ```bash
   gh pr edit N --add-label "tests-failing"
   ```
   Note the failure in the PR body. Attempt to fix the test failure once. If unable to fix, proceed — the audit agent will see the label and request changes.

   If tests pass and the PR previously had `tests-failing`:
   ```bash
   gh pr edit N --remove-label "tests-failing"
   ```

8. Update the revision marker in the PR body. Read current body, find or insert:
   ```
   <!-- audit-revision: N -->
   ```
   Update N to the current `revision_round` from state.json.

9. **Stage and commit.** Apply the same secret file exclusion as "Fix a batch" step 7 — never stage `.env`, credentials, keys, etc. Use explicit file paths.
   ```bash
   git add <specific files> && git commit -m "address review feedback (round <revision_round>) on PR #N"
   git push
   ```

10. **Post a revision summary comment on the PR** so the audit agent (and humans) can see the reasoning:
    ```bash
    gh pr comment N --body "$(cat <<'REVEOF'
    ## Revision Round <revision_round>

    ### Analysis
    <copy the parsed review tasks from step 4 here — File, Requested change, My plan for each>

    ### Changes
    <brief description of what was changed and why>

    ### Test plan
    <how to verify the revision fixes the review feedback>
    REVEOF
    )"
    ```

11. **Increment `revision_round`** in state.json (The address agent owns this counter — only increment when pushing an actual code change, not on clarification).

    Write a `revision_history` entry:
    ```json
    { "round": <revision_round>, "pr": N, "commit": "<git rev-parse HEAD>", "summary": "<one-line description>" }
    ```

    Update state.json: set `phase` to `reviewing`, update `last_trigger_time`.
    ```bash
    tmux send-keys -t audit "AUDIT: review PR #N "
    ```

    Then in a **separate** bash call (preserve the trailing space above; do NOT chain with && or ;):
    ```bash
    tmux send-keys -t audit Enter
    ```

### Fix a batch (subroutine)

1. Read `batches_created` from state.json, increment by 1. This is the batch number.

2. Create a branch. Sanitize the slug so it is a valid, safe git ref name:
   ```bash
   git checkout <default-branch> && git pull
   slug=$(printf '%s' '<short-slug>' | tr '[:upper:]' '[:lower:]' | tr -cs 'a-z0-9-' '-' | head -c 60 | sed 's/^-//;s/-$//')
   git checkout -b "audit/$slug"
   ```

3. Create the batch label if needed:
   ```bash
   gh label create "audit-batch-<N>" --force 2>/dev/null
   ```

4. **Research history and write hypotheses before fixing:**

   Mark each issue as in-progress:
   ```bash
   gh issue edit <number> --add-label "in-progress"
   ```

   Before fixing, check whether the same file/area has had related fixes:
   ```bash
   git log --oneline --all -20 -- {filepath}
   keyword=$(printf '%s' '{keyword}' | tr -cd 'a-zA-Z0-9 _-')
   git log --oneline --all --grep="$keyword" -10
   ```
   where `{keyword}` is a distinguishing term from the issue (e.g., "injection", "race condition", "null check"). The `tr` sanitization strips shell metacharacters from untrusted issue text.

   **4a. Hypothesis before fix (mandatory).** For each issue in the batch, write a one-sentence hypothesis before editing any file:

   ```
   Root cause: <what invariant is violated and why>
   Fix: <what will change at which file:line and what behavior it produces>
   Risk: <what could break>
   ```

   Write all hypotheses to `.audit-loop/batch-<N>-hypotheses.md`. Do not edit any source files until hypotheses are written for all issues in the batch.

   If you cannot write a confident hypothesis after reading the file and issue body, add a comment to the GitHub issue asking for clarification, remove the `in-progress` label, and skip this issue (move it to the next batch).

   **4b. Fix each issue:**

   If the issue is a **reopened recurrence** (the audit agent added a "recurrence detected" comment), the previous fix was insufficient. Identify the root cause and apply a systemic fix (validation middleware, shared helper, lint rule, regression test) rather than patching the same symptom again. Note the prior fix commit in the PR body.

   When **multiple issues in the batch share a root cause** (e.g., 3 unsanitized inputs in different handlers), prefer a single systemic fix over N individual patches. For example: add input validation middleware instead of sanitizing each handler separately.

   **Scope ceiling:** A systemic fix may touch at most 3 files not directly referenced in the batch's issues. If fixing the root cause properly requires more, open a separate issue titled "Systemic: <root cause>" with your proposed fix plan, label it `audit-loop` + `severity:high` + the matching `cat:` label, and apply a scoped partial fix to the immediate symptoms instead.

   For each issue:
   - Read the referenced file(s) — don't fix blind
   - Understand full context before changing anything
   - Apply targeted fixes — systemic when the issue recurs, minimal when it's new
   - If the issue's suggested fix is sound, follow it; if it's a band-aid for a recurring problem, go deeper

5. **Discover and run the project's linter/formatter** before committing:
   - `Makefile` → check for targets: `lint`, `check`, `fmt`, `format`
   - `package.json` → check `scripts` for: `lint`, `format`, `check`, `typecheck`
   - `Cargo.toml` → run `cargo clippy` and `cargo fmt --check`
   - `pyproject.toml` / `setup.cfg` → run `ruff check .` or `black --check .` or `flake8`
   - `.golangci.yml` → run `golangci-lint run`
   - Go files without golangci → run `go vet ./...`

   If a linter reports auto-fixable errors, fix them before committing. If unfixable errors remain, note them in the PR body and add the `tests-failing` label after creating the PR.

6. If tests fail after fixing:
   - Attempt to fix the test failure once
   - If unable, label the PR `tests-failing` after creating it (step 8)

7. **Stage and commit.** Never stage files that likely contain secrets (`.env`, `.env.*`, `credentials.json`, `*.pem`, `*.key`, `*.p12`, `*.pfx`, `id_rsa*`, `*.secret`). If any are among the changed files, exclude them and print a warning. Always use explicit file paths — never `git add -A` or `git add .`.
   ```bash
   git add <specific files>
   git commit -m "$(cat <<'EOF'
   fix: <batch summary>

   Closes #<issue1>, closes #<issue2>
   EOF
   )"
   ```

8. **Update documentation.** If the fix changes behavior, configuration, or APIs, update the relevant docs (README, inline comments, config examples, usage instructions). Don't leave stale docs behind.

9. **Self-review before pushing.** Run `git diff HEAD~1` and verify:
   - Each change addresses the root cause hypothesis, not just the symptom
   - No files were modified that aren't referenced in the batch's issues (exception: shared helpers for a systemic fix, up to the 3-file scope ceiling)
   - No debug output, commented-out code, or TODOs were introduced
   - Documentation affected by the changes was updated

   If the diff contains out-of-scope changes, revert them with `git checkout HEAD~1 -- <file>` and amend the commit.

10. Push and create PR. Sanitize the title to strip shell metacharacters:
   ```bash
   git push -u origin "audit/$slug"
   pr_title=$(printf 'audit: %s' '<batch summary>' | tr -d '`$"' | head -c 70)
   gh pr create \
     --title "$pr_title" \
     --body "$(cat <<'PREOF'
   ## Audit Fixes (Batch <N>)

   <!-- audit-revision: 0 -->

   Addresses:
   - #<issue1>: <title>
   - #<issue2>: <title>

   ## Root Cause Analysis
   <copy the hypotheses from .audit-loop/batch-N-hypotheses.md here — root cause, fix, risk for each issue>

   ## Changes
   <brief description of what was changed and why>

   ## Test plan
   <how to verify the fixes>
   PREOF
   )" \
     --label "audit-loop" --label "audit-batch-<N>"
   ```

   If tests are failing, also add `--label "tests-failing"`.

   **If `gh pr create` fails because a PR already exists** for this branch (crash recovery scenario), retrieve the existing PR instead:
   ```bash
   existing=$(gh pr view --json number -q '.number' 2>/dev/null)
   ```
   Use that PR number. Do not attempt to create a duplicate.

11. Update issue labels — mark all issues in this batch as fix-submitted:
    ```bash
    for issue in <issue_numbers>; do
      gh issue edit "$issue" --remove-label "in-progress" --add-label "fix-submitted"
    done
    ```

12. Update state.json: set `current_pr` to the new PR number, `current_batch` to batch number, `batches_created` to new count, `revision_round` to 0, `revision_history` to `[]`, `phase` to `reviewing`, update `last_trigger_time`.

13. Notify audit agent:
   ```bash
   tmux send-keys -t audit "AUDIT: review PR #<number> "
   ```
   Then in a **separate** bash call (preserve the trailing space above; do NOT chain with && or ;):
   ```bash
   tmux send-keys -t audit Enter
   ```

### Signal completion

Update state.json: set `phase` to `complete`, update `last_trigger_time`.

```bash
tmux send-keys -t audit "AUDIT: complete "
```
Then in a **separate** bash call (preserve the trailing space above; do NOT chain with && or ;):
```bash
tmux send-keys -t audit Enter
```

Print:
```
## Audit Fix Complete
Batches completed: <N>
Issues fixed: <N>
Issues remaining: <N> (open with audit-loop label)
PRs needing human review: <N> (labeled needs-human)
```

### Loop error handling

- If `tmux send-keys -t audit` fails: stop, tell the user to start the audit agent in a tmux session named `audit`.
- If `gh` commands fail: retry once, then stop with error.
- If merge conflicts can't be resolved: label `needs-human`, send `ADDRESS: continue` behavior (move to next batch).
- If tests fail after fixing: label `tests-failing`, note in PR body, proceed with review request (the audit agent will catch it).

---

## Manual Protocol

For direct use without the audit loop. User pastes findings or provides a file path.

### Input Handling

1. **File path as argument** (`/address path/to/findings.md`) — read the file
2. **No argument / inline** (`/address`) — the user's message contains the findings. If empty, ask once: "Paste or describe the audit findings."
3. **Directory** (`/address dir/`) — glob for `*.md` or `*.txt`

Parse findings into a structured list. Each finding should capture (when available):
- **ID** — original identifier or generate sequential (F-1, F-2, ...)
- **Severity** — critical / high / medium / low / info
- **Category** — security, correctness, performance, maintainability, style, dependency
- **File + location** — file path and line range
- **Description** — what the issue is
- **Suggestion** — what the audit tool recommended (if any)

### Step 1: Parse & Inventory

Print a compact table:

```
## Audit Inventory ({n} findings)

| #   | Sev    | Cat          | File              | Summary                |
|-----|--------|--------------|-------------------|------------------------|
| F-1 | high   | security     | src/auth.go:42    | SQL injection in login |
| F-2 | medium | correctness  | lib/parse.rs:118  | Off-by-one in boundary |
```

If more than 15 findings, group by category first.

### Step 2: Cross-Reference History

For each finding, check if it's new or recurring:

1. **Git log** — check file history and grep for related keywords:
   ```
   git log --oneline --all -20 -- {filepath}
   git log --oneline --all --grep="{keyword}" -10
   ```

2. **Git blame** — check age of flagged lines:
   ```
   git blame -L {start},{end} {filepath}
   ```

3. **Memory check** — if your agent has a persistent memory/notes system, scan it for past audit references.

4. Classify: **New**, **Recurring** (note prior commit), **Known** (documented tech debt), **Regression** (was fixed, reappeared — flag prominently).

### Step 3: Triage

```
## Triage

### Fix now ({n})
- F-1: SQL injection in login (high/security, NEW)

### Needs discussion ({n})
- F-3: Deprecated dependency (major version bump required)

### Skip ({n})
- F-7: Style nit (no functional impact)
```

**Auto-classify:**
- Fix now: severity >= high, OR regression, OR security >= medium
- Needs discussion: requires architectural change or major dependency update
- Skip: info-level, pure style nits

Ask: **"Proceed with 'Fix now' items? Any from 'Needs discussion' to include or exclude?"**

If the user indicated blanket approval, proceed without waiting.

### Step 4: Fix

1. Read relevant file(s) for full context
2. For **recurring or regression** findings, identify the root cause — apply a systemic fix (shared helper, middleware, lint rule, regression test) rather than patching the same symptom. Note the prior fix commit in the report.
3. When **multiple findings share a root cause**, prefer a single systemic fix over N individual patches
4. For new, isolated findings, apply minimal, targeted fixes via Edit
5. Follow audit suggestion if sound; otherwise use judgment and note divergence
6. Run test/lint commands if discoverable

### Step 5: Report

```
## Audit Resolution

**Source:** {tool name + date}
**Findings:** {total} | Fixed: {n} | Skipped: {n} | Deferred: {n}

### Fixed
| #   | File            | What changed                        | Status     |
|-----|-----------------|-------------------------------------|------------|
| F-1 | src/auth.go:42  | Parameterized query                 | New        |
| F-5 | lib/cache.rs:30 | Added mutex guard                   | Regression |

### Regressions detected
- F-5: previously fixed in `abc1234`, reintroduced in `def5678`. Consider adding a test.

### Recurring patterns
{Root cause analysis if multiple findings share a pattern.}

### Deferred
- F-3: Deprecated dependency — requires major version bump. Schedule separately.

### Skipped
- F-7: Style nit — no functional impact
```

### Step 6: Memory (optional)

If the audit revealed a recurring pattern or systemic issue, offer to save to memory. Don't save routine findings.
