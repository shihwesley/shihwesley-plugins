---
name: agent-dispatcher
description: Dual-mode dispatcher — Agent Teams (experimental) or classic Task-based, max 2-3 parallel
---

# Agent Dispatcher

Manages dispatch of specialized agents for tasks within a phase. Supports two modes:
- **Team mode**: Agent Teams with a coordinating lead and inter-agent messaging (experimental)
- **Classic mode**: Task-based fire-and-forget dispatch (stable, default)

## Input

A phase manifest (from skill-matcher output):
```json
{
  "phase": 1,
  "title": "...",
  "tasks": [...],
  "skills": ["swift-engineering:swift-engineer"],
  "agentType": "swift-engineering:swift-engineer",
  "worktreePath": "/path/to/project-phase-1",
  "confidence": "high"
}
```

Plus: project context (conventions, tech stack), `DISPATCH_MODE` from orchestrate.md mode resolution.

---

## Section A: Mode Resolution

Determined by the orchestrator entry point and passed to the dispatcher:

```
IF DISPATCH_MODE == "team":
  → Use Section B (Team Mode Dispatch)
ELSE:
  → Use Section C (Classic Mode Dispatch)
```

The dispatcher does NOT resolve the mode itself — it receives `DISPATCH_MODE` from `/orchestrate`.

---

## Section B: Team Mode Dispatch

When `DISPATCH_MODE == "team"`, the dispatcher creates an Agent Teams instance per phase.

### B1. Create Team

For each phase, create a team:

```
Team name: orchestrate-phase-{N}-{slug}

Team composition:
- Lead: delegate mode (coordinates only, does not write code)
  - Receives: phase manifest, project context, cheat sheets from Stage 4
  - Follows: .claude/skills/orchestrator/team-lead.md protocol
- Teammates: one per task (or grouped by tightly-coupled module)
  - Each receives: worktree path, task description, cheat sheets, acceptance criteria
  - Model: inherited from --model flag (default: sonnet)
```

The lead operates in **delegate mode** (Shift+Tab equivalent) — it coordinates teammates but never writes code directly.

### B2. Shared Task List

Create tasks with dependency mirroring from the phase manifest:

```
For each task in phase.tasks:
    TaskCreate(
        subject = "Phase {N}: {task.title}",
        description = task.description + acceptance_criteria + file_targets,
        activeForm = "Implementing {task.title}"
    )

    # Wire dependencies
    if task.dependsOn:
        TaskUpdate(taskId=task.id, addBlockedBy=task.dependsOn)
    # else: no dependencies → teammates self-claim
```

Tasks that produce files consumed by other tasks → `blockedBy` dependencies.
Independent tasks → no dependencies, teammates pick up in order.

### B3. Plan Approval Gate

For phases with `confidence: medium` or `confidence: low`:

```
Lead MUST require plan approval before teammates implement:
  - Each teammate proposes their approach (files to touch, patterns to follow)
  - Lead evaluates against: cheat sheet patterns, scope boundaries, test coverage plan
  - Lead approves or rejects with specific feedback
  - Rejected → teammate revises approach before coding

For confidence: high → no plan approval, teammates implement directly.
```

### B4. Discovery Relay

The lead monitors teammate messages and relays relevant findings:

```
When teammate reports a finding:
  1. Check: does this affect other teammates' tasks?
  2. If yes → message the affected teammate with the discovery
  3. Log all relayed discoveries for the phase report

Examples:
  - Teammate A discovers a shared type needs a new field → relay to Teammate B who uses that type
  - Teammate A finds an existing utility → relay to Teammate B who was about to write the same thing
  - Teammate A identifies an API change → broadcast to all teammates
```

### B5. Completion

When all tasks are done:

```
1. Lead waits for all teammate tasks to show status: completed
2. Lead collects results from each teammate (files modified, issues, criteria met)
3. Lead asks each teammate to shut down
4. Lead runs team cleanup
5. Lead returns aggregated results in standard format (same as Section C output)
```

The aggregated result format is identical to classic mode — the phase finisher doesn't need to know which mode was used.

---

## Section C: Classic Mode Dispatch

The original Task-based dispatch. Used when `DISPATCH_MODE == "classic"`.

### Batching

Tasks within a phase are batched for parallel execution:

```
MAX_PARALLEL = 3  (configurable, default 2-3)

tasks = phase.tasks
batches = []
current_batch = []

for task in tasks:
    current_batch.append(task)
    if len(current_batch) >= MAX_PARALLEL:
        batches.append(current_batch)
        current_batch = []

if current_batch:
    batches.append(current_batch)
```

### Batch Execution

For each batch, launch agents **in parallel** using the Task tool:

```
for batch in batches:
    # Launch all agents in this batch simultaneously
    agents = []
    for task in batch:
        agent = Task(
            description = "Phase {N} Task: {task.title}",
            subagent_type = phase.agentType,
            prompt = build_agent_prompt(phase, task, context),
            run_in_background = True
        )
        agents.append(agent)

    # Wait for all agents in batch to complete
    for agent in agents:
        result = TaskOutput(task_id=agent.id, block=True)
        collect_result(task, result)

    # Proceed to next batch only after current batch completes
```

**Important**: Do NOT launch the next batch until the current one finishes. This prevents resource exhaustion and ensures incremental progress.

## Agent Prompt Template

Each agent receives a structured prompt:

```markdown
# Task Assignment

## Context
- **Phase**: {phase.phase} — {phase.title}
- **Working Directory**: {phase.worktreePath}
- **Branch**: {phase.worktreeBranch}

## Your Task
**{task.title}**

{task.description}

## Files to Create/Modify
{task.files (as bullet list)}

## Acceptance Criteria
{task.acceptanceCriteria (as checklist)}

## Project Conventions
{context.conventions (from CLAUDE.md)}

## Architecture Context
{context.architecture (from CODEBASE_MAP.md, abbreviated)}

## Swift Skill Graph (iOS/visionOS agents only)
{IF agentType starts with "swift-engineering:" OR agentType in ["ios-specialist", "visionos-specialist"]}
Before writing any code, navigate the Swift skill graph:
1. Read `.claude/docs/swift-graph/index.md` — pick the MOC(s) matching your task
2. Read those MOC(s) — pick specific skills/docs listed
3. Read only those target files
4. For API signatures: `cd ~/Source/neo-research && .venv/bin/python3 scripts/knowledge-cli.py search "<API>" --project <store> --top-k 5`
   Stores: `spatial-computing`, `swiftui`, `ml-ai`, `foundation-core`, `networking`, `security-auth`, `media-audio`

Full protocol: `.claude/docs/swift-graph/traverse.md`

Do NOT read all docs in `.claude/docs/ios-development/` or `visionos-development/` linearly. The graph tells you which files matter.
{END IF}

## Knowledge Store (from /research)
{IF research artifacts exist for technologies in this phase}
Tech docs are indexed in the knowledge store. Query for specific API details:
- `ToolSearch(query="rlm_search")` — load the search tool first
- `rlm_search(query="<specific question>", project="<slug>")` — hybrid search for doc chunks
- `rlm_ask(question="<how to do X>", project="<slug>")` — get context chunks for a question

Available research topics:
{for each matching research topic}
- **{topic}** (project: "{slug}") — {N} pages indexed
  - Expertise: {expertise_path}
  - Deep-dive: `rlm_search(query="...", project="{slug}")`
{end for}

Do NOT read full doc files into context. Use rlm_search for targeted lookups.
{END IF}

## Instructions
1. Work ONLY in the specified working directory
2. Create/modify ONLY the files listed above (unless you discover additional files are needed)
3. Follow the project conventions exactly
4. Use rlm_search for API details — do not guess library signatures
5. When done, provide a summary of:
   - Files created/modified (with paths)
   - Key implementation decisions
   - Any issues or concerns
   - Whether acceptance criteria are met (yes/no for each)
```

### Skill Loading

The prompt is enhanced with skill content when the agent type has matching skills:

- For `swift-engineering:swift-engineer` → the agent has built-in Swift engineering skills
- For `feature-dev:code-architect` → the agent has built-in feature-dev skills
- For `general-purpose` → load any matched skills via `Skill` tool references in the prompt

**Note**: Plugin-based agents (swift-engineering, feature-dev) have their skills built in. For custom-installed skills (via AgentReverse), include the skill content directly in the agent prompt.

### Research Artifact Routing

When `/research` has produced artifacts for a technology used in this phase, the dispatcher applies three upgrades:

**1. Specialist agent substitution** — If `sources.json` has an `artifacts.subagent` path (e.g., `~/.claude/agents/visionos-development-specialist.md`), consider using that specialist as the `subagent_type` instead of the generic agent:

```
# Check: does a research-generated specialist exist for this phase's domain?
if research_artifacts[tech].subagent exists:
    # Use the specialist — it has deep expertise built in
    subagent_type = "<slug>-specialist"  # matches the agent filename
```

Only substitute if the specialist covers the phase's primary technology. If the phase mixes multiple domains, use the generic agent type and inject the knowledge store instructions.

**2. Skill injection** — If `sources.json` has an `artifacts.skill` path, the research-generated skill has patterns and conventions for this technology. Read the skill content and inject it into the agent prompt's Architecture Context section.

**3. Knowledge store access** — Always include the Knowledge Store section (from the prompt template above) when research artifacts exist. This lets agents pull specific API details mid-task without having everything in their context upfront.

## Result Collection

After each agent completes, collect:

```json
{
  "taskId": "1.1",
  "status": "success|partial|failed",
  "filesModified": ["path/to/file1.ts", "path/to/file2.ts"],
  "summary": "Agent's completion summary",
  "issues": ["Any reported concerns"],
  "acceptanceCriteriaMet": {
    "criterion1": true,
    "criterion2": false
  }
}
```

### Aggregation

After all batches complete, aggregate results:

```markdown
## Phase {N} Agent Results

### Completed Tasks
- [x] Task 1.1: {title} — {files modified}
- [x] Task 1.2: {title} — {files modified}

### Partial/Failed Tasks
- [ ] Task 1.3: {title} — {error message}

### Files Modified (all tasks)
- path/to/file1.ts (Task 1.1)
- path/to/file2.ts (Task 1.1, Task 1.2)

### Issues
- {any reported concerns from agents}

### Acceptance Criteria Summary
- 8/10 criteria met
- Missing: criterion X, criterion Y
```

## Error Handling

### Agent Failure

If an agent fails (crashes, times out, or reports failure):

1. **Log the failure** in progress.md
2. **Do NOT retry automatically** — the phase finisher handles retries during test/review
3. **Continue with remaining agents** in the batch
4. **Report to orchestrator** so it can decide whether to proceed to testing

### All Agents Fail

If every agent in a batch fails:

1. **Pause orchestration**
2. **Ask user**: "All agents in batch N failed. Retry batch / Skip phase / Abort?"
3. If retry → re-dispatch the batch with the same skills
4. If skip → mark phase as skipped, proceed to next phase
5. If abort → stop orchestration, report status

### Timeout

Default agent timeout: 10 minutes per task.
If an agent exceeds timeout:
- Check output file for partial progress
- If substantial progress → extend timeout by 5 minutes (once)
- If no progress → kill and report failure

## Configuration

Settable per-orchestration:

| Setting | Default | Description |
|---------|---------|-------------|
| `maxParallel` | 3 | Max agents per batch |
| `agentTimeout` | 600000 | Timeout per agent (ms) |
| `retryOnFailure` | false | Auto-retry failed agents |
| `model` | "sonnet" | Model for dispatched agents |

These can be overridden via `/orchestrate --max-parallel 2 --model opus`.

## Notes

- Agents are **stateless** — each task gets a fresh agent with no memory of previous tasks
- Inter-task communication happens through the **filesystem** (worktree), not through agent state
- If Task A creates a file that Task B needs, they must be in **different batches** (B after A)
- The dispatcher does NOT modify files directly — it only launches agents and collects results

## Mode Comparison

| Aspect | Classic (Section C) | Team (Section B) |
|--------|-------------------|-----------------|
| Communication | Fire-and-forget | Inter-agent messaging |
| Coordination | Batch-wait | Shared task list + self-claim |
| Discovery sharing | None | Lead relays findings |
| Plan approval | None | For medium/low confidence phases |
| Context | Shared main context | Each teammate has own context |
| Token cost | Lower | Higher (separate instances) |
| Error recovery | Log + continue | Lead redirects or replaces |

### What stays the same across modes
- Agent prompt template (with minor additions for team instructions)
- Skill loading
- Result collection format (phase finisher is mode-agnostic)
- Error handling (escalation to orchestrator)
- Configuration (maxParallel, model, timeout)
