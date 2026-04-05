# audit-loop

> **Note**: This project was built with [Claude Code](https://claude.ai/claude-code). Code, documentation, and commit messages were AI-generated with human direction and review.

Agent-agnostic skills that orchestrate an automated audit-fix loop between any two AI coding agents via tmux, using GitHub issues and PRs as the shared state store.

## Skills

| Skill | Role | Location | Purpose |
|---|---|---|---|
| `/audit` | Auditor | `audit/SKILL.md` | Audit the repo, open issues, review the address agent's PRs |
| `/address` | Fixer | `address/SKILL.md` | Fix issues in batched PRs, handle review feedback |

Works with any agent that supports the Agent Skills standard (Claude Code, Codex CLI, Copilot CLI, Gemini CLI, etc.).

## Setup

Symlink into whichever agent skill directories you use:
```bash
# Any agent that reads ~/.agents/skills/ (Codex CLI, Copilot CLI)
ln -s /path/to/audit-loop/audit ~/.agents/skills/audit
ln -s /path/to/audit-loop/address ~/.agents/skills/address

# Claude Code specifically
ln -s /path/to/audit-loop/address ~/.claude/skills/address
ln -s /path/to/audit-loop/audit ~/.claude/skills/audit
```

### tmux sessions

Start two named tmux sessions — names must be `audit` and `address`:
```bash
tmux new -s audit
tmux new -s address
```

In both sessions, `cd` to the target repo and start your chosen agent.

Ensure `gh` CLI is authenticated with repo access.

## Usage

In the `audit` session, run `/audit`. Everything else is automatic.

### What happens

1. The audit agent audits the repo and opens GitHub issues (labeled `audit-loop` + severity + category)
2. The audit agent sends `/address ADDRESS: begin` to the address agent via tmux
3. The address agent groups related issues into batches, writes root cause hypotheses, fixes batch 1, opens a PR
4. The address agent sends `AUDIT: review PR #N` to the audit agent
5. The audit agent reviews the PR:
   - **Approve** -> address agent merges and starts the next batch
   - **Request changes** (structured `WRONG|INCOMPLETE|REGRESSION` feedback) -> address agent revises
6. Repeat until all issues are addressed or safety caps are hit
7. The audit agent does a final sweep and prints a summary

### Manual mode

`/address` also works standalone (without the audit loop). Paste findings inline or provide a file path:
```
/address path/to/findings.md
/address  (then paste findings)
```

## Trigger Protocol

Audit -> Address (telling the address agent to fix):
```
/address ADDRESS: begin      # first trigger, loads the skill
ADDRESS: continue             # PR approved, merge and do next batch
ADDRESS: revise PR #N         # changes requested, revise
```

Address -> Audit (telling the audit agent to review):
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
- `audit-loop`, `severity:{critical,high,medium,low}`, `cat:{security,correctness,performance,maintainability,dependency}`

### State transitions (IssueOps FSM)
- `in-progress` -> `fix-submitted` -> `fix-verified` -> closed

### Control
- `needs-human`, `tests-failing`, `audit-batch-{N}`

## Anti-Thrashing

- **Hypothesis gate**: Addressing agent must articulate root cause, fix, and risk before editing
- **Structured reviews**: Auditing agent uses `WRONG|INCOMPLETE|REGRESSION` classifiers with specific file/behavior/change fields
- **Nitpick guard**: Auditing agent approves if functionally correct, no style-only rejections
- **Convergence detection**: At revision round 2, if diffs are cancelling out, escalate instead of looping
- **Clarification flow**: Addressing agent asks instead of guessing on ambiguous feedback
- **Scope ceiling**: Systemic fixes limited to 3 extra files; larger fixes get their own issue
- **Duplicate detection**: Auditing agent reopens prior issues with escalated severity instead of filing duplicates
