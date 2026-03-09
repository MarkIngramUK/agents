# Agent Registry

Reusable AI agents for structured workflows. This file is the entry point.
Load the relevant agent spec file before acting as that agent.

## Bug Fix Pipeline

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

## Agent Registry

| Agent             | Model    | Spec                   | Purpose                                          |
|-------------------|----------|------------------------|--------------------------------------------------|
| Bug Triager       | fast     | `bug-triager.md`       | Parse bug reports; decide if actionable          |
| Bug Reporter      | thinking | `bug-reporter.md`      | Investigate a bug and produce a human-readable report (no fix) |
| Bug Investigator  | thinking | `bug-investigator.md`  | Root-cause analysis + detailed fix plan          |
| Bug Fixer         | standard | `bug-fixer.md`         | Execute fix plan; make incremental commits       |
| Bug Reviewer      | thinking | `bug-reviewer.md`      | Review code changes against Fix Plan; identify gaps |
| PR Reviewer       | thinking | `pr-reviewer.md`       | Deep review of all changed artefacts (code, config, docs, instructions) for any PR; processes BugBot and reviewer feedback |
| PR Submitter      | fast     | `pr-submitter.md`      | Push branch, open PR, monitor CI and reviews     |

## Model Tiers

| Tier     | Default model          | Use when                                          |
|----------|------------------------|---------------------------------------------------|
| fast     | claude-haiku-4-5       | Low-latency, high-volume, simple decision tasks   |
| standard | claude-sonnet-4-6      | Balanced reasoning and code execution             |
| thinking | claude-opus-4-6        | Deep analysis, architectural review, edge cases   |

Model choices can be overridden per-project or per-invocation.

## Global Conventions (all agents)

- UK English in all commit messages, comments, PR descriptions, and analysis output.
- Never include `Co-Authored-By: Claude` lines in commits.
- Never include a "Generated with Claude Code" footer in PR descriptions.
- All paths use forward slashes in shell commands. Never prefix commands with `cd <dir>` — the working directory is already set to the project root and persists across Bash calls. Use absolute paths or rely on the cwd directly.
- Agents do not force-push, reset hard, or take other destructive git actions without explicit user confirmation.
- **Delegate, never inline:** When the pipeline calls for a specific agent (e.g. Bug Reviewer), always invoke it via the Task tool — never perform its work inline in the main context, even when the user requests the action conversationally (e.g. "review this" or "do a deep dive"). The main context orchestrates; agents do the specialised work.

## Escalation

If any agent is blocked (ambiguous plan, unresolvable build failure, conflicting requirements), it must:
1. Stop immediately — do not guess or proceed.
2. State clearly what it attempted and what blocked it.
3. Ask the user for guidance before continuing.

## Continuous Improvement

Each agent should capture lessons during its work. At pipeline completion, the
orchestrating context collects these and proposes spec updates to the user.

**Convention:** Any agent may include an optional `### Lessons` section in its
handoff output, listing:

- A checklist item that was missing and would have caught a real gap
- An edge case not covered by the current spec
- A process step that caused rework or required human correction

Lessons are scoped to the agent that produced them (e.g. a Bug Reviewer lesson
targets `bug-reviewer.md`). The orchestrating context is responsible for
reviewing collected lessons at the end of the pipeline and proposing concrete
spec edits to the user.

## Additional Agent Ideas (stubs — not yet implemented)

- **Regression Test Writer** — specialises in writing thorough regression tests for a confirmed bug, separate from the fix itself. Useful when Bug Investigator identifies a large test coverage gap.
- **Security Reviewer** — reviews bug fixes that touch authentication, authorisation, data handling, or external inputs for OWASP-category issues.
- **Documentation Updater** — updates changelogs, API docs, and CLAUDE.md after a fix is merged.
