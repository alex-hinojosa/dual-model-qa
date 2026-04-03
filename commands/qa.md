---
description: Run a dual-model QA review on recent changes
allowed-tools: Read, Bash, Grep, Glob, Write
argument-hint: [target — "last commit", "staged", file path, or commit range]
---

Run an independent QA review using the Dual-AI CQ methodology.

First, read the critique-loop skill at `${CLAUDE_PLUGIN_ROOT}/skills/critique-loop/SKILL.md` for the severity framework.

Determine what to review based on the argument:
- No argument or "last commit": `git diff HEAD~1`
- "staged": `git diff --cached`
- A file path: `git diff HEAD -- $1` (or read the full file if not in git)
- A commit range like "abc123..def456": `git diff $1`
- "branch" or "pr": `git diff main...HEAD`

If the diff is empty, tell the user there are no changes to review.

Build the QA prompt using the template at `${CLAUDE_PLUGIN_ROOT}/skills/critique-loop/references/qa-prompt-template.md`. Fill in:
- Project name from the current directory name or CLAUDE.md
- Task description from recent git log messages
- The captured diff

Dispatch to ~~reviewer-cli using a temp file (avoids piping issues, macOS mktemp suffix bugs, and argument length limits):
```bash
QA_PROMPT_FILE=$(mktemp /tmp/qa-review-prompt-XXXXXXXX)
echo "$QA_PROMPT" > "$QA_PROMPT_FILE"
~~reviewer-cli "$(cat "$QA_PROMPT_FILE")" 2>&1 | tee /tmp/qa-review-output.log
rm -f "$QA_PROMPT_FILE"
```

For Gemini CLI: use `gemini -y -p "$(cat ...)"` — the `-y` flag is required for tool access in headless mode.

Present findings to the user organized by severity:
1. **Critical Issues** — highlight prominently
2. **Warnings** — list with context
3. **Suggestions** — summarize briefly
4. **Overall Assessment** — the reviewer's verdict

If there are Critical findings, recommend addressing them before merging.
