---
name: skill-matcher
description: Resolves skills and agent types for each phase using registry, AgentReverse, and web search
---

# Skill Matcher

Takes reviewed phase manifests and resolves which skills and agent types to use for each phase.

## Input

- Array of reviewed phase objects (from plan-reviewer)
- Skill registry: `.claude/skills/orchestrator/skill-registry.json`

## Resolution Pipeline

For each phase, run this 3-step cascade:

### Step 1: Local Registry Match

Read `.claude/skills/orchestrator/skill-registry.json`.

Extract keywords from the phase:
- From `languages` array
- From `domains` array
- From task descriptions (grep for keyword matches)
- From `filesTargeted` (use `languageDetection` map from registry)

For each matched keyword, collect the `skills` and `agentType` from the registry.

**Scoring**: If multiple keywords match, prefer the most specific match:
- File extension match (`.swift` → swift) = highest priority
- Explicit domain match (`tca`, `storekit`) = high priority
- General language match (`python`, `node`) = medium priority
- Generic domain match (`api`, `frontend`) = low priority

Select the top skill set and agent type.

### Step 1.5: Skill Graph Enrichment (if graphPaths match)

After registry match, check `graphPaths` in the registry JSON. If any matched keyword has a graphPath entry:

1. Read the skill graph index file (e.g., `.claude/docs/swift-graph/index.md`)
2. Scan section headings for relevance to the phase's task description
3. If a matching MOC section is found, read that MOC file
4. Extract additional skills/plugins listed in the MOC that aren't already in the registry match
5. Append these to the phase's `skills` array

**Why this matters:** The registry has static keyword→skill mappings. The skill graph has richer, curated mappings that include plugins (AvdLee's swiftui-expert, swift-concurrency, swift-testing-expert, core-data-expert, apple-hig-skills), custom skills (/swift-logging, /hig-audit), and cross-references between domains. The graph catches skills the registry doesn't know about.

**Example:** Phase keywords include `swiftui` and `accessibility`.
- Registry returns: `swift-engineering:swiftui-specialist`
- Graph reads `swift-graph/swiftui.md` → finds `swiftui-expert:swiftui-expert-skill` plugin and HIG section → reads `swift-graph/hig-design.md` → adds `apple-hig-skills:hig-foundations` and `swift-engineering:ios-hig`
- Final skills: registry skills + graph-discovered skills

**Graph path resolution:** `graphPaths` values are relative to the project root. Resolve to absolute paths before passing to Read.

**Available skill graphs:**
- Swift/iOS/visionOS: `.claude/docs/swift-graph/index.md`
- (Add future graphs here as they're created)

### Step 2: AgentReverse Suggester (if Step 1 has no match or low confidence)

Call `mcp__agent-reverse__suggester_check` with the workspace root.

If the suggester returns recommendations:
- Cross-reference with the phase's keywords
- If a suggested capability matches → check if already installed
- If not installed → call `mcp__agent-reverse__install_capability` to install permanently

### Step 3: Web Search Fallback (if Steps 1-2 fail)

If no skills matched:
1. `WebSearch("claude code skill {keyword1} {keyword2}")` using the phase's primary keywords
2. Look for GitHub repos with Claude Code skills/commands
3. If found → use `/agent-reverse analyze <url>` flow:
   - `mcp__agent-reverse__repo_fetch` → `mcp__agent-reverse__repo_analyze`
   - Present found capabilities
   - `mcp__agent-reverse__install_capability` for matching ones
4. Register the repo for future use: `mcp__agent-reverse__suggester_add_repo`

### Step 4: Fallback

If all steps fail:
- Use `general-purpose` agent type (from registry `fallbackAgentType`)
- Log a warning: "No specialized skill found for Phase N — using general-purpose agent"

## Output

Enriched phase manifest with skill assignments:

```json
{
  "phase": 1,
  "title": "...",
  "skills": ["swift-engineering:swift-engineer", "swift-engineering:swift-architect"],
  "agentType": "swift-engineering:swift-engineer",
  "skillSource": "registry",
  "matchedKeywords": ["swift", "ios"],
  "confidence": "high",
  "tasks": [...]
}
```

Plus a summary for user display:

```markdown
## Skill Assignments

| Phase | Agent Type | Skills Loaded | Source |
|-------|-----------|---------------|--------|
| 1 | swift-engineering:swift-engineer | swift-engineer, swift-architect | registry |
| 2 | feature-dev:code-architect | code-architect, feature-dev | registry |
| 3 | general-purpose | (none) | fallback |
```

## Notes

- Skills installed via AgentReverse are permanent — they'll be available in future orchestrations
- The skill matcher runs ONCE before execution begins, not per-agent
- If a phase has mixed domains (after reviewer couldn't split further), load ALL matching skills
- The agent dispatcher (Phase 2 of the orchestrator build) uses this output to configure agents
