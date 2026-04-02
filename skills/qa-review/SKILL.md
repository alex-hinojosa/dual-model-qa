---
name: qa-review
description: >
  This skill should be used when the user asks to "run QA", "get a second opinion",
  "review with another model", "cross-model review", "critique this code",
  "dual-model review", or wants an independent AI code review of recent changes.
  Dispatches code to a second AI model for severity-tagged findings.
version: 0.1.0
---

# QA Review

Dispatch code changes to ~~reviewer-cli for an independent, severity-tagged code review.

## Inputs
- `$target`: What to review — "recent changes", a file path, "last commit", or "staged changes"

## Goal
Get an independent QA review from a second AI model and present structured findings.

## Steps

### 1. Determine Review Scope
Identify what code to send for review:

- **"recent changes" or "last commit"**: `git diff HEAD~1`
- **"staged changes"**: `git diff --cached`
- **Specific file**: `git diff HEAD -- <file>` or full file contents
- **Commit range**: `git diff <from>..<to>`
- **PR or branch**: `git diff main...HEAD`

Capture the diff output.

**Success criteria**: Have a concrete diff or file contents to review.

### 2. Build the QA Prompt
Construct the reviewer prompt using the Dual-AI CQ severity framework:

Prepend context:
- Project name (from directory or CLAUDE.md)
- What task produced these changes (from git log or user input)

Include the severity tiers:
1. CRITICAL: Security, breaking changes, data corruption
2. WARNING: Code quality, test gaps, edge cases
3. SUGGESTION: Readability, performance, naming

Append the diff.

**Success criteria**: Structured prompt ready to pipe to reviewer.

### 3. Dispatch to Reviewer
Write the prompt to a temp file and pipe to ~~reviewer-cli:

```bash
echo "$QA_PROMPT" | ~~reviewer-cli 2>&1 | tee /tmp/qa-review-output.log
```

**Success criteria**: Reviewer completes and output is captured.

### 4. Present Findings
Read the reviewer output and present to the user organized by severity:

- **Critical Issues** — highlight these prominently, recommend immediate action
- **Warnings** — list with context on why they matter
- **Suggestions** — summarize briefly
- **Overall Assessment** — the reviewer's ship/no-ship verdict

If there are Critical findings, recommend the user address them before merging.

**Success criteria**: User has a clear, actionable summary of the review.

### 5. Optional: Route Fixes
If the user wants to fix the findings:
- Create a task list from Critical and Warning items
- Address each finding, re-running the affected tests
- Optionally re-run QA to confirm fixes

**Success criteria**: All Critical items resolved or documented.

## Rules
- Always use the structured severity framework — never free-form reviews
- Include file paths and line references when available
- Never modify code based solely on the reviewer's suggestions without user approval
- The reviewer's output is advisory — the human gate decides
- If the diff is too large (>10,000 lines), split into logical chunks
