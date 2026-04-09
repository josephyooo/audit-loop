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

2. **Classify each step** as one of:
   - **`change`** — edit, add, or remove code/docs. The default.
   - **`execute`** — run a script, submit a job, trigger a pipeline, rerun a process. The issue must produce artifacts, not code changes.

   If a step involves both (e.g. "write a script and run it"), split it into two issues with a dependency link.

3. **Research before filing.** For each step — especially `execute` steps — verify the operational details against the actual repo before writing the issue:
   - Read project docs (README, AGENTS.md, Makefile, CI configs) to find the exact commands, scripts, and parameters.
   - Check that referenced files exist (`ls`, `grep`).
   - For execution steps, identify the entry-point script/command and any required env vars, flags, or input files.

   If you cannot determine the concrete command or files for a step after researching, the step is not ready to be an issue. Either decompose it further, or ask the user for the missing detail before filing.

4. For each step, derive the labels before issue creation:
   - `severity_label`: preserve an explicit plan hint if it matches one of `severity:critical`, `severity:high`, `severity:medium`, or `severity:low`; otherwise default to `severity:medium`
   - `category_label`: preserve an explicit plan hint if it matches one of `cat:security`, `cat:correctness`, `cat:performance`, `cat:maintainability`, or `cat:dependency`
   - If no category hint is present, infer it from the step content using a fixed mapping:
     - auth, permission, secret, sanitize, validation, injection -> `cat:security`
     - bug, edge case, test, regression, null, panic, race -> `cat:correctness`
     - perf, latency, cache, throughput, query hot path -> `cat:performance`
     - dependency, upgrade, version, package, library -> `cat:dependency`
     - otherwise -> `cat:maintainability`
   - Only emit labels from the supported set above; do not invent new `severity:*` or `cat:*` labels at issue-creation time

5. For each step, create an issue using the appropriate template:

   **For `change` issues:**
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
   <file paths that need to be changed — REQUIRED, never omit>

   ## Acceptance Criteria
   - <verifiable by a shell command: file exists, test passes, grep matches, script exits 0>
   - <verifiable by a shell command>
   EOF
   )"
   ```

   **For `execute` issues:**
   ```bash
   gh issue create \
     --title "<concise step title>" \
     --label "audit-loop" \
     --label "<severity_label>" \
     --label "<category_label>" \
     --body "$(cat <<'EOF'
   ## Context
   <why this step exists and how it fits into the broader plan>

   ## Command
   <exact command(s) to run — script path, env vars, flags, arguments>
   <if a job scheduler is needed: the submission command (e.g. sbatch, qsub)>
   <expected runtime estimate if nontrivial>

   ## Files
   <script(s) being executed and expected output artifact paths — REQUIRED>

   ## Acceptance Criteria
   - <verifiable by a shell command: output file exists, row count matches, script exits 0>
   - <verifiable by a shell command>
   EOF
   )"
   ```

6. If a step depends on another step, note it in the body: `**Depends on:** #<number>`.

7. Print a summary table when done:
   ```
   ## Issues Created

   | # | Title | Type | Severity | Category | Depends on |
   |---|-------|------|----------|----------|------------|
   | 1 | ...   | change | medium | correctness | —       |
   | 2 | ...   | execute | high | security | #1       |
   ```
