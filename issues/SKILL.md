---
name: issues
description: "Create GitHub issues from a plan. Each step becomes a self-contained issue with context, files, and acceptance criteria. Labels every issue audit-loop."
argument-hint: "[plan text or file path]"
---

# Create Issues from Plan

Turn a plan into GitHub issues so the `/address` agent (or a human) can pick them up.

## Input

- If an argument is provided and it's a file path, read it
- If an argument is provided and it's text, use it as the plan
- If no argument, check whether a plan exists in the current conversation context (e.g. from plan mode). If not, ask: "Paste or describe the plan."

## Steps

1. Parse the plan into discrete steps. Each step becomes one issue.

2. For each step, create an issue:
   ```bash
   gh issue create \
     --title "<concise step title>" \
     --label "audit-loop" \
     --body "$(cat <<'EOF'
   ## Context
   <why this step exists and how it fits into the broader plan>

   ## Files
   <which files are likely involved, if known>

   ## Acceptance Criteria
   - <concrete, verifiable condition>
   - <concrete, verifiable condition>
   EOF
   )"
   ```

3. If a step depends on another step, note it in the body: `**Depends on:** #<number>`.

4. Print a summary table when done:
   ```
   ## Issues Created

   | # | Title | Depends on |
   |---|-------|------------|
   | 1 | ...   | —          |
   | 2 | ...   | #1         |
   ```
