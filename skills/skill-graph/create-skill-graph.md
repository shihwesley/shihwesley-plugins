---
description: "Create a skill graph for a domain — from research output or from existing skills/docs"
argument-hint: "<domain-slug> [--from-research | --from-existing]"
user-invocable: true
---

# /create-skill-graph

Creates a navigable skill graph for a domain. Works from two sources:

- **From research**: uses `~/.claude/research/<slug>/expertise.md` + `question-tree.md` to derive MOCs
- **From existing files**: inventories `.claude/commands/`, `.claude/skills/`, `.claude/docs/` and clusters them

## Step 1: Detect Source

```
slug = first argument (e.g., "android-xr", "swift", "agent-infra")

IF ~/.claude/research/<slug>/expertise.md exists:
  source = "research"
  Read expertise.md and question-tree.md
ELSE IF .claude/docs/ or .claude/commands/ have files matching the slug:
  source = "existing"
  Inventory matching files
ELSE:
  Tell user: "No research or existing files found for '<slug>'. Run /research <slug> first."
  Stop.
```

## Step 2: Derive MOC Structure

**From research** (question-tree.md drives the structure):
- Each top-level branch in the question tree → candidate MOC
- Branches with <2 sub-questions → merge into a related MOC
- Branches with >8 sub-questions → consider splitting
- Target: 5-10 MOCs

**From existing files**:
- Scan `.claude/commands/`, `.claude/skills/`, `.claude/docs/`, `.claude/agents/` for domain-related files
- Cluster by topic (use filenames, frontmatter descriptions, and content keywords)
- Each cluster with 2+ files → candidate MOC
- Target: 5-10 MOCs

Present the proposed MOC structure to the user:
```
Proposed skill graph for <slug>:
  1. <moc-name> — <description> (N files)
  2. <moc-name> — <description> (N files)
  ...
Proceed?
```

## Step 3: Build the Graph

Create the directory and files:

```
.claude/docs/<slug>-graph/
├── index.md            ← Hub with all MOC descriptions
├── <moc-1>.md          ← Each MOC links to skills/docs/agents
├── <moc-2>.md
├── ...
└── traverse.md         ← Copy from swift-graph, adapt domain name
```

**Index format:**
```markdown
---
description: "Hub MOC for <domain>. Entry point for all <domain> skill graph navigation."
updated: <today>
---

# <Domain> Skill Graph

## <MOC Title>
- [[<moc>.md]] — <description>
...

## Navigation Protocol
1. Read this index — pick the MOC(s) matching your task
2. Read the MOC — pick the specific skills/docs you need
3. Read those files — now you have the knowledge to act
```

**MOC format:**
```markdown
---
description: "<one-line description of this MOC's scope>"
keywords: [<relevant keywords>]
---

# <MOC Title>

<Orientation paragraph>

## <Section>
- **Skill/Doc/Agent: `name`** → `<file path>`
  <one-line description of what this covers and when to use it>

## Cross-References
- Related MOC → [[<other-moc>.md]]
```

**From research**: MOC content comes from the expertise.md sections. Each section's key points become the MOC descriptions. If the research generated a skill or subagent, link those too.

**From existing**: MOC content links to the inventoried files with descriptions from their frontmatter.

## Step 4: Wire Enforcement

After building the graph:

1. **Check skill-registry.json** — if keywords for this domain exist, add `graphPaths` entries pointing to the new index.md
2. **Check for agents** — if domain-specific agents exist (e.g., a research-generated specialist), update their "Before Starting" section to read the graph first
3. **Report**: list all created files and enforcement points

## Step 5: Validate

Run the validation pattern (same as swift-graph-validate.sh but adapted):
- Check all file references resolve
- Check cross-references between MOCs are valid
- Report any issues

## Step 6: Report

```
Skill graph created: .claude/docs/<slug>-graph/
  <N> MOCs, <M> linked files
  Index: .claude/docs/<slug>-graph/index.md
  Enforcement: <what was wired>
  Validation: HEALTHY / <issues>
```
