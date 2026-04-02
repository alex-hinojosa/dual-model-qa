---
name: critique-loop
description: >
  This skill should be used when the user asks about "dual-model QA",
  "critique loop", "CQ methodology", "dual-AI review", "second opinion workflow",
  or needs guidance on setting up cross-model code review processes.
  Provides the Dual-AI Continuous Critique (CQ) framework for using two
  different AI models to catch bugs, security issues, and blind spots that
  a single model misses.
version: 0.1.0
---

# Dual-AI Continuous Critique (CQ) Methodology

A proven framework for using two AI models from different vendors to validate code at every step. Different models have different training distributions, different attention patterns, and different blind spots — the CQ loop exploits this divergence to catch what neither model finds alone.

## The Three-Phase Pipeline

```
Phase 1: Builder AI (builds/fixes, does NOT commit)
    ↓
Phase 2: Reviewer AI (reviews uncommitted diff) → QA_VERDICT: PASS or FAIL
    ↓
Phase 3: Auto-commit + optional deploy (only if PASS)
```

### Phase 1 — Build

**Builder AI** — the primary coding model. Proposes code, tests, and data transforms. Operates with full project context (CLAUDE.md, rules, codebase access). The launcher appends "Do NOT commit or deploy" to the prompt, so all changes stay uncommitted and reviewable.

### Phase 2 — Review

**Reviewer AI (from a different vendor)** — reviews the uncommitted diff for logic, security, data handling, and code quality. Receives the diff plus a structured QA prompt. Tags findings by severity and ends with a binary verdict:

- **`QA_VERDICT: PASS`** — no critical issues found. Warnings and suggestions may still be present.
- **`QA_VERDICT: FAIL`** — one or more critical issues detected (security, breaking changes, data loss). Changes must NOT be committed.

The Reviewer does NOT have project context by default — this is a feature, not a bug. Fresh eyes see what familiar ones miss.

### Phase 3 — Commit Gate

If QA passed, the pipeline auto-commits with a message derived from the original prompt. If the word "deploy" appeared in the prompt, it also runs deployment (e.g., `gcloud run deploy`).

If QA failed: changes stay uncommitted, the user gets an alert, and the QA log details what to fix. The human reviews the findings and decides whether to fix and re-run or override.

### Roles

**Builder AI** — does the building. Never commits or deploys.

**Reviewer AI** — does the reviewing. Must produce a `QA_VERDICT` line.

**Human gate** — reviews QA findings the next morning. Has final say on overrides.

## Severity Framework

Structure all QA output using these tiers:

### Critical Issues (must fix before merge)
- Security vulnerabilities, credential exposure, data leaks
- Breaking changes, missing error handling, race conditions
- Logic reversals, sign convention errors, silent data corruption
- Changes that would cause production failures

### Warnings (should fix soon)
- Code quality issues, anti-patterns, missed edge cases
- Test coverage gaps for new or changed code
- Type mismatches, implicit conversions
- Performance traps (O(n²) at scale, inefficient queries)

### Suggestions (nice to have)
- Readability improvements, naming conventions
- Documentation gaps
- Minor refactoring opportunities
- Style consistency

### Overall Assessment
One paragraph: is this safe to ship? Would you approve this PR?

### QA Verdict (required)
The final line of every review must be exactly one of:
- `QA_VERDICT: PASS` — no critical issues. Warnings and suggestions are acceptable.
- `QA_VERDICT: FAIL` — one or more critical issues found. Do not commit.

Only **Critical Issues** trigger FAIL. Warnings and Suggestions alone still PASS.

## Why Two Models, Not One

A single model reviewing its own output is like proofreading your own writing — you read what you meant, not what's there. Cross-model review works because:

- **Different training data** produces different pattern recognition biases
- **Different attention mechanisms** catch different classes of edge cases
- **No shared blind spots** — where Model A consistently overlooks a pattern, Model B is likely to flag it precisely because it doesn't share that bias
- **Adversarial tension** improves both outputs without human effort

## Proven Results

Deployed on a sports analytics predictive platform:

| Metric | Without CQ | With CQ |
|--------|-----------|---------|
| Time to production | 16-20 weeks | 7 weeks |
| Critical bugs in production | 13+ | 0 |
| Project success rate | 30-40% | 95% |
| Hours saved | — | 320+ |
| ROI on review cost | — | 100-200x |

### Failure modes caught

- **Sign convention errors** — model learning opposite patterns from domain intent
- **Silent data corruption** — syntactically valid code producing wrong answers
- **Edge case gaps** — mishandling None vs 0, boundary conditions
- **Type mismatches** — decimal vs float, implicit conversions
- **Performance traps** — O(n²) queries, inefficient data access patterns
- **Environmental drift** — code works locally, fails in production

## When to Use CQ

- Any autonomous or overnight AI coding session (no human watching)
- Pre-merge review on AI-generated PRs
- Security-sensitive changes (auth, payments, data handling)
- Complex refactors touching multiple files
- Any code that will reach production without manual review

## Reference Materials

- **`references/qa-prompt-template.md`** — the structured prompt sent to the Reviewer AI
- **`references/implementation-guide.md`** — setup instructions for common Reviewer CLI tools
