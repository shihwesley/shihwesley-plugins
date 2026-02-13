# shihwesley-plugins

A personal collection of Claude Code plugins built to sharpen AI-assisted development — better codebase understanding, tighter context windows, structured workflows, and faster research loops.

Seven plugins across three categories, each solving a specific friction point in agent-driven development. Together, they compose into an automated orchestration pipeline that takes a plan from outline to committed code.

## Table of Contents

- [Install](#install)
- [Plugins](#plugins)
  - [Codebase Intelligence](#codebase-intelligence)
  - [Workflow & Environment](#workflow--environment)
  - [Research & Extraction](#research--extraction)
  - [Orchestration (preview)](#orchestration-preview)
- [How They Work Together](#how-they-work-together)
- [Context Window Management](#context-window-management)
  - [The TLDR Read Protocol](#the-tldr-read-protocol)
  - [Token savings in practice](#token-savings-in-practice)
  - [Setup](#setup)
- [Update](#update)
- [License](#license)

## Install

```bash
/plugin marketplace add shihwesley/shihwesley-plugins
```

Then install individual plugins as needed (see tables below).

## Plugins

### Codebase Intelligence

Help your AI agent understand, map, and efficiently consume codebases.

| Plugin | What it does | Install |
|--------|-------------|---------|
| [**mercator-ai**](https://github.com/shihwesley/mercator-ai) | Merkle-enhanced codebase mapping with O(1) change detection | `/plugin install mercator-ai@shihwesley-plugins` |
| [**chronicler**](https://github.com/shihwesley/chronicler) | Ambient `.tech.md` generation with freshness tracking | `/plugin install chronicler@shihwesley-plugins` |
| [**code-simplifier-tldr**](https://github.com/shihwesley/code-simplifier-tldr) | TLDR-aware codebase simplifier — surveys via AST summaries, edits surgically | `/plugin install code-simplifier-tldr@shihwesley-plugins` |

- **mercator-ai** — Generates `CODEBASE_MAP.md` with file purposes, architecture layers, and dependency graphs. Uses a merkle manifest (`docs/.mercator.json`) so re-runs only re-analyze changed files instead of rescanning everything.
- **chronicler** — Watches your source files and auto-generates `.tech.md` docs alongside them. Tracks freshness per file — flags stale docs when the source changes, so documentation stays current without manual upkeep. Reads mercator-ai's merkle manifest to know which files changed, so it only regenerates docs for what's actually different.
- **code-simplifier-tldr** — A TLDR-aware agent that simplifies your codebase. Reads AST summaries to survey file structure, identifies simplification targets (dead code, redundant abstractions, over-engineered patterns), then requests only the specific line ranges it needs to edit. Integrates with mercator-ai's merkle tree so it only considers files whose hash actually changed. Logs every change to a simplification log for auditability.

### Workflow & Environment

Structure your planning process and manage runtime environments without manual switching.

| Plugin | What it does | Install |
|--------|-------------|---------|
| [**interactive-planning**](https://github.com/shihwesley/interactive-planning) | File-based planning with interactive gates and task tracking | `/plugin install interactive-planning@shihwesley-plugins` |
| [**orbit**](https://github.com/shihwesley/Orbit) | Ambient dev environment management — auto-switches dev/test/staging/prod via Docker | `/plugin install orbit@shihwesley-plugins` |

- **interactive-planning** — Combines Manus-style file-based planning with spec-driven multi-file architecture. In task mode, it creates a single `task_plan.md` with phases and dependencies. In spec mode, it generates a manifest with separate spec files per component. Uses `AskUserQuestion` at interactive gates to pause and get user input at key decision points — prevents agents from charging ahead on the wrong path.
- **orbit** — Classifies what you're doing (running tests, debugging, deploying) and auto-switches the right Docker environment. Manages container lifecycle, sidecars, and port mapping across dev/test/staging/prod so you never run tests against the wrong database.

### Research & Extraction

Sandboxed experimentation and capability extraction from external sources.

| Plugin | What it does | Install |
|--------|-------------|---------|
| [**rlm-sandbox**](https://github.com/shihwesley/rlm-sandbox) | Docker sandbox for Python/DSPy + memvid knowledge store (16 MCP tools) | `/plugin install rlm-sandbox@shihwesley-plugins` |
| [**agent-reverse**](https://github.com/shihwesley/agent-reverse) | Reverse engineer capabilities from repos, configs, articles into your workflow | `/plugin install agent-reverse@shihwesley-plugins` |

- **rlm-sandbox** — My take on the [Recursive Language Model](https://arxiv.org/abs/2512.24601) concept from the published paper. Spins up an isolated Docker container with Python, DSPy, and a memvid-backed knowledge store. Exposes 16 MCP tools for code execution, sub-agent orchestration, session persistence, and research automation — all sandboxed so nothing touches your host machine.
- **agent-reverse** — Point it at a GitHub repo, local config, binary, or article and it extracts capabilities, patterns, and skills into your agent workflow. Includes security scanning, manifest tracking, and cross-agent restore so you can port setups between machines.

### Orchestration (preview)

`/orchestrate` is an automated pipeline that chains these plugins into a single execution flow. It takes output from `/interactive-planning` and runs it through plan review, skill matching, worktree isolation, parallel agent dispatch, testing, code review, and incremental commits — hands-off from plan to merged code.

Not yet packaged as a standalone plugin. Currently runs as a personal workflow on top of the installed plugins above. The plan is to ship it once the remaining dependencies (code-review, commit-split) are also pluginized.

[Full pipeline breakdown](docs/orchestrate-workflow.md)

## How They Work Together

These plugins aren't just a collection — they compose into a pipeline. The orchestrator consumes output from each plugin at different stages:

```mermaid
graph LR
    IP["interactive-planning"] --> O["/orchestrate"]
    MA["mercator-ai"] --> O
    CH["chronicler"] --> O
    TLDR["code-simplifier-tldr"] --> O
    AR["agent-reverse"] --> O
    O --> OB["orbit"]
    O --> Ship["commit + merge"]
```

| Pipeline Stage | What happens | Plugins used |
|---------------|-------------|--------------|
| **Plan** | User creates phased plan with tasks, specs, and dependencies | interactive-planning |
| **Ingest** | Reads plan + project context (codebase map, tech docs, AST summaries) | mercator-ai, chronicler, code-simplifier-tldr |
| **Match** | Finds the right skills and agent types for each phase | agent-reverse |
| **Research** | Fetches official docs for unfamiliar tech before agents write code | Context7 / web search |
| **Gate** | Shows full execution plan, gets user approval before touching code | — |
| **Execute** | Creates git worktree per phase, dispatches 2-3 agents in parallel | — |
| **Test** | Runs tests in an isolated environment per phase | orbit |
| **Review** | Automated code review, auto-fixes critical issues | code-review (coming soon) |
| **Commit** | Incremental commits per phase, merge back to feature branch | commit-split (coming soon) |

Stages marked "coming soon" work today as personal skills — they'll become installable plugins in a future release.

## Context Window Management

A recurring problem with AI agents: they burn through context reading full source files, then lose track of earlier work when the window fills up. These plugins include a built-in protocol to prevent that.

### The TLDR Read Protocol

A `PreToolUse` hook intercepts every `Read` tool call. Instead of returning the full file, it returns an AST summary — function signatures, class shapes, imports, key types — at ~200 tokens per file instead of thousands. Agents get enough structure to navigate and decide what to look at. When they need the actual code (to edit a specific function, for example), they request a line range, which bypasses the hook.

How it works:

1. Agent calls `Read` on a file
2. Hook checks the TLDR cache (keyed by merkle hash from mercator-ai, or MD5 fallback)
3. Cache hit → returns the AST summary immediately
4. Cache miss → generates a language-specific summary (Python, TypeScript, Swift, Markdown all have dedicated parsers), caches it, returns the summary
5. Agent requests `offset`/`limit` → hook steps aside, full content returned

**What gets summarized:**
- Python — imports, constants, class/function signatures with type hints
- TypeScript/JavaScript — exports, classes with methods, functions, types/interfaces, enums
- Swift — structs/classes/enums/protocols, properties with types, full function signatures
- Markdown — table of contents, headings, code block languages, key terms, links

**What bypasses the hook:**
- Small files (<100 lines) — not worth summarizing
- Config files (JSON, YAML, TOML, lock files) — structure is the content
- Test files — agents need full assertions to verify behavior
- Line-range requests — the agent already knows what it wants

### Token savings in practice

| Scenario | Without TLDR | With TLDR | Savings |
|----------|-------------|-----------|---------|
| Read one large file | ~2,500 tokens | ~200 tokens | 92% |
| Survey 100-file codebase | ~250,000 tokens | ~20,000 tokens | 92% |
| Merkle diff + TLDR (3 files changed out of 100) | ~250,000 tokens | ~650 tokens | 99.7% |

The merkle integration is the big win. When mercator-ai's manifest tells code-simplifier-tldr which files actually changed, the agent skips everything else entirely. On a 100-file codebase where 3 files changed, you go from 250k tokens to 650.

### Setup

The hook is configured in `.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Read",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/tldr-read-enforcer.sh"
          }
        ]
      }
    ]
  }
}
```

Cache lives at `.claude/cache/tldr/`. Files are named by hash for O(1) lookup. The cache self-populates on first read and invalidates when the merkle hash or file content changes.

## Update

```bash
/plugin marketplace update shihwesley-plugins
```

Individual plugins version independently. Push a fix to a plugin's repo and users pick it up on their next marketplace update — no changes needed here.

## License

MIT
