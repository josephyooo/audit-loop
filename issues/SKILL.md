---
name: issues
description: "Create GitHub issues from a plan. Each step becomes a self-contained issue with context, files, acceptance criteria, and audit-loop severity/category labels."
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

2. For each step, derive the labels before issue creation:
   - `severity_label`: preserve an explicit plan hint if it matches one of `severity:critical`, `severity:high`, `severity:medium`, or `severity:low`; otherwise default to `severity:medium`
   - `category_label`: preserve an explicit plan hint if it matches one of `cat:security`, `cat:correctness`, `cat:performance`, `cat:maintainability`, or `cat:dependency`
   - If no category hint is present, infer it from the step content using a fixed mapping:
     - auth, permission, secret, sanitize, validation, injection -> `cat:security`
     - bug, edge case, test, regression, null, panic, race -> `cat:correctness`
     - perf, latency, cache, throughput, query hot path -> `cat:performance`
     - dependency, upgrade, version, package, library -> `cat:dependency`
     - otherwise -> `cat:maintainability`
   - Only emit labels from the supported set above; do not invent new `severity:*` or `cat:*` labels at issue-creation time

3. For each step, create an issue:
   ```bash
   gh issue create \
     --title "<concise step title>" \
     --label "audit-loop" \
     --label "<severity_label>" \
     --label "<category_label>" \
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

4. If a step depends on another step, note it in the body: `**Depends on:** #<number>`.

5. Print a summary table when done:
   ```
   ## Issues Created

   | # | Title | Severity | Category | Depends on |
   |---|-------|----------|----------|------------|
   | 1 | ...   | medium   | correctness | —       |
   | 2 | ...   | high     | security | #1         |
   ```
