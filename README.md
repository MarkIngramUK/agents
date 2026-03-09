# AI Agent Pipeline

A collection of reusable AI agents for structured software development workflows, built for use with Claude Code.

## Overview

Agents are specialised roles that handle distinct stages of a workflow. Each agent has a defined scope, a set of inputs it expects, a process it follows, and structured outputs it hands to the next agent. The entry point for loading agents is [`agents.md`](agents.md).

Agents are invoked via the Claude Code Task tool. The main context orchestrates; agents do the specialised work. Never perform an agent's work inline — always delegate.

---

## Workflows

### Bug Fix Pipeline

The primary workflow. Takes a bug report through investigation, fixing, review, and submission.

```
[Bug Report]
     │
     ▼
[Bug Triager] ──not actionable──► (reject / request info)
     │ actionable
     ▼
[Bug Investigator] ──produces Fix Plan──►
     │
     ▼
[Bug Fixer] ◄──────────────────────────────────┐
     │ commits ready                            │
     ▼                                          │
[Bug Reviewer] ──gaps found──► update Fix Plan ─┘
     │ approved
     ▼
[PR Reviewer] ◄────────────────────────────────────────────┐
     │ gaps found──► author / Bug Fixer                     │
     │ approved                                             │
     ▼                                                      │
[PR Submitter] ──CI failure / reviewer feedback / BugBot──►─┘
     │ merged / success
     ▼
   Done
```

### Investigation Only

Use **Bug Reporter** when you need a human-readable analysis of a bug without triggering a fix. Useful for triage meetings, issue comments, or deciding whether to prioritise a fix.

```
[Bug Report] ──► [Bug Reporter] ──► Report (terminal — no handoff)
```

### Non-Bug PRs (config, docs, features)

For changes that do not originate from a bug report, enter the pipeline at PR Reviewer.

```
[Commits ready] ──► [PR Reviewer] ──► [PR Submitter]
```

---

## Agents

### Bug Triager
**Model:** fast | **Spec:** [`bug-triager.md`](bug-triager.md)

Parses an incoming bug report from Asana or GitHub. Determines whether it is actionable — i.e. describes a reproducible defect with enough context to locate the affected code. Routes actionable reports to Bug Investigator; rejects or requests clarification otherwise.

**Inputs:** Bug report URL, issue ID, or raw text
**Outputs:** Structured Bug Report Summary → Bug Investigator, or rejection notice

---

### Bug Reporter
**Model:** thinking | **Spec:** [`bug-reporter.md`](bug-reporter.md)

Investigates a bug and produces a plain-language report covering root cause, impact, affected code, and recommended direction. Performs the same codebase analysis as Bug Investigator but outputs a document for a human reader rather than a machine-executable plan. Makes no code changes.

**Inputs:** Bug report URL, issue ID, or raw text
**Outputs:** Bug Report document (terminal — no further handoff)

---

### Bug Investigator
**Model:** thinking | **Spec:** [`bug-investigator.md`](bug-investigator.md)

Performs deep root cause analysis. Traces execution paths, confirms the defect exists in code, reviews surrounding architecture, checks test coverage, and produces a self-contained Fix Plan that Bug Fixer can execute without re-analysing the codebase.

**Inputs:** Bug Report Summary (from Bug Triager)
**Outputs:** Fix Plan → Bug Fixer

---

### Bug Fixer
**Model:** standard | **Spec:** [`bug-fixer.md`](bug-fixer.md)

Executes the Fix Plan. Makes incremental, logically separated commits — each individually reviewable and leaving the codebase in a valid state (compiles, lints, tests pass). Uses `git commit --fixup` when closing gaps identified by Bug Reviewer.

**Inputs:** Fix Plan (from Bug Investigator); optionally Gap Report (from Bug Reviewer)
**Outputs:** Ordered commit list + change summary → Bug Reviewer

---

### Bug Reviewer
**Model:** thinking | **Spec:** [`bug-reviewer.md`](bug-reviewer.md)

Reviews all code changes produced by Bug Fixer against the Fix Plan. Verifies correctness, completeness, architecture fit, test coverage, and code style. Either approves (passes to PR Reviewer) or produces a Gap Report and returns to Bug Fixer for remediation.

**Inputs:** Commit list + Fix Plan
**Outputs:** Approval → PR Reviewer, or Gap Report → Bug Fixer

---

### PR Reviewer
**Model:** thinking | **Spec:** [`pr-reviewer.md`](pr-reviewer.md)

The final quality gate before submission, for all PRs. Performs a deep review of every changed artefact — code, configuration, documentation, instruction files, CI/workflow files — not just source code. Runs build and tests. Also handles post-submission feedback from BugBot, CI failures, and human reviewers.

Unlike Bug Reviewer, this agent has no dependency on a Fix Plan and applies to any PR regardless of origin.

**Inputs:** Branch name and target, or PR URL + feedback
**Outputs:** Approval → PR Submitter, or Gap Report → author / Bug Fixer

**File-type coverage:**
| File type | What is checked |
|---|---|
| Code | Correctness, architecture, tests, style |
| Config (JSON/YAML/etc.) | Syntax, semantics, wildcard scope, hardcoded values, missing guards |
| Instructions (CLAUDE.md, agent specs) | Local availability of referenced commands/files, approach soundness, fragility |
| CI/workflow | Trigger coverage, secret handling, conflicts |
| Cross-file | Consistency between all changed files |

---

### PR Submitter
**Model:** fast | **Spec:** [`pr-submitter.md`](pr-submitter.md)

Pushes the branch, opens the PR (if not already open), monitors CI checks, and watches for reviewer comments. Routes CI failures, BugBot findings, and reviewer change requests back to PR Reviewer.

**Inputs:** Approval from PR Reviewer + commit list
**Outputs:** PR URL on success, or feedback handoff → PR Reviewer

---

## Global Conventions

All agents follow these rules regardless of their specific role:

- **UK English** in all commit messages, comments, PR descriptions, and analysis output.
- **No `Co-Authored-By: Claude` lines** in commits.
- **No "Generated with Claude Code" footer** in PR descriptions.
- **Forward slashes** in all shell command paths.
- **No destructive git actions** (force-push, reset --hard) without explicit user confirmation.
- **Delegate, never inline** — when the pipeline calls for an agent, invoke it via the Task tool; never perform its work in the main context.
- **Escalate when blocked** — if any agent encounters an ambiguous plan, unresolvable failure, or conflicting requirements, it must stop, state what it attempted, and ask the user for guidance.

---

## Model Tiers

| Tier | Default model | Use when |
|---|---|---|
| fast | claude-haiku-4-5 | Low-latency, high-volume, simple decision tasks |
| standard | claude-sonnet-4-6 | Balanced reasoning and code execution |
| thinking | claude-opus-4-6 | Deep analysis, architectural review, edge cases |

Model choices can be overridden per-project or per-invocation.

---

## Planned Agents

| Agent | Purpose |
|---|---|
| Regression Test Writer | Writes thorough regression tests for a confirmed bug, separate from the fix |
| Security Reviewer | Reviews changes touching authentication, authorisation, or data handling for OWASP-category issues |
| Documentation Updater | Updates changelogs, API docs, and CLAUDE.md after a fix is merged |
