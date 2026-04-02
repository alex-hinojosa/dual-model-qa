---
name: overnight-run
description: >
  This skill should be used when the user asks to "run overnight", "queue for tonight",
  "autonomous run with QA", "batch run", "run while I sleep",
  "overnight tasks", or wants to launch unattended AI coding sessions
  that automatically get a second-model QA review on completion.
version: 0.1.0
---

# Overnight Run

Launch autonomous AI coding sessions that automatically get an independent QA review from ~~reviewer-cli when they complete. Wake up to both the finished work and a severity-tagged code review.

## Pipeline

```
Phase 1: Primary AI (builds/fixes, does NOT commit)
    ↓
Phase 2: ~~reviewer-cli reviews uncommitted diff → QA_VERDICT: PASS or FAIL
    ↓
Phase 3 (PASS only): Auto-commit + optional deploy → logs + notification
    ↓
Phase 3 (FAIL): Changes stay uncommitted → alert + QA log → human reviews in morning
```

## Inputs
- `$project`: Project directory path or name
- `$prompt`: What the primary AI should do
- `--no-qa` (optional): Skip the reviewer step

## Goal
Launch one or more unattended coding tasks with automatic cross-model QA, so the user wakes up to completed work plus an independent review.

## Steps

### 1. Validate the Project
Confirm the project directory exists and has a git repo initialized.

Read any project-level configuration (CLAUDE.md, rules files) to understand the codebase context.

**Success criteria**: Valid project directory with git confirmed.

### 2. Enhance the Prompt
Wrap the user's prompt with safety context:

Prepend: "Read CLAUDE.md and any project context files before starting."
Append: "Do NOT commit or deploy. Leave all changes uncommitted for QA review."

This ensures the autonomous session:
- Picks up project conventions
- Leaves changes uncommitted (so the QA step reviews a clean diff)
- Does not deploy unreviewed code

**Success criteria**: Prompt enhanced with safety bookends.

### 3. Launch the Session
Execute the autonomous session using the project's launcher script:

```bash
./launcher.sh <project> '<enhanced-prompt>' --model sonnet
```

The launcher handles the three-phase pipeline:
1. **Phase 1 — Build**: Runs the primary AI with auto-approval. Changes are NOT committed.
2. **Phase 2 — QA**: Computes the uncommitted diff (excluding lockfiles, truncated at 2000 lines), sends it to ~~reviewer-cli with a structured QA prompt requesting a `QA_VERDICT: PASS` or `QA_VERDICT: FAIL`.
3. **Phase 3 — Commit Gate**: If QA_VERDICT is PASS, auto-commits with a message derived from the prompt. If the original prompt contained "deploy", also runs deployment. If FAIL, changes stay uncommitted and user gets an alert.
4. Logs both the session output and QA findings.
5. Sends a desktop notification on completion.

**Success criteria**: Session launched and running autonomously.

### 4. Report to User
Confirm what was launched:
- Project name and directory
- Model used (sonnet/opus)
- Whether QA is enabled
- Where to find logs when it completes
- That the QA review will run automatically after the primary AI finishes

**Success criteria**: User knows the pipeline is running and where to check results.

## Multi-Task Runs

For running multiple tasks overnight, create a tasks file:

```
project-a|Fix failing unit tests and commit|sonnet
project-b|Refactor auth middleware for JWT RS256|opus
project-c|Add input validation to all API endpoints|sonnet
```

Tasks run sequentially. Each task gets its own QA review.

## Reading Results

Next morning, check:
1. **QA log** — look for `QA_VERDICT: PASS` or `QA_VERDICT: FAIL` first
2. **Session log** — full output of what the primary AI did
3. **Git log** — if QA passed, the auto-commit will be here with a descriptive message

**If QA passed:** Changes are committed. Review the QA log for any Warnings or Suggestions worth addressing in a follow-up.

**If QA failed:** Changes are uncommitted but present in the working tree. Review the Critical findings in the QA log, fix the issues, then either re-run the pipeline or manually commit.

Priority order for reviewing QA findings:
1. Any **Critical Issues** — must fix (these caused the FAIL)
2. **Warnings** — address soon or document why they're acceptable
3. **Suggestions** — nice to have, batch for later

## Rules
- Always enhance prompts with safety context (read config, don't deploy)
- Default to sonnet unless the user specifies opus
- QA is ON by default — only skip with explicit --no-qa
- Never deploy from an overnight run — that's a separate human decision
- Log everything — session output + QA findings + timestamps
- Send a notification on completion so the user knows it's done
