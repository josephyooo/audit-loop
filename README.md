# audit-loop

A pair of skills that orchestrate an automated audit-fix loop between OpenAI Codex and Claude Code via tmux, using GitHub issues and PRs as the shared state store.

## Skills

| Skill | Agent | Location | Purpose |
|---|---|---|---|
| `/audit` | Codex | `codex/audit/SKILL.md` | Audit the repo, open issues, review Claude's PRs |
| `/address` | Claude Code | `claude/address/SKILL.md` | Fix issues in batched PRs, handle review feedback |

## Setup

1. Symlink the skills into the agent skill directories:
   ```bash
   # Claude Code
   ln -s /path/to/audit-loop/claude/address ~/.claude/skills/address

   # Codex
   ln -s /path/to/audit-loop/codex/audit ~/.agents/skills/audit
   ```

2. Start named tmux sessions:
   ```bash
   tmux new -s codex
   tmux new -s claude
   ```

3. In both sessions, `cd` to the target repo.

4. Start Codex in the `codex` session, Claude Code in the `claude` session.

5. Ensure `gh` CLI is authenticated with repo access.

## Usage

In the Codex session, run `/audit`. Everything else is automatic.

### What happens

1. Codex audits the repo and opens GitHub issues (labeled `codex-audit` + severity + category)
2. Codex sends `/address ADDRESS: begin` to Claude via tmux
3. Claude groups related issues into batches, writes root cause hypotheses, fixes batch 1, opens a PR
4. Claude sends `AUDIT: review PR #N` to Codex
5. Codex reviews the PR:
   - **Approve** -> Claude merges and starts the next batch
   - **Request changes** (structured `WRONG|INCOMPLETE|REGRESSION` feedback) -> Claude revises
6. Repeat until all issues are addressed or safety caps are hit
7. Codex does a final sweep and prints a summary

### Manual mode

`/address` also works standalone (without the Codex loop). Paste findings inline or provide a file path:
```
/address path/to/findings.md
/address  (then paste findings)
```

## Trigger Protocol

Codex -> Claude (telling Claude to address):
```
/address ADDRESS: begin      # first trigger, loads the skill
ADDRESS: continue             # PR approved, merge and do next batch
ADDRESS: revise PR #N         # changes requested, revise
```

Claude -> Codex (telling Codex to review):
```
AUDIT: review PR #N           # PR ready for review
AUDIT: complete                # all batches done
```

**Important:** The text and `Enter` must be sent as separate bash calls. Chaining with `&&` or `;` causes a newline instead of submit.

## Safety Caps

- **5 PRs max** per audit cycle
- **3 revision rounds max** per PR
- **30-minute timeout** for normal operations
- **2-hour timeout** for clarification requests
- Exceeded caps -> `needs-human` label, move on

## State

Loop state is stored in `{repo}/.audit-loop/state.json` (gitignored). This enables crash recovery — both agents check state.json on session restart to resume where they left off.

Hypotheses are written to `{repo}/.audit-loop/batch-N-hypotheses.md` before any code is edited, and copied into the PR description.

## Labels

### Static (set at issue creation)
- `codex-audit`, `severity:{critical,high,medium,low}`, `cat:{security,correctness,performance,maintainability,dependency}`

### State transitions (IssueOps FSM)
- `in-progress` -> `fix-submitted` -> `fix-verified` -> closed

### Control
- `needs-human`, `tests-failing`, `audit-batch-{N}`

## Anti-Thrashing

- **Hypothesis gate**: Claude must articulate root cause, fix, and risk before editing
- **Structured reviews**: Codex uses `WRONG|INCOMPLETE|REGRESSION` classifiers with specific file/behavior/change fields
- **Nitpick guard**: Codex approves if functionally correct, no style-only rejections
- **Convergence detection**: At revision round 2, if diffs are cancelling out, escalate instead of looping
- **Clarification flow**: Claude asks instead of guessing on ambiguous feedback
- **Scope ceiling**: Systemic fixes limited to 3 extra files; larger fixes get their own issue
- **Duplicate detection**: Codex reopens prior issues with escalated severity instead of filing duplicates
