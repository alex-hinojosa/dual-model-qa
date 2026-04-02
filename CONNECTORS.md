# Connectors

## How tool references work

Plugin files use `~~category` as a placeholder for whatever tool the user connects in that category. Plugins are tool-agnostic — they describe workflows in terms of categories rather than specific products.

## Connectors for this plugin

| Category | Placeholder | Options |
|----------|-------------|---------|
| Reviewer CLI | `~~reviewer-cli` | Gemini CLI, GitHub Copilot CLI, Ollama, LM Studio, any CLI that accepts piped prompts |
| Project tracker | `~~project tracker` | Jira, Linear, Asana, GitHub Issues |

## Reviewer CLI Setup

The Dual-Model QA plugin sends code diffs to a second AI model for independent review. The reviewer model must accept prompts via stdin and output to stdout.

### Gemini CLI (recommended)
```bash
echo "review this code" | gemini -p
```

### GitHub Copilot CLI
```bash
echo "review this code" | gh copilot explain -
```

### Ollama (local models)
```bash
echo "review this code" | ollama run codellama
```

### Custom / API-based
Any CLI wrapper that reads stdin and writes to stdout works. The plugin calls:
```bash
echo "$QA_PROMPT" | ~~reviewer-cli 2>&1
```

Replace `~~reviewer-cli` with your actual command during plugin customization.
