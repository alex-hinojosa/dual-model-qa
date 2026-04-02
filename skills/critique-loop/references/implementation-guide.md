# Implementation Guide

How to set up the Dual-Model QA pipeline for different reviewer CLIs and environments.

## Prerequisites

- A primary AI coding tool (Claude Code, Cursor, Copilot, etc.)
- A second AI model accessible via CLI that accepts piped input
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
gemini "Hello, what model are you?" -p
```

### Integration
The plugin writes the prompt to a temp file and passes it as a positional argument:
```bash
QA_PROMPT_FILE=$(mktemp /tmp/gemini-qa-prompt-XXXXXX.txt)
echo "$QA_PROMPT" > "$QA_PROMPT_FILE"
gemini "$(cat "$QA_PROMPT_FILE")" -p 2>&1
rm -f "$QA_PROMPT_FILE"
```

**Important**: Gemini CLI expects the prompt as a positional argument followed by `-p` (non-interactive mode). Piping via `echo | gemini -p` fails with "Not enough arguments following: p". The temp-file approach also avoids shell argument length limits for large diffs.

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

# Run primary AI
claude --dangerously-skip-permissions --model "$MODEL" -p "$PROMPT" 2>&1 | tee -a "$LOGFILE"
CLAUDE_EXIT=${PIPESTATUS[0]}

# QA step
if [ "$RUN_QA" = true ] && [ "$CLAUDE_EXIT" -eq 0 ] && [ "$GIT_BEFORE" != "no-git" ]; then
  # Exclude lockfiles (noise for QA review)
  DIFF=$(git diff "$GIT_BEFORE"..HEAD \
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

$DIFF"
    # Write to temp file to avoid piping issues and arg length limits
    QA_PROMPT_FILE=$(mktemp /tmp/gemini-qa-prompt-XXXXXX.txt)
    echo "$QA_PROMPT" > "$QA_PROMPT_FILE"
    gemini "$(cat "$QA_PROMPT_FILE")" -p 2>&1 | tee -a "$QA_LOGFILE" || true
    rm -f "$QA_PROMPT_FILE"
  fi
fi
```

### Key flags

- `--no-qa` — skip the reviewer step (for docs-only or config changes)
- `--model opus` — use a stronger primary model for complex tasks

### Production hardening tips

- **Exclude lockfiles**: The `':!package-lock.json'` pathspec filters out lockfile noise that can be tens of thousands of lines
- **Truncate large diffs**: The 2000-line guard prevents context window overflows in the reviewer model
- **Temp file for prompt**: Avoids both shell argument length limits and the Gemini CLI piping bug ("Not enough arguments following: p")
