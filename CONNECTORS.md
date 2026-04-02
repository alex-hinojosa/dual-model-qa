# Connectors

## How tool references work

Plugin files use `~~category` as a placeholder for whatever tool the user connects in that category. Plugins are tool-agnostic — they describe workflows in terms of categories rather than specific products.

## Connectors for this plugin

| Category | Placeholder | Options |
|----------|-------------|---------|
| Reviewer CLI | `~~reviewer-cli` | Gemini CLI, GitHub Copilot CLI, Ollama, LM Studio, any CLI that accepts a prompt argument |
| Project tracker | `~~project tracker` | Jira, Linear, Asana, GitHub Issues |

## Reviewer CLI Setup

The Dual-Model QA plugin sends code diffs to a second AI model for independent review. The plugin writes the prompt to a temp file, then passes it as a positional argument to the reviewer CLI.

### Gemini CLI (recommended)
```bash
gemini "review this code" -p
```
**Note**: Gemini CLI expects the prompt as a positional argument, not piped stdin. The `-p` flag enables non-interactive mode.

### GitHub Copilot CLI
```bash
gh copilot explain "review this code"
```

### Ollama (local models)
```bash
ollama run codellama "review this code"
```

### Custom / API-based
Any CLI wrapper that accepts a prompt argument works. The plugin calls:
```bash
QA_PROMPT_FILE=$(mktemp /tmp/qa-review-prompt-XXXXXX.txt)
echo "$QA_PROMPT" > "$QA_PROMPT_FILE"
~~reviewer-cli "$(cat "$QA_PROMPT_FILE")" 2>&1
rm -f "$QA_PROMPT_FILE"
```

Replace `~~reviewer-cli` with your actual command during plugin customization.
