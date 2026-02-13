# shihwesley-plugins

A personal collection of Claude Code plugins built to sharpen AI-assisted development — better codebase understanding, tighter context windows, structured workflows, and faster research loops.

Seven plugins across three categories, each solving a specific friction point in agent-driven development.

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
| [**code-simplifier-tldr**](https://github.com/shihwesley/code-simplifier-tldr) | AST-based code summarization — 80%+ token savings | `/plugin install code-simplifier-tldr@shihwesley-plugins` |

**mercator-ai** builds a structural map of your codebase so agents navigate by map instead of blind grep. **chronicler** keeps per-file documentation fresh without manual effort. **code-simplifier-tldr** compresses source files into AST summaries that preserve structure while cutting token costs.

### Workflow & Environment

Structure your planning process and manage runtime environments without manual switching.

| Plugin | What it does | Install |
|--------|-------------|---------|
| [**interactive-planning**](https://github.com/shihwesley/interactive-planning) | File-based planning with interactive gates and task tracking | `/plugin install interactive-planning@shihwesley-plugins` |
| [**orbit**](https://github.com/shihwesley/Orbit) | Ambient dev environment management — auto-switches dev/test/staging/prod via Docker | `/plugin install orbit@shihwesley-plugins` |

**interactive-planning** gives agents a structured plan/execute cycle with user checkpoints at decision points. **orbit** detects which environment you need and manages Docker containers so you stop manually toggling between dev, test, staging, and prod.

### Research & Extraction

Sandboxed experimentation and capability extraction from external sources.

| Plugin | What it does | Install |
|--------|-------------|---------|
| [**rlm-sandbox**](https://github.com/shihwesley/rlm-sandbox) | Docker sandbox for Python/DSPy + memvid knowledge store (16 MCP tools) | `/plugin install rlm-sandbox@shihwesley-plugins` |
| [**agent-reverse**](https://github.com/shihwesley/agent-reverse) | Reverse engineer capabilities from repos, configs, articles into your workflow | `/plugin install agent-reverse@shihwesley-plugins` |

**rlm-sandbox** spins up isolated Python environments with a built-in knowledge store for running experiments without touching your host. **agent-reverse** extracts patterns, skills, and configurations from any source — GitHub repos, local configs, articles — and wires them into your agent setup.

## Update

```bash
/plugin marketplace update shihwesley-plugins
```

Individual plugins version independently. Push a fix to a plugin's repo and users pick it up on their next marketplace update — no changes needed here.

## License

MIT
