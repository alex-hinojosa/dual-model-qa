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
Primary AI (builds/fixes) → git diff → ~~reviewer-cli (QA review) → logs + notification
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
Append: "When finished, commit your changes with a descriptive message. Do NOT deploy unless explicitly instructed."

This ensures the autonomous session:
- Picks up project conventions
- Commits work (so the QA diff can be computed)
- Does not deploy unreviewed code

**Success criteria**: Prompt enhanced with safety bookends.

### 3. Launch the Session
Execute the autonomous session using the project's launcher script:

```bash
./launcher.sh <project> '<enhanced-prompt>' --model sonnet
```

The launcher handles:
1. Recording git HEAD before the session
2. Running the primary AI with auto-approval
3. Computing the git diff after completion
4. Piping the diff to ~~reviewer-cli with a structured QA prompt
5. Logging both the session output and QA findings
6. Sending a desktop notification on completion

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
1. **Session log** — full output of what the primary AI did
2. **QA log** — the reviewer's severity-tagged findings
3. **Git log** — commits made during the session

Priority order for reviewing QA findings:
1. Any **Critical Issues** — address before merging
2. **Warnings** — address soon or document why they're acceptable
3. **Suggestions** — nice to have, batch for later

## Rules
- Always enhance prompts with safety context (read config, don't deploy)
- Default to sonnet unless the user specifies opus
- QA is ON by default — only skip with explicit --no-qa
- Never deploy from an overnight run — that's a separate human decision
- Log everything — session output + QA findings + timestamps
- Send a notification on completion so the user knows it's done
