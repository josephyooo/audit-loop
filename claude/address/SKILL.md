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

### Trigger phrases

- `ADDRESS: begin` — Read all open `codex-audit` issues, plan batches, fix batch 1
- `ADDRESS: continue` — Current PR was approved (or escalated). Merge it, move to next batch.
- `ADDRESS: revise PR #N` — Codex requested changes. Read review comments and revise.

### Constraints

- **Max 5 PRs** per audit cycle
- **Max 3 revision rounds** per PR
- **One PR at a time** — never have two audit PRs open simultaneously
- Each PR gets its own branch off the default branch

### On "ADDRESS: begin"

1. Identify the default branch:
   ```bash
   gh repo view --json defaultBranchRef -q '.defaultBranchRef.name'
   ```

2. Fetch all open audit issues:
   ```bash
   gh issue list --label codex-audit --state open --json number,title,labels,body --limit 100
   ```

3. If no issues found, signal completion immediately:
   ```bash
   tmux send-keys -t codex "AUDIT: complete" Enter
   ```

4. Group issues into batches by relatedness:
   - Same file or subsystem/directory → same batch
   - Same category (e.g., all `cat:security` input validation) → same batch
   - Unrelated issues → separate batches
   - Order batches by severity (critical/high first)
   - Target 2-5 issues per batch. Single-issue batches are fine for complex fixes.

5. Print the batch plan:
   ```
   ## Batch Plan
   Batch 1: #12, #13, #15 (auth input validation)
   Batch 2: #14, #17 (cache race conditions)
   Batch 3: #16 (deprecated dependency)
   ```

6. Fix batch 1 (see "Fix a batch" below).

### On "ADDRESS: continue"

1. Merge the current PR if one is open and approved:
   ```bash
   gh pr merge <number> --squash --delete-branch
   ```
   If merge fails due to conflicts, rebase:
   ```bash
   git fetch origin && git rebase origin/<default-branch>
   git push --force-with-lease
   ```
   Then retry merge. If still failing, label `needs-human` and move on.

2. Close the issues fixed by this PR:
   ```bash
   gh issue close <number> --reason completed
   ```

3. Check the batch cap — count how many `audit-batch-*` PRs have been created:
   ```bash
   gh pr list --label codex-audit --state all --json labels --limit 100
   ```
   If 5 batches created, signal completion.

4. Check for remaining open issues:
   ```bash
   gh issue list --label codex-audit --state open --json number,title,labels,body --limit 100
   ```
   If none remain, signal completion. Otherwise, fix the next batch.

### On "ADDRESS: revise PR #N"

1. Read review comments:
   ```bash
   gh pr view N --json reviews --jq '.reviews[-1].body'
   gh api repos/{owner}/{repo}/pulls/N/comments --jq '.[] | select(.pull_request_review_id) | {path, body, line}'
   ```

2. Check revision count:
   ```bash
   gh pr view N --json reviews --jq '[.reviews[] | select(.state=="CHANGES_REQUESTED")] | length'
   ```
   If >= 3, do NOT revise. Instead:
   ```bash
   gh pr edit N --add-label "needs-human"
   ```
   Print: "3 revision rounds reached. Labeled `needs-human`, waiting for Codex to advance."
   Then wait for `ADDRESS: continue`.

3. Check out the PR branch:
   ```bash
   gh pr checkout N
   ```

4. Address each review comment with targeted fixes.

5. Commit, push, notify Codex:
   ```bash
   git add <specific files> && git commit -m "address review feedback on PR #N"
   git push
   tmux send-keys -t codex "AUDIT: review PR #N" Enter
   ```

### Fix a batch (subroutine)

1. Determine batch number (count existing `audit-batch-*` labels + 1).

2. Create a branch:
   ```bash
   git checkout <default-branch> && git pull
   git checkout -b audit/<short-slug>
   ```

3. Create the batch label if needed:
   ```bash
   gh label create "audit-batch-<N>" --force 2>/dev/null
   ```

4. Fix each issue in the batch:
   - Read the referenced file(s) — don't fix blind
   - Understand full context before changing anything
   - Apply minimal, targeted fixes
   - If the issue's suggested fix is sound, follow it; otherwise use judgment
   - Run available test/lint commands if discoverable (Makefile, package.json, Cargo.toml, pyproject.toml, etc.)

5. Commit with issue references:
   ```bash
   git add <specific files>
   git commit -m "fix: <batch summary>

   Closes #<issue1>, closes #<issue2>"
   ```

6. Push and create PR:
   ```bash
   git push -u origin audit/<short-slug>
   gh pr create \
     --title "audit: <batch summary>" \
     --body "## Audit Fixes (Batch <N>)

   Addresses:
   - #<issue1>: <title>
   - #<issue2>: <title>

   ## Changes
   <brief description of what was changed and why>

   ## Test plan
   <how to verify the fixes>" \
     --label "codex-audit" --label "audit-batch-<N>"
   ```

7. Notify Codex:
   ```bash
   tmux send-keys -t codex "AUDIT: review PR #<number>" Enter
   ```

### Signal completion

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
- If merge conflicts can't be resolved: label `needs-human`, notify Codex with `AUDIT: complete`, move on.
- If tests fail after fixing: one attempt to fix. If unable, note in PR body and proceed.

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
