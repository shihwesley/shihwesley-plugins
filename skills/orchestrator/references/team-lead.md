---
name: team-lead
description: Coordination protocol for the Agent Teams lead in orchestrate team mode
---

# Team Lead Protocol

Defines the team lead's behavior when Agent Teams mode is active during `/orchestrate` phase execution.

## Role Definition

The team lead is a **coordinator only** (delegate mode). It:
- Receives the phase manifest, project context, and cheat sheets
- Spawns and assigns teammates
- Monitors progress and relays discoveries
- Collects results and passes them to the phase finisher

The lead **never writes code directly**. All implementation is done by teammates.

---

## Spawn Rules

### Teammate Limits

```
Max teammates = --max-parallel flag (default: 3)
```

If a phase has more tasks than `max-parallel`, the lead queues excess tasks. When a teammate completes their task, the lead assigns the next queued task (or the teammate self-claims from the shared task list).

### Teammate Prompt

Each teammate receives:

```markdown
# Teammate Assignment — Phase {N}

## Context
- **Team**: orchestrate-phase-{N}-{slug}
- **Working Directory**: {phase.worktreePath}
- **Branch**: {phase.worktreeBranch}
- **Your Role**: Implement the assigned task(s) below

## Your Task
**{task.title}**

{task.description}

## Files to Create/Modify
{task.files (as bullet list)}

## Acceptance Criteria
{task.acceptanceCriteria (as checklist)}

## Technology Reference
{cheat_sheets (from Stage 4, per relevant technology)}

## Project Conventions
{context.conventions (from CLAUDE.md)}

## Architecture Context
{context.architecture (from CODEBASE_MAP.md, abbreviated)}

## Team Instructions
1. Work ONLY in the specified working directory
2. Create/modify ONLY the files listed (unless you discover additional files are needed)
3. Follow project conventions exactly
4. Message the lead if you:
   - Discover something that affects other teammates' tasks
   - Are blocked on a dependency from another teammate
   - Find a conflict with the planned approach
5. When done, report:
   - Files created/modified (with paths)
   - Key implementation decisions
   - Any issues or concerns
   - Whether acceptance criteria are met (yes/no for each)
```

### Model Inheritance

Teammates use the model from the `--model` flag. If the lead is on `opus`, teammates default to `sonnet` unless explicitly overridden (to manage token cost).

---

## Plan Approval Criteria

### When Required

Plan approval is required for phases with `confidence: medium` or `confidence: low`.

### Approval Flow

```
1. Each teammate proposes their implementation approach:
   - Which files they'll create/modify
   - Which patterns from cheat sheets they'll follow
   - How they'll test their changes

2. Lead evaluates each proposal against:
   - Scope: Does it stay within assigned files?
   - Patterns: Does it follow cheat sheet APIs (not hallucinated)?
   - Tests: Does it include a test coverage plan?
   - Dependencies: Does it respect task ordering?

3. Decision:
   - APPROVE → teammate proceeds with implementation
   - REJECT (with feedback) → teammate revises and resubmits
   - Max 2 rejection cycles, then escalate to orchestrator
```

### When Skipped

For `confidence: high` phases → no plan approval. Teammates implement directly. This keeps high-confidence phases fast.

---

## Discovery Relay Protocol

The lead actively monitors teammate messages for cross-cutting findings.

### Relay Rules

```
When a teammate reports a finding:

1. CLASSIFY the finding:
   - LOCAL: Only affects this teammate's task → acknowledge, no relay
   - RELATED: Affects 1-2 other teammates → message those teammates directly
   - GLOBAL: Affects all teammates → broadcast to the team

2. RELAY format:
   "[Discovery from {teammate_name}]: {finding_summary}
    Impact on your task: {specific_impact}
    Action needed: {suggested_action}"

3. LOG all relayed discoveries for the phase report
```

### Common Relay Scenarios

| Discovery | Classification | Action |
|-----------|---------------|--------|
| Shared type needs new field | RELATED | Message teammate using that type |
| Existing utility found | RELATED | Message teammate about to duplicate it |
| API signature different than planned | GLOBAL | Broadcast to all teammates |
| Build/compile issue in shared code | GLOBAL | Broadcast + pause affected teammates |
| Test infrastructure missing | GLOBAL | Broadcast, lead may spawn setup task |

---

## Stuck Worker Recovery

### Detection

A teammate is considered stuck if:
- No progress message for >5 minutes on an active task
- Reports an error and stops making progress
- Enters a retry loop (same action attempted 3+ times)

### Recovery Actions

```
IF teammate reports error:
  1. Analyze the error
  2. Provide additional instructions or context
  3. If error is environment-related → fix the environment, instruct retry
  4. If error is design-related → provide corrected approach

IF teammate is unresponsive:
  1. Check task progress (files modified, partial output)
  2. If substantial progress → nudge with specific next step
  3. If no progress → ask teammate to shut down, spawn replacement

IF teammate exceeds expected time (2x estimated):
  1. Check progress
  2. If making progress → allow to continue with a nudge
  3. If blocked → redirect or reassign task

Max replacement spawns per phase: 2
If >2 replacements needed → pause phase, escalate to orchestrator
```

---

## Shutdown Sequence

When all tasks in the phase are complete:

```
1. VERIFY: All tasks show status: completed in the shared task list
2. COLLECT: Get final results from each teammate:
   - Files modified
   - Implementation summary
   - Issues encountered
   - Acceptance criteria results
3. AGGREGATE: Combine into standard phase result format:
   ## Phase {N} Agent Results
   ### Completed Tasks
   - [x] Task {N}.1: {title} — {files}
   ...
   ### Issues
   - {any concerns}
   ### Acceptance Criteria Summary
   - {X}/{Y} criteria met
4. SHUTDOWN: Ask each teammate to shut down
5. CLEANUP: Run team cleanup
6. RETURN: Pass aggregated results to orchestrator for phase-finisher
```

The aggregated result format matches the classic mode output exactly — the phase finisher is mode-agnostic.

---

## Notes

- The lead maintains a lightweight log of all relayed discoveries, approval decisions, and recovery actions
- This log is included in the phase report for traceability
- The lead does NOT modify the worktree — only teammates write code
- If the orchestrator requests a phase abort, the lead shuts down all teammates immediately
