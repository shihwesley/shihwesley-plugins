---
name: phase-runner
description: Self-contained agent that executes a single phase (worktree → dispatch → test → review → commit → merge) in its own context window
---

# Phase Runner

Runs inside a **subagent** with its own context window. The main orchestrator launches one phase-runner per phase. This keeps the main context lean — it only sees the ~20-line result summary, not the 15-30k tokens of test output, review findings, and agent coordination.

## Input (provided in agent prompt)

The orchestrator builds a self-contained prompt with:

```
PHASE_MANIFEST    — phase number, title, tasks, files, acceptance criteria
SKILLS            — agent type + skill list from skill-matcher
CHEAT_SHEETS      — tech reference content from Stage 4 research
PROJECT_CONTEXT   — conventions, architecture, CLAUDE.md summary
DISPATCH_MODE     — "classic" or "team"
MAX_PARALLEL      — max agents per batch
MODEL             — model for sub-agents
GIT_ROOT          — absolute path to project git root
PLAN_DIR          — absolute path to plan directory
SPEC_DRIVEN       — true/false (affects commit messages and progress format)
SPEC_INFO         — if spec-driven: spec name, sprint number, requirements list
WORKFLOW_CONFIG    — execution settings from workflow.md (retries, gates, enforcement) or null
PROMPT_TEMPLATES   — rendered phase/continuation prompt templates from workflow.md or null
HANDOFF_PATH      — absolute path to handoff.md or null
PROGRESS_LOG_PATH — absolute path to progress-log.md or null
ATTEMPT_NUMBER    — 1 for first run, 2+ for retries (from orchestrator retry logic)
PREVIOUS_RESULT   — null for first run, or {status, error} from prior failed attempt
```

## Execution Steps

### Step 0: Read Handoff and Render Prompt

#### 0a. Read handoff state

```
if HANDOFF_PATH exists and file exists at that path:
    Read: HANDOFF_PATH  # Always <150 lines, TLDR-safe
    Extract:
    - LAST_PHASE = last_completed_phase (name, summary, key_files, decisions)
    - CURRENT_PHASE = current_phase (name, criteria)
    - WORKSPACE_STATE = workspace_state (branch, last_commit, build_status)
    - ARCH_DECISIONS = architecture_decisions (list — carry forward)
    - COMPLETED_PHASES_SUMMARY = build list from handoff context

if HANDOFF_PATH is null or file doesn't exist:
    # First phase or no workflow contract. Use defaults.
    LAST_PHASE = null
    ARCH_DECISIONS = []
    COMPLETED_PHASES_SUMMARY = []
```

#### 0b. Render prompt from template

```
if PROMPT_TEMPLATES is not null:
    if ATTEMPT_NUMBER == 1:
        RENDERED_PROMPT = PROMPT_TEMPLATES.phase with substitutions:
            phase.name, phase.number, phase.total, phase.description,
            phase.criteria, completed_phases = COMPLETED_PHASES_SUMMARY,
            project.name, project.build_cmd, project.test_cmd,
            workspace.path = WORKTREE_PATH (computed in Step 1)
    else:
        RENDERED_PROMPT = PROMPT_TEMPLATES.continuation with substitutions:
            attempt = ATTEMPT_NUMBER, phase.name,
            previous_result.status, previous_result.error

    # RENDERED_PROMPT replaces the default task prompt in Step 2 agent dispatch.

if PROMPT_TEMPLATES is null:
    # Use existing built-in prompt construction (backward compatible).
    RENDERED_PROMPT = null  # Step 2 uses its default prompt building
```

Note: WORKTREE_PATH isn't known yet at Step 0 — the template substitution for `workspace.path` happens after Step 1 creates the worktree. Store the template with a placeholder and substitute after Step 1.

### Step 1: Create Worktree

```bash
GIT_ROOT={provided}
PROJECT_NAME=$(basename "$GIT_ROOT")
PHASE_NUM={phase.phase}
PHASE_SLUG=$(echo "{phase.title}" | tr '[:upper:]' '[:lower:]' | tr ' ' '-' | tr -cd 'a-z0-9-' | head -c 30)
BRANCH_NAME="orchestrate/phase-${PHASE_NUM}-${PHASE_SLUG}"
WORKTREE_PATH="${GIT_ROOT}/../${PROJECT_NAME}-phase-${PHASE_NUM}"

# Create worktree
git worktree add "$WORKTREE_PATH" -b "$BRANCH_NAME"

# Symlink .claude for skill/command access
if [ -d "${GIT_ROOT}/.claude" ] && [ ! -L "${WORKTREE_PATH}/.claude" ]; then
    ln -s "${GIT_ROOT}/.claude" "${WORKTREE_PATH}/.claude"
fi

# Copy CLAUDE.md
if [ -f "${GIT_ROOT}/CLAUDE.md" ] && [ ! -f "${WORKTREE_PATH}/CLAUDE.md" ]; then
    cp "${GIT_ROOT}/CLAUDE.md" "${WORKTREE_PATH}/CLAUDE.md"
fi
```

If worktree or branch already exists → attempt to resume (check for prior commits on the branch).

### Step 2: Dispatch Agents

Follow `.claude/skills/orchestrator/agent-dispatcher.md`:

- **Classic mode**: Batch tasks (max MAX_PARALLEL), launch via Agent tool with `run_in_background=true`, wait for batch, next batch.
- **Team mode**: Create team, launch lead + teammates, wait for completion.

Each agent prompt includes:
- Working directory = WORKTREE_PATH
- Task description + acceptance criteria
- Cheat sheets for relevant tech
- Knowledge store query instructions (rlm_search) if research artifacts exist
- Project conventions

### Step 3: Test

```bash
cd "$WORKTREE_PATH"
```

Run project tests. Use `Skill: orbit` with `Args: test` if Orbit is initialized, otherwise run the project's native test command.

**Retry loop (max 2):**
If tests fail → dispatch fix agent with failure output → re-test.

If still failing after retries → record failure and return `needs_user_input`.

### Step 4: Test Quality Review

After tests pass, run test quality review:
```
Skill: test-review
Args: local
```

P0/P1 gaps → dispatch test-writing agent → re-test → re-review (max 1 cycle).
P2/P3 → log as deferred, continue.

### Step 5: Code Review

```
Skill: code-review-pro
Args: local
```

P0/P1 → dispatch fix agent → re-review (max 1 cycle).
P2/P3 → log as deferred, continue.

### Step 6: Performance Review

```
Skill: perf-review
Args: local
```

P0/P1 → dispatch fix agent → re-review (max 1 cycle).
P2/P3 → log as deferred, continue.

### Step 7: Commit

```bash
cd "$WORKTREE_PATH"
git add -A
```

Standard mode commit:
```bash
git commit -m "orchestrate(phase-{N}): {phase.title}

Co-Authored-By: Claude <noreply@anthropic.com>"
```

Spec-driven mode commit:
```bash
git commit -m "orchestrate(phase-{N}/sprint-{M}): {spec.name} — {spec.overview}

Spec: docs/plans/specs/{spec.name}-spec.md
Requirements completed: {REQ-1, REQ-2, ...}

Co-Authored-By: Claude <noreply@anthropic.com>"
```

If multiple logical changes → use commit-split.

Record the commit hash(es).

### Step 8: Merge + Cleanup

```bash
cd "$GIT_ROOT"
git merge "$BRANCH_NAME" --no-ff -m "Merge orchestrate phase ${PHASE_NUM}: {phase.title}"
```

If merge conflict → do NOT auto-resolve. Return `needs_user_input` with conflict details.

On successful merge:
```bash
git worktree remove "$WORKTREE_PATH" --force
git branch -d "$BRANCH_NAME"
git worktree prune
```

### Step 8.5: Update Handoff and Progress Log

#### 8.5a. Write handoff.md

If HANDOFF_PATH is not null, overwrite the file with current state:

```markdown
# Handoff: {project.name from WORKFLOW_CONFIG or inferred}

## Project Context
- Type: {from WORKFLOW_CONFIG.project.type or "unknown"}
- Build: {from WORKFLOW_CONFIG.project.build_cmd or ""}
- Test: {from WORKFLOW_CONFIG.project.test_cmd or ""}

## Architecture Decisions
{carry forward ALL existing decisions from Step 0 ARCH_DECISIONS}
{append any NEW architecture/design decisions made during this phase, prefixed with "Phase N:"}

## Last Completed Phase
name: {this phase title}
session: {session number — increment from handoff read or start at 1}
status: {completed or failed}
summary: {2-3 sentence summary of what this phase accomplished}
key_files:
{list files created or significantly modified, from git diff --name-only}
decisions:
{list architecture/design choices made during this phase}

## Current Phase
name: {next phase title from manifest, or "none — all phases complete"}
number: {next phase number, or N/A}
status: ready
criteria:
{next phase acceptance criteria from manifest, or "N/A — orchestration complete"}

## Workspace State
branch: {current branch after merge, from git branch --show-current}
last_commit: {merge commit short hash, from git rev-parse --short HEAD}
uncommitted_changes: none
build_status: {passing or failing, from Step 3 test result}
```

**Size guard:** After writing, check `wc -l HANDOFF_PATH`. If >150 lines, truncate the Architecture Decisions section to keep only the 10 most recent entries, then re-write.

If HANDOFF_PATH is null, skip this step entirely (backward compatible).

#### 8.5b. Append to progress-log.md

If PROGRESS_LOG_PATH is not null, prepend a session entry at the TOP of the `## Sessions` section:

```markdown
### Session {N} — Phase {phase.number}: {phase.title}
- **started**: {start timestamp from Step 1}
- **attempt**: {ATTEMPT_NUMBER}
- **status**: {completed | failed | blocked}
- **duration**: {approximate duration}
- **completed**:
{checklist of acceptance criteria: [x] for met, [ ] for unmet}
- **files_changed**:
{from git diff --name-status of the phase commits}
- **commits**: {commit hashes from Step 7-8}
- **blockers**: {none or description}
- **next**: {what the next phase should pick up, or "all phases complete"}
```

To prepend: read the file, find `## Sessions`, insert the entry after that heading (before existing entries), write back.

If PROGRESS_LOG_PATH is null, skip (backward compatible).

#### 8.5c. Enforcement check

```
if WORKFLOW_CONFIG and WORKFLOW_CONFIG.execution.progress_enforcement == "strict":
    Verify HANDOFF_PATH was written (file exists, stat shows recent mtime).
    If not written → set HANDOFF_UPDATED = false (reported in PHASE_RESULT)

if enforcement == "warn":
    Log if handoff wasn't updated but don't affect status.

if enforcement == "none" or WORKFLOW_CONFIG is null:
    Skip check.
```

### Step 9: Documentation Refresh

After the merge lands on the main branch, update project documentation for the files
this phase touched. Run from `GIT_ROOT` (main branch, post-merge).

#### 9a. Mercator Codebase Map

Check if the codebase map needs structural updates:

```bash
MANIFEST="docs/.mercator.json"
if [ -f "$MANIFEST" ]; then
  # Check for structural changes (new/removed modules, not just edits)
  DIFF=$(python3 "$SCANNER" . --diff "$MANIFEST" 2>/dev/null)
  ADDED=$(echo "$DIFF" | jq '.added | length' 2>/dev/null || echo "0")
  REMOVED=$(echo "$DIFF" | jq '.removed | length' 2>/dev/null || echo "0")
fi
```

| Condition | Action |
|-----------|--------|
| No manifest + first phase | `Skill: mercator-ai` (create initial map) |
| Manifest exists, 0 added/removed | Skip — only file edits, prose is still accurate |
| Manifest exists, added > 0 OR removed > 0 | `Skill: mercator-ai` with `Args: --diff` (update mode) |

If mercator is not installed or the scan command fails, skip with a note in the PHASE_RESULT.

#### 9b. Chronicler Tech Docs

Check if Chronicler is initialized for this project, then update `.tech.md` files
for source files this phase created or modified.

```bash
# Check if Chronicler is active
if [ -d ".chronicler" ] && [ -f ".chronicler/INDEX.md" ]; then
    CHRONICLER_ACTIVE=true
else
    CHRONICLER_ACTIVE=false
fi
```

**If Chronicler is active:**

```
Skill: chronicler:regenerate
```

This scans for stale `.tech.md` files (source changed since last doc generation)
and regenerates them. It handles:
- New files created by this phase → generates new `.tech.md`
- Modified files → regenerates their `.tech.md` with updated content
- Deleted files → marks their `.tech.md` as orphaned

**If Chronicler is NOT active but the project has >10 source files:**

Skip — don't initialize Chronicler mid-orchestration. Note it in the PHASE_RESULT
`docs_updated` field so the user can run `/chronicler:init` afterward if they want.

**If Chronicler is NOT active and the project is small (<10 files):**

Skip entirely. Small projects don't benefit enough from per-file docs.

#### 9c. Cost Guard

Both Mercator and Chronicler run on Sonnet via subagents, so they don't eat the
phase-runner's own context. But they do cost tokens:

- Mercator `--diff` mode: ~5-15k tokens (only re-explores changed modules)
- Chronicler regenerate: ~2-5k tokens per changed file

For phases that touched >15 files, the Chronicler step could be expensive. In that case,
batch the regeneration to only the 10 most important files (sorted by: new files first,
then files with the most lines changed). Log the skipped files in the PHASE_RESULT so
the user can run a full `/chronicler:regenerate` later.

### Step 10: Update Progress

Write to `{PLAN_DIR}/progress.md` (append):

```markdown
### Phase {N}: {title}
- **Status:** completed
- **Tests:** passed ({X passing})
- **Review:** {clean/issues} (P0: 0, P1: 0, P2: {n}, P3: {n})
- **Commits:** {sha_list}
- **Docs:** {mercator: updated|skipped|n/a, chronicler: {N} files regenerated|skipped|not initialized}
- **Deferred:** {P2/P3 list or "none"}
```

If spec-driven, also update manifest.md status and spec frontmatter.

## Return Format

The phase-runner MUST end its response with this exact structure (the orchestrator parses it):

```
## PHASE_RESULT
- status: completed|failed|needs_user_input
- phase: {N}
- title: {phase.title}
- commits: {comma-separated sha list, or "none"}
- tests: passed|failed
- test_count: {X passing, Y failing}
- review: clean|p2_p3_only|p0_p1_found
- deferred_items: {count}
- mercator: updated|skipped|not_installed|error
- chronicler: {N}_files_regenerated|skipped|not_initialized|error
- chronicler_skipped_files: {list if >15 files touched and some skipped, otherwise "none"}
- error: {description if failed or needs_user_input, otherwise "none"}
- conflict_files: {list if merge conflict, otherwise "none"}
- duration: {approx minutes}
- handoff_updated: true|false|skipped    # NEW — whether handoff.md was written
- progress_log_updated: true|false|skipped  # NEW — whether progress-log.md was appended
- architecture_decisions: {count}         # NEW — number of new decisions added this phase
```

This is the ONLY part the main orchestrator reads. Everything else (test output, review details, agent logs) stays inside this agent's context and is discarded when the agent completes.

## Error Handling

| Scenario | Action |
|----------|--------|
| Worktree creation fails | Return failed + error description |
| All agents fail | Return failed + error description |
| Tests fail after 2 retries | Return needs_user_input + test failure summary |
| P0/P1 persists after fix | Return needs_user_input + finding summary |
| Merge conflict | Return needs_user_input + conflict file list |
| Agent timeout | Log, continue with other agents |

The phase-runner never asks the user directly (it's a subagent). It returns structured status and lets the orchestrator handle user interaction.

## Context Budget

Typical phase-runner context usage:
- Agent prompt: ~2-3k tokens
- Worktree setup: ~500 tokens
- Agent dispatch (2-3 agents): ~3-5k tokens per agent result
- Test output: ~2-5k tokens
- Review results: ~3-5k tokens each (test-review, code-review, perf-review)
- Commit/merge: ~500 tokens
- Doc refresh (mercator + chronicler): ~1-2k tokens (skill invocations + results)

Total: ~22-37k tokens for a full phase — well within a single agent's context window.

Mercator and Chronicler both delegate to Sonnet subagents, so the heavy doc generation
work happens outside the phase-runner's context. Only the invocation and result summary
hit the phase-runner's budget.

If a phase has >5 tasks or very verbose test output, the agent may approach 50-60% context. This is fine — it completes and returns. The main orchestrator never sees any of it.
