---
name: gemini-scout
description: >
  Use when you need a read-only, long-context second opinion — review code or a PR for correctness and
  architecture fit, prepare implementation context before starting an issue, red-team a plan, or audit
  the repo for consistency with its declared architecture. Triggers on phrases like "review this code",
  "code review", "review this PR", "look over this diff", "scout this", "second opinion", "red team this",
  "prepare context for this issue", "architecture scout", "find stale code paths", "is this PR safe",
  "what does this break", or any request for external analysis that should not modify files.
  Uses `gemini --approval-mode plan -p` for safe read-only recon.
---

# Gemini Scout

Gemini is a **read-only scout** and **second-opinion reviewer**. It does not own implementation.

| Role | Who |
|------|-----|
| Product direction, issue spec, roadmap memory | ChatGPT |
| Implementation | Claude Code |
| Formal code review | Codex |
| Scout reconnaissance, code review, architecture review, red team | Gemini |

Gemini's output is **never authoritative** — it is a scout report for Claude Code, a counterargument for the PM, and auxiliary input for product decisions.

## What Good Looks Like

A good Gemini Scout report is **repo-grounded, concise, and actionable**.

It should:
- Inspect actual repository evidence (grep/search, then read) before making claims
- Name the specific files, flows, or tests that support each important finding
- Separate observed facts from inferences and recommendations — mark inferences explicitly
- Prioritize 3-7 high-signal findings over exhaustive coverage
- Produce next actions that Claude Code can execute in one pass
- State what could not be verified from available evidence

A bad report:
- Reasons only from the issue description or prompt text, without inspecting code
- Lists generic risks without file evidence
- Recommends large rewrites or speculative architecture changes
- Treats guesses as facts
- Produces a long inventory instead of a decision-oriented scout report

## Hard Rules

- Always use `gemini --approval-mode plan -p`. Never `--yolo` / `-y`.
- Do not edit files. Do not run destructive commands. Do not create patches unless explicitly asked.
- Optimize for architectural and product consistency — not code style nits.
- Prefer 3-7 high-signal findings over exhaustive coverage. Skip low-signal findings.
- If evidence is missing or ambiguous, say so explicitly. Do not invent.

## Base Command

```bash
gemini --approval-mode plan -p "<PROMPT>"
```

The ordering matters: `-p` (short for `--prompt`) consumes the next argument as the prompt text. If `-p` comes before `--approval-mode`, it swallows `--approval-mode` as the prompt value instead of treating it as a flag. Always place `--approval-mode plan` (or any other flags) before `-p`, or use the long form `--prompt` which is explicit about where the prompt ends.

Add `-m <model>` to specify a model if the default isn't suitable. The default model works for most scout tasks; use a stronger model for red-team or architecture-scout on large PRs.

## Long Prompts

The inline `-p "..."` form works for short prompts. For the multi-line prompt templates in the Modes section, use a temp file or heredoc to avoid shell escaping issues:

```bash
# Write the prompt to a temp file
cat > .gemini/tmp/prompt.txt << 'PROMPT_EOF'
You are a read-only repo scout.
Goal: prepare implementation context for this issue:

<ISSUE SUMMARY>

First, inspect repository evidence...
PROMPT_EOF

# Pass it to gemini
gemini --approval-mode plan --prompt "$(cat .gemini/tmp/prompt.txt)"
```

The `--prompt` long form is explicit about boundaries — it avoids the argument ordering risk of `-p`. The heredoc delimiter (`PROMPT_EOF`) must be quoted (`'PROMPT_EOF'`) to prevent shell expansion inside the prompt body.

## Setup Check

Before first use, verify the operational baseline:

- `gemini --version` — Gemini CLI is installed and on PATH.
- `--approval-mode plan` allows read and search (`read_file`, `grep`, `glob`) but blocks mutation (`write`, `edit`) and delegation (`invoke_agent`).
- `mkdir -p .gemini/tmp/` — external artifacts (PR diffs, exported logs, screenshots) must be copied into the project workspace or `.gemini/tmp/` before analysis. See [Workspace Access Constraints](#workspace-access-constraints).

A quick smoke test:

```bash
echo "list files in the current directory" | gemini --approval-mode plan -p "test"
```

## Shared Prompt Rules

Every scout prompt must include these instructions. They ensure the report is repo-grounded rather than speculative.

```
First, inspect repository evidence: grep or search for relevant terms, read the most relevant files, and only then report findings.
Do not rely only on the prompt, issue description, or prior assumptions.
If evidence is missing or ambiguous, say so explicitly.
Prefer 3-7 high-signal findings over exhaustive coverage.
If a finding is low-signal, skip it.
Separate observed facts, inferences, and recommendations.
Mark inferred risks as [inference].
Do not recommend broad rewrites unless the evidence clearly supports them.
```

## Unified Output Skeleton

All modes return results in this structure. Every section is required; if a section has nothing to report, write "None."

```
1. Evidence read
   - Files, diffs, tests, or commands inspected.
   - Be specific: include file paths and what was searched for.

2. Findings
   - High-signal observations grounded in repository evidence.
   - Name the files or tests that support each finding.

3. Risks
   - Architecture, product, or implementation risks.
   - Mark inferred risks as [inference].

4. Files to inspect next
   - Only files likely to change or clarify uncertainty.
   - Do not list every file in the repo.

5. Recommended actions
   - Concrete next steps for Claude Code, Codex, or PM.
   - Keep actions small and executable in one pass.

6. Uncertainties
   - What could not be verified from the available evidence.
   - What additional information would reduce uncertainty.
```

## Modes

Four distinct roles, never combined in a single prompt:

| Mode | Role | When |
|------|------|------|
| Scout | Find a repo-grounded implementation path | Before starting an issue |
| Code Review | Review code, a diff, or PR for correctness, security, and maintainability | Before or after a merge |
| PR Consistency Review | Inspect a PR for architecture drift against project invariants | Before or after a merge |
| Red Team | Challenge assumptions, tradeoffs, failure modes, and alternative designs | Before roadmap or architecture decisions |

Architecture Scout is a Scout variant that audits the full repo against declared architecture invariants rather than a specific issue.

### 1. Issue Scout

Use **before starting implementation**. Feeds Claude Code with a pre-read of relevant files and risks.

```bash
gemini --approval-mode plan -p "You are a read-only repo scout.
Goal: prepare implementation context for this issue:

<ISSUE SUMMARY>

First, inspect repository evidence: grep or search for relevant terms, read the most relevant files, and only then report findings.
Do not rely only on the issue description.
If evidence is missing or ambiguous, say so explicitly.
Prefer 3-7 high-signal findings over exhaustive coverage. Skip low-signal findings.
Separate observed facts, inferences, and recommendations. Mark inferred risks as [inference].

Return using this structure:
1. Evidence read — files inspected, terms searched for.
2. Findings — relevant files, existing patterns to follow, tests likely to change.
3. Risks — architectural constraints that must not be violated, implementation risks.
4. Files to inspect next — only files likely to change.
5. Recommended actions — a minimal implementation plan for Claude Code, small and executable.
6. Uncertainties — what could not be verified from available evidence.

Do not edit files. Do not produce code unless needed as pseudocode."
```

### 2. Code Review

Use **before or after a merge** to review code for correctness, security, performance, and maintainability. This is the general-purpose code review mode — use it when someone says "review this code", "code review this PR", or "look over this diff". It is project-agnostic and does not assume any specific architecture.

```bash
gemini --approval-mode plan -p "You are reviewing code for correctness, security, performance, and maintainability.

First, inspect the repository evidence: read the diff or changed files, grep for relevant patterns, and read the affected files. Only then report findings.
Do not rely only on the PR description or commit messages.
If evidence is missing or ambiguous, say so explicitly.
Prefer 3-7 high-signal findings over exhaustive coverage. Skip low-signal findings.
Separate observed facts, inferences, and recommendations. Mark inferred risks as [inference].

Focus on these concerns, in priority order:
1. Correctness — bugs, logic errors, edge cases, off-by-one, null/undefined handling, race conditions
2. Security — injection risks, auth/authz bypass, data exposure, unsafe deserialization, missing validation
3. Performance — N+1 queries, unnecessary allocations, blocking I/O, missing indexes, large payloads
4. Maintainability — unclear naming, missing error handling, tightly coupled modules, dead code, missing tests

Additional lenses to apply when relevant:
- Does the change follow existing patterns in the codebase, or does it introduce inconsistency?
- Are there tests for the new behavior? Do they cover edge cases?
- Could this change break existing callers or downstream consumers?
- Is there a simpler approach that achieves the same goal?

If the codebase has a CLAUDE.md, ARCHITECTURE.md, or similar conventions file, read it and check the change against it.

Return using this structure:
1. Evidence read — diff or files inspected, patterns searched for, conventions file if any.
2. Findings — high-signal observations grounded in the diff. Mark as Bug, Security, Performance, Maintainability, or Style. Distinguish Blockers from Non-blocking.
3. Risks — what could go wrong if this change lands as-is.
4. Files to inspect next — files that need a closer look or could be affected.
5. Recommended actions — concrete fixes or follow-up items, small and executable in one pass.
6. Uncertainties — what could not be verified from the diff alone, what context would help.

Do not nitpick formatting or style unless it meaningfully impacts readability. Do not flag issues that a linter or formatter would catch. Focus on things a machine cannot detect."

### 3. PR Consistency Review

Use **before or after Codex review**. Codex checks for bugs; this checks for architecture drift.

```bash
gemini --approval-mode plan -p "You are reviewing a PR for product architecture consistency, not code style.

First, inspect the repository evidence: read the PR diff, grep for relevant terms, and read the affected files. Only then report findings.
Do not rely only on the PR description.
If evidence is missing or ambiguous, say so explicitly.
Prefer 3-7 high-signal findings over exhaustive coverage. Skip low-signal findings.
Separate observed facts, inferences, and recommendations. Mark inferred risks as [inference].

Focus on these architecture concerns:
- material-first architecture (Materials → Compiled Views → Context Assembly → Generation Snapshot)
- generated_post treated as output history only, never as factual evidence
- explicit stale state — nothing hides staleness
- server authority over compiled understanding
- no legacy intelligence sync, Source Workspace, or mandatory pre-generation wizard assumptions
- compatibility with current roadmap (SHIP-69, SHIP-73, SHIP-75)

Return using this structure:
1. Evidence read — PR diff, files inspected, terms searched.
2. Findings — high-signal observations grounded in the diff. Mark Blockers, Non-blocking concerns, and Questions explicitly.
3. Risks — architecture or product risks introduced by this PR.
4. Files to inspect next — files worth a closer look.
5. Recommended actions — suggested follow-up issues or pre-merge actions.
6. Uncertainties — what could not be verified from the diff alone."
```

### 4. Red Team

Use **before roadmap or architecture decisions**. Gemini plays skeptic — it argues against the plan to stress-test it before committing.

```bash
gemini --approval-mode plan -p "Act as a skeptical product and architecture reviewer.
Argue against the following plan:

<PLAN>

First, inspect repository evidence: grep or search for relevant code, read affected files, and ground your critique in the actual codebase.
Do not rely only on the plan description.
If evidence is missing or ambiguous, say so explicitly.
Prefer 3-7 high-signal concerns over exhaustive coverage. Skip low-signal concerns.
Separate observed facts, inferences, and recommendations. Mark inferred risks as [inference].

Assume these project goals: stay material-first, avoid overbuilding, preserve implementation momentum.

Look for:
1. hidden complexity the plan glosses over
2. wrong abstractions — things that should not be generalized yet
3. premature generalization — config/extension points with no second consumer
4. risks to the current roadmap
5. cheaper alternatives that achieve the same goal

Return using this structure:
1. Evidence read — files inspected, what was searched for.
2. Findings — strongest arguments against the plan, grounded in repo evidence.
3. Risks — what could go wrong if the plan proceeds as stated.
4. Files to inspect next — files that would be affected.
5. Recommended actions — a concise alternative or modification, not a full rewrite.
6. Uncertainties — what could not be verified, what would change the assessment.

Do not propose a full rewrite. Return a concise critique and a recommended decision."
```

### 5. Architecture Scout

Use **before or after a PR** to audit the repo against its declared architecture.

```bash
gemini --approval-mode plan -p "You are a read-only architecture scout.

First, inspect the repository: grep or search for key terms (material, compiled_view, context_assembly, generation, generated_post, stale, source_workspace, intelligence), read the most relevant files, and only then report.
Do not rely only on prior assumptions about the codebase.
If evidence is missing or ambiguous, say so explicitly.
Prefer 3-7 high-signal findings over exhaustive coverage. Skip low-signal findings.
Separate observed facts, inferences, and recommendations. Mark inferred risks as [inference].

Use these architecture principles as lenses, not as a checklist.
Report only places where the code meaningfully supports, weakens, or complicates the intended architecture:
- material-first generation (Materials → Compiled Views → Context Assembly → Generation Snapshot)
- Materials as evidence truth
- Compiled Views as derived/current project understanding
- Context Assembly as artifact-specific source selection
- Generation Snapshot as artifact-specific truth
- generated_post as output history, not factual evidence
- explicit stale state — nothing hides when context is stale
- server authority for refresh/hydrate/strengthen flows
- no legacy Source Workspace or mandatory pre-generation Intelligence Surface assumptions

Do not list a passing score for each principle. Focus on the seams where things diverge or degrade.

Return using this structure:
1. Evidence read — files inspected, terms searched for, relevant flows traced.
2. Findings — where the code supports the architecture, where it weakens it, where it complicates it.
3. Risks — seams that could worsen with further changes, inferred risks marked [inference].
4. Files to inspect next — files that need human review.
5. Recommended actions — concrete next Claude Code tasks, small and executable.
6. Uncertainties — what could not be verified, what additional context would help."
```

## Integration Flow

```text
ChatGPT → direction, issue spec
    ↓
Gemini Scout → issue-scout (pre-implementation recon)
    ↓
Claude Code → implement
    ↓
Gemini Scout → code review / architecture / PR consistency scan
    ↓
Codex → formal code review
    ↓
ChatGPT → final decisions, roadmap memory integration
```

## Validating a Scout Report

Before acting on a Gemini Scout report, do a quick sanity check:

- **Evidence read**: Are specific files listed? Do they exist in the repo?
- **Findings**: Do they reference the files from Evidence read, or do they float free?
- **Files to inspect next**: Concrete paths, or vague areas?
- **Recommended actions**: Do they name specific files to edit, or are they generic?
- **Uncertainties**: If empty or "None" on a non-trivial question, the report is probably overconfident.

A report that passes is ready to use. A report that fails should be re-run with a more specific prompt or a narrower scope.

## Workspace Access Constraints

Gemini in `--approval-mode plan` can only read files within:

- The current workspace / project directory
- `.gemini/tmp/` (project-relative)

It cannot access `/tmp/`, `~/Downloads/`, or any path outside these two roots. Reads outside these paths will fail silently with a "path not in workspace" error.

Before running a scout that depends on external artifacts, copy them into the allowed workspace:

```bash
# PR diff from another source
cp /tmp/pr-206.diff .gemini/tmp/

# Screenshot or exported log
cp ~/Downloads/error-log.txt .gemini/tmp/

# Then reference the workspace-relative path in the prompt
```

If the artifact is already a project file (e.g., a diff generated from `git diff`), no copy is needed.

## Cleanup

`.gemini/tmp/` artifacts are not automatically cleaned up. After the scout run completes, remove them to keep the repo clean:

```bash
rm -rf .gemini/tmp/
```

Consider adding `.gemini/tmp/` to `.gitignore` if scout runs are frequent. Leftover temp files accumulate over time and can cause confusion in later runs.

## Adapting Prompts

The prompt templates reference Shiplog-specific architecture invariants. For other projects, replace those invariants with the project's own. The scout structure (inspect evidence → report findings → recommend actions) is project-agnostic.

The shared rules (read-first, separate facts from inferences, prefer high-signal, mark uncertainties) should never be removed — they are the core of what makes a scout report trustworthy.
