---
description: Queue an autonomous overnight run with dual-model QA
allowed-tools: Read, Bash, Write
argument-hint: <project> <prompt> [--model opus|sonnet] [--no-qa]
---

Launch an autonomous overnight coding session with automatic dual-model QA review.

First, read the overnight-run skill at `${CLAUDE_PLUGIN_ROOT}/skills/overnight-run/SKILL.md` for the full pipeline.

Parse the arguments:
- `$1` — project directory path or shortcut name
- Remaining arguments — the prompt describing what to build/fix
- `--model opus|sonnet` — optional model override (default: sonnet)
- `--no-qa` — optional flag to skip the reviewer step

Validate the project directory exists and has git initialized.

Enhance the prompt with safety bookends:
- Prepend: "Read CLAUDE.md and any project context before starting."
- Append: "Commit changes with descriptive messages. Do NOT deploy."

Launch via the project's autonomous runner script. If no launcher exists, explain the setup needed (see `${CLAUDE_PLUGIN_ROOT}/skills/critique-loop/references/implementation-guide.md`).

Report back:
- What was launched (project, model, prompt summary)
- Where logs will be written
- Whether QA is enabled
- That a notification will fire on completion
- Remind the user to check QA findings in the morning
