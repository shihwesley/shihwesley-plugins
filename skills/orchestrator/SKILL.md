---
name: orchestrator
description: "Automated plan execution pipeline. Use when running /orchestrate on an /interactive-planning output. Handles plan ingestion, review, skill matching, tech research, user gating, and phased execution with worktree isolation, testing, code review, and incremental commits."
metadata:
  author: shihwesley
  version: 1.0.0
user-invocable: true
allowed-tools: ["Bash", "Glob", "Grep", "Read", "Write", "Edit", "Task", "AskUserQuestion", "TaskCreate", "TaskUpdate", "TaskList", "Skill", "ToolSearch", "WebSearch"]
---

# Orchestrator

Takes `/interactive-planning` output and executes it through a 6-stage pipeline: ingest, review, skill-match, research, user gate, and phased execution with worktree isolation.

## When to Use

- After `/interactive-planning` produces a plan directory with `task_plan.md` or `manifest.md`
- When you need automated, isolated execution of a multi-phase plan
- When resuming an interrupted orchestration (`--resume`)

## Pipeline Stages

1. **INGEST** -- read plan files (`task_plan.md` or `manifest.md` + specs), discover project docs (CLAUDE.md, codebase map)
2. **REVIEW** -- agent checks phase sizing, gaps, clarity, dependencies, confidence (skipped on `--resume`)
3. **MATCH** -- resolve skills per phase via registry lookup, AgentReverse fallback, web search fallback, or general-purpose default
4. **RESEARCH** -- fetch official docs for unfamiliar tech (Context7, knowledge store, web), produce cheat sheets
5. **GATE** -- present full execution plan to user for approval
6. **EXECUTE** -- for each phase sequentially, launch a phase-runner subagent that handles worktree creation, agent dispatch, testing, code review, commit, and merge

## Dispatch Modes

The orchestrator picks classic or team mode per-phase based on task count, shared files, and dependencies:

| Condition | Mode |
|-----------|------|
| 1-2 tasks | classic |
| 3+ tasks, all independent | classic |
| 4+ tasks with shared files or deps | team |
| Low/medium confidence, 3+ tasks | team |
| `--team` flag | team (forced) |
| `--no-team` flag | classic (forced) |

## Flags

| Flag | Effect |
|------|--------|
| `--dry-run` | Run stages 1-4, show plan at stage 5, stop |
| `--resume` | Skip completed phases, skip stage 2 review |
| `--max-parallel N` | Max agents per batch (default: 3) |
| `--model sonnet\|opus\|haiku` | Model for dispatched agents |
| `--phase N` | Execute only phase N |
| `--team` | Force Agent Teams mode |
| `--no-team` | Force classic Task dispatch |

## Component Files

All component specifications live in `references/`:

| File | Role |
|------|------|
| `references/plan-ingester.md` | Parse plan files, handle spec-driven and task-based modes |
| `references/plan-reviewer.md` | Review and restructure plans (sizing, gaps, clarity) |
| `references/skill-matcher.md` | 3-step skill resolution cascade |
| `references/tech-researcher.md` | Fetch docs, produce cheat sheets, manage knowledge store |
| `references/skill-registry.json` | Keyword-to-skill/agent mappings |
| `references/worktree-manager.md` | Worktree lifecycle per phase |
| `references/agent-dispatcher.md` | Dual-mode dispatch (team or classic) |
| `references/team-lead.md` | Agent Teams lead coordination protocol |
| `references/phase-runner.md` | Self-contained protocol for one phase |
| `references/phase-finisher.md` | Test, review, commit chain (runs inside phase-runner) |

## Context Window Design

The main orchestrator stays lean (~800 tokens per phase) by delegating all heavy work to phase-runner subagents. Each phase-runner gets its own context window and returns a ~20-line structured result. A 5-phase plan uses roughly 4k tokens for execution.

## Examples

```bash
# Standard execution
/orchestrate ./plans/my-feature

# Preview without executing
/orchestrate --dry-run ./plans/my-feature

# Resume after interruption
/orchestrate --resume ./plans/my-feature

# Execute specific phase with limited parallelism
/orchestrate --phase 2 --max-parallel 2 ./plans/my-feature

# Force team mode
/orchestrate --team ./plans/my-feature
```

## Common Issues

- **Plan file not found**: The orchestrator resolves `PLAN_DIR` to an absolute path via `realpath`. Make sure the directory exists and contains `task_plan.md` or `manifest.md`.
- **No skills matched**: Falls back to `general-purpose` agent type with a warning. Add keywords to your plan's task descriptions to improve matching.
- **Phase-runner fails after retries**: You get four options: fix manually, skip the phase, abort, or retry from scratch. The phase-runner's error message usually points to the root cause.
- **Context window fills up**: The phase-runner architecture prevents this by design. If the main orchestrator still fills up, it's likely from stages 1-4 reading too many files. Use `--resume` to skip stages on retry.
- **Merge conflicts**: The orchestrator pauses and asks you to resolve manually. It won't force-merge.
