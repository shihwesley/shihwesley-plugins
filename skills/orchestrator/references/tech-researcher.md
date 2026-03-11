---
name: tech-researcher
description: Researches unfamiliar technologies before agents code — fetches official docs via Context7 or web, caches locally, produces cheat sheets per phase
---

# Tech Researcher

**Purpose**: Before any agent writes code, research every technology in the plan that isn't already well-documented locally. Fetch official documentation, analyze it, and produce cached reference material so agents code with accurate API knowledge — not hallucinated patterns.

**When**: Runs as Stage 3.5 in the `/orchestrate` pipeline, after skill matching (Stage 3) and before the user gate (Stage 4).

## Why This Matters

Without doc research, agents will:
- Hallucinate API signatures that don't exist
- Use deprecated patterns from training data
- Mix up similar-looking APIs from different libraries
- Write "plausible-looking slop" that fails at runtime

With doc research, agents receive **verified API reference** in their prompt context, producing code that matches what the library maintainers intended.

## Input

- Reviewed phase manifests (from plan-reviewer)
- Skill assignments (from skill-matcher)
- List of already-cached docs: `.claude/docs/*/`

## Process

### Step 0: Check Existing Research Artifacts

Before identifying technologies, scan for existing research from `/research`:

```bash
HOME_DIR=$(echo ~)
ls "$HOME_DIR/.claude/research/" 2>/dev/null
```

For each research topic found, read `sources.json` to check the artifacts field:

```json
// ~/.claude/research/<slug>/sources.json → artifacts
{
  "expertise": "~/.claude/research/<slug>/expertise.md",
  "skill": "~/.claude/skills/<slug>/SKILL.md",
  "subagent": "~/.claude/agents/<slug>-specialist.md"
}
```

Build a **research inventory** — map of topic slugs to their available artifacts. This is checked in Step 2 before any fetching.

Also check what's in the knowledge store:
```
ToolSearch(query="rlm_search")
rlm_knowledge_status()
```

This tells you what's already indexed and searchable. Technologies that are already in the store can skip straight to cheat sheet production (Step 4).

### Step 1: Identify Technologies Per Phase

For each phase, extract technology identifiers from:
- `languages` array (e.g., "swift", "typescript")
- `domains` array (e.g., "database", "networking")
- Task descriptions (grep for library names, framework names, import statements)
- `filesTargeted` (infer from file extensions and paths)
- `findings.md` technical decisions (often mention specific libraries)

Produce a tech inventory per phase:
```json
{
  "phase": 2,
  "technologies": [
    {"name": "memvid", "type": "library", "confidence": "unknown"},
    {"name": "GRDB", "type": "library", "confidence": "known-skill"},
    {"name": "SwiftUI", "type": "framework", "confidence": "known-skill"},
    {"name": "Vapor", "type": "framework", "confidence": "unknown"}
  ]
}
```

### Step 2: Check What's Already Known

For each technology, check these sources (in priority order):

**a) Research artifacts** (from `/research` pipeline) — **check this first**

Match the technology name against the research inventory from Step 0:
```bash
# Check if a research topic exists for this tech
ls "$HOME_DIR/.claude/research/<tech-slug>/" 2>/dev/null
```

If `expertise.md` + `knowledge.mv2` exist:
- Mark as **"research-cached"** — highest confidence source
- Derive cheat sheet from expertise.md (no re-research needed)
- The knowledge store is available for deep-dives: `rlm_search(query="...", project="<slug>")`
- If `sources.json` has a `skill` artifact → note the skill for agent dispatch
- If `sources.json` has a `subagent` artifact → note for specialist agent routing

**b) Knowledge store** (indexed docs from any source)

```
rlm_search(query="{tech-name} API", project="<slug-or-default>", top_k=3)
```
If results exist and are relevant → mark as "store-cached", skip fetching.

**c) Local doc cache** — `.claude/docs/<library>/`
```
# Use absolute path or omit path param to default to CWD. Never pass ~ to Glob.
Glob: .claude/docs/{tech-name}*/**/*.md
```
If found and <7 days old → mark as "cached", skip research.

**d) Loaded skills** — Does the skill-matcher already assigned a specialized skill?
- If `swift-engineering:grdb` is loaded → GRDB is "known-skill", skip deep research
- If `swift-engineering:composable-architecture` is loaded → TCA is "known-skill"
- If no specialized skill → mark as "needs-research"

**e) Context7 availability** — Can Context7 resolve this library?
```
mcp__context7__resolve-library-id(
  libraryName: "{tech-name}",
  query: "How to use {tech-name} for {phase-context}"
)
```
If resolved → mark as "context7-available"
If not found → mark as "web-search-needed"

### Step 3: Fetch Documentation

For each technology marked "needs-research":

**Priority 1: Context7 (structured docs)**
```
# Resolve library ID
result = mcp__context7__resolve-library-id(
  libraryName: "memvid",
  query: "Complete API reference for memvid library"
)

# Fetch comprehensive docs — multiple queries for full coverage
mcp__context7__query-docs(
  libraryId: result.libraryId,
  query: "Getting started, installation, basic usage examples"
)

mcp__context7__query-docs(
  libraryId: result.libraryId,
  query: "Complete API reference, all methods and parameters"
)

mcp__context7__query-docs(
  libraryId: result.libraryId,
  query: "Advanced patterns, best practices, common pitfalls"
)
```

The existing `context7-doc-cacher.sh` hook auto-saves these to `.claude/docs/<library>/`.

**Priority 2: Web search (if Context7 doesn't have it)**
```
WebSearch("{tech-name} official documentation API reference")
WebSearch("{tech-name} getting started tutorial")
```

Then fetch the official docs:
```
WebFetch(url: official_docs_url, prompt: "Extract complete API reference, all public methods, parameters, return types, and usage examples")
```

Save to local cache:
```
Write: .claude/docs/{tech-name}/api-reference.md
Write: .claude/docs/{tech-name}/getting-started.md
```

**Priority 3: GitHub README (last resort)**
```
WebFetch(url: "https://github.com/{owner}/{repo}", prompt: "Extract README with installation, usage, API docs")
```

### Step 4: Analyze and Produce Cheat Sheet

**For "research-cached" tech (from Step 2a):** Derive the cheat sheet directly from the expertise.md file. Read it and extract the Quick Reference and Key APIs sections. Optionally query the knowledge store for additional detail:

```
rlm_search(query="{tech} common patterns and API usage", project="<slug>", top_k=5)
```

This skips all fetching — the research pipeline already did the heavy lifting.

**For "store-cached" tech (from Step 2b):** Query the knowledge store to build the cheat sheet:

```
rlm_search(query="{tech} core API methods and signatures", project="<slug>", top_k=8)
rlm_search(query="{tech} common pitfalls", project="<slug>", top_k=5)
```

Synthesize the search results into the cheat sheet format below.

For each researched technology, produce a **concise cheat sheet** that agents receive in their prompt:

```markdown
# {Tech Name} — Agent Reference

## Installation
{how to add to project}

## Core API
{key classes/functions with signatures}

## Usage Patterns
{3-5 most common patterns with code examples}

## Common Pitfalls
{things that look right but are wrong}

## Version Notes
{current version, breaking changes from previous}
```

Save to: `.claude/docs/{tech-name}/cheat-sheet.md`

This cheat sheet is what gets injected into the agent's prompt during dispatch (Step 5b of /orchestrate).

### Step 5: Report

Output a research summary for the user gate:

```markdown
## Tech Research Results

| Technology | Source | Status | Cached At |
|-----------|--------|--------|-----------|
| visionOS | /research artifact | Research-cached — expertise + store available | ~/.claude/research/visionos-development/ |
| RealityKit | /research artifact | Research-cached — expertise + store + skill + specialist | ~/.claude/research/visionos-development/ |
| memvid | Web (GitHub) | Researched — cheat sheet created | .claude/docs/memvid/ |
| GRDB | Existing skill | Known — swift-engineering:grdb loaded | — |
| Vapor | Context7 | Researched — 3 doc pages cached | .claude/docs/vapor/ |
| SwiftUI | Existing skill | Known — swiftui-patterns loaded | — |

### New Caches Created
- .claude/docs/memvid/api-reference.md (12k)
- .claude/docs/memvid/cheat-sheet.md (3k)
- .claude/docs/vapor/getting-started.md (8k)
- .claude/docs/vapor/api-reference.md (15k)
- .claude/docs/vapor/cheat-sheet.md (4k)
```

## Integration with Agent Dispatch

The agent-dispatcher (Stage 5b) must include cached docs in agent prompts:

```markdown
# In agent prompt template, add:

## Technology Reference
{for each tech the phase uses}

### {tech-name}
{contents of .claude/docs/{tech-name}/cheat-sheet.md}
```

This ensures agents have **verified, official API knowledge** — not training-data hallucinations.

## Parallel Research

Technologies across phases can be researched in parallel:

```
# Launch research agents (max 2-3) for different technologies
Task(subagent_type="Explore", prompt="Research memvid: ...", run_in_background=True)
Task(subagent_type="Explore", prompt="Research Vapor: ...", run_in_background=True)
```

Each research agent:
1. Calls Context7 or WebSearch
2. Analyzes the docs
3. Writes the cheat sheet to `.claude/docs/`
4. Returns summary

## Cache Management

- Docs older than 7 days → re-fetch on next orchestration
- Cheat sheets include a `<!-- Cached: {ISO date} -->` header for age checking
- To force refresh: delete `.claude/docs/{tech-name}/` and re-run
- Cache is permanent and shared across all projects in `~/Source/`

## Edge Cases

| Scenario | Handling |
|----------|----------|
| Context7 rate limited (max 3 calls) | Switch to web search immediately |
| WebFetch blocked | Ask user to paste docs manually |
| Docs are enormous (>50k) | Summarize to cheat sheet only, don't cache full docs |
| Tech is internal/proprietary | Check project's own docs/ folder, README, inline comments |
| Multiple versions exist | Match version from package.json/Package.swift/requirements.txt |
| Tech is too new for Context7 | Web search → GitHub → README as fallback |

## Notes

- This stage is the **anti-slop layer** — it's what prevents agents from making up APIs
- Research agents are `Explore` type (read-only, no file modifications except doc cache)
- The cheat sheet format is deliberately concise — agents don't need full docs, they need accurate API signatures and patterns
- If a technology is already covered by a loaded skill (e.g., swift-engineering:grdb), skip research — the skill IS the documentation
