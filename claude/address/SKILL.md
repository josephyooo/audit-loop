---
name: address
description: Address external audit findings. Works in two modes — (1) manual: paste findings or point to a file, (2) loop: respond to Codex triggers (ADDRESS: begin/continue/revise) to fix GitHub issues in batched PRs with automated review.
argument-hint: "[filepath | ADDRESS: begin | ADDRESS: continue | ADDRESS: revise PR #N]"
---

# Address Audit Findings

Fix code issues identified by an external audit. Operates in two modes:

- **Manual mode**: user pastes findings or provides a file path. Parse, triage, fix, report.
- **Loop mode**: activated by `ADDRESS:` trigger phrases from Codex via tmux. Reads GitHub issues, fixes in batched PRs, coordinates review with Codex.

---

## Mode Detection

- If input starts with `ADDRESS:` → **loop mode** (see Loop Protocol below)
- Otherwise → **manual mode** (see Manual Protocol below)

---

## Loop Protocol

Triggered by Codex via tmux send-keys. All GitHub operations use `gh` CLI.

### Receive Protocol

**On any input**, check whether it matches a known trigger phrase (`ADDRESS: begin`, `ADDRESS: continue`, `ADDRESS: revise PR #N`). If it does, handle it per the protocol below.

**On session start or after any interruption**, check `.audit-loop/state.json`. If a cycle is in progress and `phase` is `claude-fixing`, resume:
- If `current_pr` is set, check whether that PR has been reviewed. If not, re-send `AUDIT: review PR #N` to Codex. If it has reviews, process the latest review as if receiving the appropriate trigger.
- If `current_pr` is null, treat it as `ADDRESS: begin` (start/continue grouping issues).

**Important:** `needs-human` on one PR does NOT halt the cycle. The agent proceeds to the next batch. Only the capped PR is deferred.

### State File

All loop state lives in `.audit-loop/state.json` (in the repo root, created by Codex's `/audit` skill). This file is the source of truth for cycle progress, batch counts, and revision rounds.

After every action, update `last_trigger_time` to the current ISO 8601 timestamp.

### Timeout Check

On receiving any trigger, read `last_trigger_time` from state.json. If more than 30 minutes have elapsed:

```bash
for pr in $(gh pr list --label codex-audit --state open --json number -q '.[].number'); do
  gh pr edit "$pr" --add-label "needs-human"
done
```

Print: "WARNING: {elapsed} since last activity. Other agent may be stalled. Labeled open PRs `needs-human` and stopping."

Update state.json: set `phase` to `complete`. Do NOT send a tmux trigger (the other agent may be dead). Stop.

### Trigger phrases

- `ADDRESS: begin` — Read all open `codex-audit` issues, plan batches, fix batch 1
- `ADDRESS: continue` — Current PR was approved (or escalated). Merge it, move to next batch.
- `ADDRESS: revise PR #N` — Codex requested changes. Read review comments and revise.

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

2. **Deadlock recovery:** Before grouping new issues, check for open `codex-audit` PRs without reviews:
   ```bash
   gh pr list --label codex-audit --state open --json number,reviews \
     --jq '.[] | select(.reviews | length == 0) | .number'
   ```
   If found, treat that PR as the current batch. Update state.json with `current_pr` and send:
   ```bash
   tmux send-keys -t codex "AUDIT: review PR #<number>" Enter
   ```
   Then wait. Do NOT create new batches.

   Also check state.json: if `current_pr` is set and that PR is still open with no reviews, do the same.

3. Fetch all open audit issues:
   ```bash
   gh issue list --label codex-audit --state open --json number,title,labels,body --limit 100
   ```

4. If no issues found, signal completion immediately (see "Signal completion" below).

5. Group issues into batches by relatedness:
   - Same file or subsystem/directory → same batch
   - Same category (e.g., all `cat:security` input validation) → same batch
   - Unrelated issues → separate batches
   - Order batches by severity (critical/high first)
   - Target 2-5 issues per batch. Single-issue batches are fine for complex fixes.

6. Print the batch plan:
   ```
   ## Batch Plan
   Batch 1: #12, #13, #15 (auth input validation)
   Batch 2: #14, #17 (cache race conditions)
   Batch 3: #16 (deprecated dependency)
   ```

7. Fix batch 1 (see "Fix a batch" below).

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
   gh issue list --label codex-audit --state open --json number,title,labels,body --limit 100
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

4. Address each review comment with targeted fixes.

5. **Discover and run the project's linter/formatter** (same discovery as "Fix a batch" step 5). Fix auto-fixable errors before committing.

6. If tests/lint fail:
   ```bash
   gh pr edit N --add-label "tests-failing"
   ```
   Note the failure in the PR body. Attempt to fix the test failure once. If unable to fix, proceed — Codex will see the label and request changes.

   If tests pass and the PR previously had `tests-failing`:
   ```bash
   gh pr edit N --remove-label "tests-failing"
   ```

7. Update the revision marker in the PR body. Read current body, find or insert:
   ```
   <!-- audit-revision: N -->
   ```
   Update N to the current `revision_round` from state.json.

8. **Stage and commit.** Apply the same secret file exclusion as "Fix a batch" step 7 — never stage `.env`, credentials, keys, etc. Use explicit file paths.
   ```bash
   git add <specific files> && git commit -m "address review feedback (round <revision_round>) on PR #N"
   git push
   ```
   Update state.json: set `phase` to `codex-reviewing`, update `last_trigger_time`.
   ```bash
   tmux send-keys -t codex "AUDIT: review PR #N" Enter
   ```

### Fix a batch (subroutine)

1. Read `batches_created` from state.json, increment by 1. This is the batch number.

2. Create a branch:
   ```bash
   git checkout <default-branch> && git pull
   git checkout -b audit/<short-slug>
   ```

3. Create the batch label if needed:
   ```bash
   gh label create "audit-batch-<N>" --force 2>/dev/null
   ```

4. Mark each issue as in-progress, then fix it:
   ```bash
   gh issue edit <number> --add-label "in-progress"
   ```
   - Read the referenced file(s) — don't fix blind
   - Understand full context before changing anything
   - Apply minimal, targeted fixes
   - If the issue's suggested fix is sound, follow it; otherwise use judgment

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
   git commit -m "fix: <batch summary>

   Closes #<issue1>, closes #<issue2>"
   ```

8. Push and create PR:
   ```bash
   git push -u origin audit/<short-slug>
   gh pr create \
     --title "audit: <batch summary>" \
     --body "## Audit Fixes (Batch <N>)

   <!-- audit-revision: 0 -->

   Addresses:
   - #<issue1>: <title>
   - #<issue2>: <title>

   ## Changes
   <brief description of what was changed and why>

   ## Test plan
   <how to verify the fixes>" \
     --label "codex-audit" --label "audit-batch-<N>"
   ```

   If tests are failing, also add `--label "tests-failing"`.

   **If `gh pr create` fails because a PR already exists** for this branch (crash recovery scenario), retrieve the existing PR instead:
   ```bash
   existing=$(gh pr view --json number -q '.number' 2>/dev/null)
   ```
   Use that PR number. Do not attempt to create a duplicate.

9. Update issue labels — mark all issues in this batch as fix-submitted:
   ```bash
   for issue in <issue_numbers>; do
     gh issue edit "$issue" --remove-label "in-progress" --add-label "fix-submitted"
   done
   ```

10. Update state.json: set `current_pr` to the new PR number, `current_batch` to batch number, `batches_created` to new count, `revision_round` to 0, `phase` to `codex-reviewing`, update `last_trigger_time`.

11. Notify Codex:
   ```bash
   tmux send-keys -t codex "AUDIT: review PR #<number>" Enter
   ```

### Signal completion

Update state.json: set `phase` to `complete`, update `last_trigger_time`.

```bash
tmux send-keys -t codex "AUDIT: complete" Enter
```

Print:
```
## Audit Fix Complete
Batches completed: <N>
Issues fixed: <N>
Issues remaining: <N> (open with codex-audit label)
PRs needing human review: <N> (labeled needs-human)
```

### Loop error handling

- If `tmux send-keys -t codex` fails: stop, tell the user to start Codex in a tmux session named `codex`.
- If `gh` commands fail: retry once, then stop with error.
- If merge conflicts can't be resolved: label `needs-human`, send `ADDRESS: continue` behavior (move to next batch).
- If tests fail after fixing: label `tests-failing`, note in PR body, proceed with review request (Codex will catch it).

---

## Manual Protocol

For direct use without the Codex loop. User pastes findings or provides a file path.

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

3. **Memory check** — scan `~/.claude/projects/*/memory/MEMORY.md` for past audit references.

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
2. Apply minimal, targeted fixes via Edit
3. Follow audit suggestion if sound; otherwise use judgment and note divergence
4. For recurring/regressions, note systemic cause in report (don't over-engineer prevention unless asked)
5. Run test/lint commands if discoverable

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
