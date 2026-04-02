# Dual-Model QA

**Use a second AI model to automatically review code produced by your primary AI — catching blind spots neither model finds alone.**

## Why

A single AI model reviewing its own code is like proofreading your own essay. You read what you *meant* to write, not what's actually there. Different models have different training distributions, different attention patterns, and different blind spots. The Dual-AI Continuous Critique (CQ) methodology exploits this divergence to catch bugs, security issues, and quality problems that a single model misses.

## Proven Results

Deployed on a sports analytics predictive platform:

| Metric | Single Model | Dual-Model CQ |
|--------|-------------|---------------|
| Time to production | 16-20 weeks | 7 weeks |
| Critical bugs shipped | 13+ | 0 |
| Project success rate | 30-40% | 95% |
| Hours saved | — | 320+ |
| ROI | — | 100-200x |

Bugs caught include sign convention errors (model learning backwards), silent data corruption, type mismatches, O(n²) performance traps, and environmental drift between local and production.

## What's Included

### Skills

| Skill | Purpose |
|-------|---------|
| **critique-loop** | The Dual-AI CQ methodology — severity framework, when to use it, proven results |
| **qa-review** | Dispatch code changes to a second model for independent, severity-tagged review |
| **overnight-run** | Launch unattended coding sessions with automatic QA on completion |

### Commands

| Command | Purpose |
|---------|---------|
| `/qa [target]` | Run a QA review on recent changes, staged files, or a specific commit range |
| `/overnight <project> <prompt>` | Queue an autonomous overnight run with built-in QA |

## Setup

### 1. Choose Your Reviewer Model

This plugin is tool-agnostic. You need any AI model accessible via CLI that accepts a prompt argument. See `CONNECTORS.md` for options.

**Recommended:** Gemini CLI (free, fast, different vendor from Claude)
```bash
npm install -g @google/gemini-cli
gemini auth login
```

**Alternatives:** Ollama (local/private), GitHub Copilot CLI, any custom API wrapper.

### 2. Customize the Plugin

After installing, use the plugin customizer to replace `~~reviewer-cli` with your actual reviewer command:

```
customize dual-model-qa
```

### 3. Set Up the Launcher (for overnight runs)

The `/overnight` command needs a launcher script that runs your primary AI autonomously. See `skills/critique-loop/references/implementation-guide.md` for a reference implementation.

## The Three-Phase Pipeline

```
Phase 1: Builder AI (builds/fixes, does NOT commit)
    ↓
Phase 2: Reviewer AI (reviews uncommitted diff) → QA_VERDICT: PASS or FAIL
    ↓
Phase 3: Auto-commit + optional deploy (only if PASS)
```

1. **Phase 1 — Build**: Primary AI makes changes, runs tests, stages files. The launcher appends "Do NOT commit or deploy" to the prompt.
2. **Phase 2 — Review**: Reviewer AI (different vendor) receives the uncommitted diff with a structured severity prompt. Findings tagged as **Critical** (must fix), **Warning** (should fix), **Suggestion** (nice to have). Must end with `QA_VERDICT: PASS` or `QA_VERDICT: FAIL`.
3. **Phase 3 — Commit Gate**: If PASS, auto-commits with a descriptive message. If the prompt contained "deploy", also deploys. If FAIL, changes stay uncommitted and you get an alert.

Only Critical issues trigger FAIL — warnings and suggestions still PASS. The human reviews findings the next morning and has final say on overrides.

## Usage Examples

**Quick QA on last commit:**
```
/qa last commit
```

**QA on a feature branch:**
```
/qa branch
```

**Overnight run with QA:**
```
/overnight my-project "Refactor auth middleware for JWT RS256"
```

**Overnight run without QA (docs only):**
```
/overnight my-project "Update README and API docs" --no-qa
```

## Author

**Alex Hinojosa** — [alexanderhinojosa.com](https://alexanderhinojosa.com)

Senior Lead Recruiter turned AI practitioner. Built the Dual-AI CQ methodology while shipping production AI products and validated it with real business outcomes.

## License

MIT
