# gemini-scout

A read-only scout skill for Gemini CLI — architecture review, issue prep, and red-team analysis.

## Install

```bash
npx skills add ajaxjiang96/gemini-scout
```

## Modes

| Mode | Role | When |
|------|------|------|
| Scout | Find a repo-grounded implementation path | Before starting an issue |
| Review | Inspect a diff or PR for correctness and architecture fit | Before or after a merge |
| Red Team | Challenge assumptions, tradeoffs, failure modes, and alternative designs | Before roadmap or architecture decisions |

## Requirements

- [Gemini CLI](https://geminicli.com) installed and on PATH
- `--approval-mode plan` for read-only safety

## How it works

Gemini Scout inspects repository evidence before making claims, separates facts from inferences, and produces actions Claude Code can execute in one pass.

It is never authoritative — it's a scout report for Claude Code, a counterargument for the PM, and auxiliary input for product decisions.
