# Implementation Guide

How to set up the Dual-Model QA pipeline for different reviewer CLIs and environments.

## Prerequisites

- A primary AI coding tool (Claude Code, Cursor, Copilot, etc.)
- A second AI model accessible via CLI that accepts a prompt argument
- Git installed (the pipeline diffs committed changes)

## Option 1: Gemini CLI (Recommended)

Google's Gemini CLI is free, fast, and from a different vendor than Claude — ideal for cross-model review.

### Install
```bash
npm install -g @google/gemini-cli
```

### Authenticate
```bash
gemini auth login
```

### Test
```bash
gemini -y -p "Hello, what model are you?"
```

### Integration
The plugin writes the prompt to a temp file and passes it as a positional argument:
```bash
QA_PROMPT_FILE=$(mktemp /tmp/gemini-qa-prompt-XXXXXXXX)
echo "$QA_PROMPT" > "$QA_PROMPT_FILE"
gemini -y -p "$(cat "$QA_PROMPT_FILE")" 2>&1
rm -f "$QA_PROMPT_FILE"
```

**Important — three gotchas discovered in production:**
1. **Positional argument, not pipe**: Gemini CLI expects the prompt as a positional argument. Piping via `echo | gemini -p` fails with "Not enough arguments following: p".
2. **`-y` flag required for headless mode**: Without `-y` (yolo/auto-approve), Gemini's `-p` mode blocks tools that need approval (like `run_shell_command`). The tool exists in Gemini's system prompt but fails at runtime with "Tool not found" unless `-y` is passed.
3. **macOS mktemp suffix**: `mktemp` on macOS requires the template to end with `X`s only — no `.txt` suffix. `mktemp /tmp/foo-XXXXXX.txt` fails silently or creates a literal `XXXXXX.txt` file, crashing any script with `set -euo pipefail`.

## Option 2: Ollama (Local / Private)

For air-gapped or privacy-sensitive environments. Runs entirely on your machine.

### Install
```bash
brew install ollama
ollama pull codellama:34b  # or any model
```

### Test
```bash
echo "Review this function for bugs" | ollama run codellama:34b
```

### Integration
```bash
echo "$QA_PROMPT" | ollama run codellama:34b 2>&1
```

**Note:** Local models have smaller context windows. For large diffs, the plugin can automatically truncate to changed files only.

## Option 3: Custom API Wrapper

Create a simple shell script that calls any API:

```bash
#!/bin/bash
# ~/bin/review-cli.sh
INPUT=$(cat -)
curl -s https://api.openai.com/v1/chat/completions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"model\": \"gpt-4o\",
    \"messages\": [{\"role\": \"user\", \"content\": $(echo "$INPUT" | jq -Rs .)}]
  }" | jq -r '.choices[0].message.content'
```

Make executable: `chmod +x ~/bin/review-cli.sh`

### Integration
```bash
echo "$QA_PROMPT" | ~/bin/review-cli.sh 2>&1
```

## Launcher Script Setup

The overnight-run skill expects a launcher script. Here is the reference implementation:

### claude-auto.sh

Place at `~/dev/Shared/claude-auto.sh` (or any location on PATH):

```bash
#!/bin/bash
# Usage: claude-auto.sh <project-dir> <prompt> [--model opus|sonnet] [--no-qa]
# Three-phase pipeline: Build → QA → Commit Gate

set -euo pipefail

PROJECT_DIR="$1"
PROMPT="$2"
MODEL="sonnet"
RUN_QA=true

shift 2
while [[ $# -gt 0 ]]; do
  case "$1" in
    --model) MODEL="$2"; shift 2 ;;
    --no-qa) RUN_QA=false; shift ;;
    *) shift ;;
  esac
done

TIMESTAMP=$(date +%Y-%m-%d_%H%M)
LOG_DIR="${PROJECT_DIR}/logs"
mkdir -p "$LOG_DIR"
LOGFILE="$LOG_DIR/session_${TIMESTAMP}.log"
QA_LOGFILE="$LOG_DIR/session_${TIMESTAMP}_qa.log"

cd "$PROJECT_DIR"
GIT_BEFORE=$(git rev-parse HEAD 2>/dev/null || echo "no-git")

# ── Phase 1: Build (do NOT commit or deploy) ──
BUILD_PROMPT="$PROMPT

Do NOT commit or deploy. Leave all changes uncommitted for QA review."

claude --dangerously-skip-permissions --model "$MODEL" -p "$BUILD_PROMPT" 2>&1 | tee -a "$LOGFILE"
CLAUDE_EXIT=${PIPESTATUS[0]}

# ── Phase 2: QA Review ──
QA_PASSED=false
if [ "$RUN_QA" = true ] && [ "$CLAUDE_EXIT" -eq 0 ] && [ "$GIT_BEFORE" != "no-git" ]; then
  # Diff uncommitted changes, excluding lockfiles
  DIFF=$(git diff \
    -- ':!package-lock.json' ':!yarn.lock' ':!pnpm-lock.yaml' ':!*.lock' \
    2>/dev/null || echo "")

  # Truncate very large diffs to avoid context window limits
  DIFF_LINES=$(echo "$DIFF" | wc -l)
  if [ "$DIFF_LINES" -gt 2000 ]; then
    DIFF=$(echo "$DIFF" | head -2000)
    DIFF="$DIFF

... [TRUNCATED: diff was $DIFF_LINES lines, showing first 2000]"
  fi

  if [ -n "$DIFF" ]; then
    QA_PROMPT="You are a senior code reviewer... [see qa-prompt-template.md]
End with QA_VERDICT: PASS or QA_VERDICT: FAIL.
Only FAIL for critical issues (security, breaking changes, data loss).

$DIFF"
    # Write to temp file to avoid piping issues and arg length limits
    QA_PROMPT_FILE=$(mktemp /tmp/gemini-qa-prompt-XXXXXXXX)
    echo "$QA_PROMPT" > "$QA_PROMPT_FILE"
    gemini -y -p "$(cat "$QA_PROMPT_FILE")" 2>&1 | tee -a "$QA_LOGFILE" || true
    rm -f "$QA_PROMPT_FILE"

    # Parse verdict
    QA_VERDICT=$(grep -o 'QA_VERDICT: \(PASS\|FAIL\)' "$QA_LOGFILE" | tail -1 || echo "")
    if [ "$QA_VERDICT" = "QA_VERDICT: PASS" ]; then
      QA_PASSED=true
    fi
  else
    QA_PASSED=true  # No diff = nothing to review
  fi
elif [ "$RUN_QA" = false ]; then
  QA_PASSED=true  # QA skipped by user
fi

# ── Phase 3: Commit Gate ──
if [ "$QA_PASSED" = true ]; then
  # Auto-commit with message derived from prompt
  COMMIT_MSG=$(echo "$PROMPT" | head -c 72)
  git add -A
  git commit -m "$COMMIT_MSG" || true

  # Deploy only if "deploy" appears in the original prompt
  if echo "$PROMPT" | grep -qi "deploy"; then
    gcloud run deploy 2>&1 | tee -a "$LOGFILE" || true
  fi

  osascript -e 'display notification "Build + QA complete. Changes committed." with title "Claude Auto" sound name "Glass"'
else
  osascript -e 'display notification "QA FAILED — changes NOT committed. Check QA log." with title "Claude Auto" sound name "Sosumi"'
fi
```

### Key flags

- `--no-qa` — skip the reviewer step (for docs-only or config changes)
- `--model opus` — use a stronger primary model for complex tasks

### Production hardening tips

- **Three-phase gate**: Builder never commits. Reviewer produces a binary verdict. Commit only happens on PASS.
- **Exclude lockfiles**: The `':!package-lock.json'` pathspec filters out lockfile noise that can be tens of thousands of lines
- **Truncate large diffs**: The 2000-line guard prevents context window overflows in the reviewer model
- **Temp file for prompt**: Avoids both shell argument length limits and the Gemini CLI piping bug ("Not enough arguments following: p")
- **`-y` flag on Gemini**: Required for headless mode tool access. Without it, `-p` mode blocks tools like `run_shell_command` even though they appear in Gemini's system prompt.
- **macOS mktemp**: Template must end with `X`s only — no `.txt` suffix. Use `mktemp /tmp/prefix-XXXXXXXX` not `mktemp /tmp/prefix-XXXXXX.txt`.
- **Conditional deploy**: Only triggers if "deploy" appears in the original prompt — overnight runs don't accidentally push to production
