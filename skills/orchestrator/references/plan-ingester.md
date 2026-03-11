---
name: plan-ingester
description: Parses interactive-planning output files into structured phase manifests
---

# Plan Ingester

Reads plan files from a directory and extracts structured phase data.

## Input

A plan directory containing EITHER:

**Task-based mode** (detected by `task_plan.md`):
- `task_plan.md` — phases, tasks, deliverables
- `findings.md` — decisions, requirements, approach
- `progress.md` — current status, session history
- `workflow.md` — execution contract (optional, from /interactive-planning)

**Spec-driven mode** (detected by `manifest.md`):
- `manifest.md` — index of all specs, phases, sprints, dependency DAG
- `specs/*.md` — individual spec files with requirements + tasks
- `findings.md` — decisions, requirements, approach
- `progress.md` — current status, session history
- `workflow.md` — execution contract (optional, from /interactive-planning)

Plus project-level docs (auto-discovered):
- `CLAUDE.md` (root + any nested)
- `docs/` folder (all .md files)
- `docs/CODEBASE_MAP.md` if it exists
- Any `README.md` files

## Process

Two modes: **Full Ingest** (fresh runs) and **Resume Ingest** (`--resume` flag).

---

### Full Ingest (default)

#### Step 1: Locate Plan Files + Detect Mode

```
plan_dir = ARGUMENTS or current directory

# CRITICAL: Resolve to absolute path. Glob/Read tools do NOT expand ~.
# If plan_dir contains ~ or is relative, resolve it:
#   Bash("realpath {plan_dir}")
# If plan_dir == CWD, omit the path parameter from Glob calls entirely.

# Quick inventory — one ls call to see what exists before any Glob/Read:
Bash("ls {plan_dir}")

# Mode detection — check manifest.md FIRST
if exists({plan_dir}/manifest.md):
    MODE = "spec-driven"
    Read: {plan_dir}/manifest.md
    # Parse the Phase/Sprint/Spec Map table to get spec list
    # For each spec in the table:
    #   Read: {plan_dir}/specs/{spec-name}-spec.md
elif exists({plan_dir}/task_plan.md):
    MODE = "task-based"
    Read: {plan_dir}/task_plan.md
else:
    ERROR: "No manifest.md or task_plan.md found in {plan_dir}"
    STOP

Read: {plan_dir}/findings.md (if exists)
Read: {plan_dir}/progress.md (if exists)

# Check for workflow contract (Symphony-inspired execution config)
if exists({plan_dir}/workflow.md):
    Read: {plan_dir}/workflow.md
    WORKFLOW = parse_yaml_frontmatter(workflow.md)
    WORKFLOW_PROMPT = parse_markdown_body(workflow.md)
    # Extract execution config
    EXECUTION_CONFIG = WORKFLOW.execution
    PROMPT_TEMPLATES = {
        phase: section matching "## Phase Prompt",
        continuation: section matching "## Continuation Prompt"
    }
else:
    WORKFLOW = null
    EXECUTION_CONFIG = {
        phase_runner: "subagent",
        max_phase_retries: 2,
        continuation_prompt: true,
        progress_enforcement: "strict",
        gates: { between_phases: "auto", on_failure: "retry" }
    }
    PROMPT_TEMPLATES = null
```

#### Step 2: Discover Project Docs

From the git root (or cwd if not a git repo):
```
Glob: **/CLAUDE.md
Glob: docs/**/*.md
Glob: **/README.md
Glob: docs/CODEBASE_MAP.md
```

Read each discovered doc. Extract:
- Tech stack indicators (languages, frameworks, tools)
- Architecture patterns mentioned
- Existing conventions or constraints
- Module/directory structure

#### Step 3: Parse Phases from task_plan.md

Extract each `## Phase N:` section. For each phase, capture:
- Phase number and title
- Description text
- Task list (numbered items under `### Tasks`)
- Files to create/modify (under `### Files to Create`)
- Acceptance criteria
- Dependencies (look for "blocked by", "depends on", "after Phase X")

#### Step 3a: Parse Specs from manifest.md (spec-driven mode only)

If MODE == "spec-driven":

1. From manifest.md, extract the Phase/Sprint/Spec Map table rows
2. For each row, read the corresponding spec file: `{plan_dir}/specs/{spec-name}-spec.md`
3. From each spec's YAML frontmatter, extract: name, phase, sprint, parent, depends_on, status
4. From each spec's markdown body, extract: requirements, acceptance criteria, tasks, files
5. Group specs by phase number
6. Build phase objects where each phase contains the specs (and their tasks) for that phase

Output format is the SAME as task-based mode — the flattened tasks and acceptanceCriteria arrays match exactly:
```json
[
  {
    "phase": 1,
    "title": "Phase 1",
    "mode": "spec-driven",
    "specs": [
      {
        "name": "data-model-spec",
        "sprint": 1,
        "tasks": [...],
        "files": [...],
        "dependsOn": ["root-spec"],
        "status": "draft"
      }
    ],
    "tasks": [...all tasks from all specs in this phase, flattened...],
    "acceptanceCriteria": [...all AC from all specs...],
    "dependsOn": [...phase-level deps derived from spec deps...]
  }
]
```

The `specs` array is additional metadata for spec-driven mode. The flattened `tasks` and `acceptanceCriteria` arrays ensure downstream consumers (skill-matcher, agent-dispatcher) work without changes.

#### Step 4: Enrich with Findings

From `findings.md`, extract:
- Priority (speed/quality/flexibility/simplicity)
- Approach chosen
- Technical decisions table
- Requirements list

Apply to phase manifests:
- Tag phases with relevant decisions
- Note which requirements each phase addresses

#### Step 5: Output

Return array of raw phase objects:
```json
[
  {
    "phase": 1,
    "title": "Setup and Configuration",
    "description": "...",
    "tasks": [
      {"id": "1.1", "title": "...", "description": "...", "files": ["path/to/file"]},
      {"id": "1.2", "title": "...", "description": "...", "files": []}
    ],
    "acceptanceCriteria": ["..."],
    "dependsOn": [],
    "rawText": "full markdown of this phase section"
  }
]
```

Plus a context object:
```json
{
  "projectDocs": ["list of doc paths read"],
  "techStack": ["typescript", "swift"],
  "conventions": ["...extracted from CLAUDE.md..."],
  "architecture": "...from CODEBASE_MAP...",
  "workflow": null or {
    "project": { "name": "...", "type": "...", "build_cmd": "...", "test_cmd": "..." },
    "execution": {
      "phase_runner": "subagent",
      "max_phase_retries": 2,
      "continuation_prompt": true,
      "progress_enforcement": "strict",
      "gates": { "between_phases": "auto", "on_failure": "retry" }
    },
    "handoff": { "file": ".claude/handoff.md", "max_lines": 150 },
    "progress": { "file": ".claude/progress-log.md", "mode": "append_only" },
    "templates": { "phase": "...", "continuation": "..." }
  }
}
```

---

### Resume Ingest (--resume)

When resuming, the goal is to read the minimum needed to identify where to pick up.
The TLDR hook gives you a table of contents with line numbers and status markers
for free — use that as the navigation index instead of reading full files.

#### Step R1: Read progress.md via TLDR (navigation pass)

Attempt a plain `Read: {plan_dir}/progress.md` — the TLDR hook will intercept and return:
- Table of Contents with line numbers for each `### Phase` heading
- Key Terms showing `**Status:** completed/pending/in_progress` with line numbers

From this summary, extract:
```
COMPLETED_PHASES = [phase entries where Status == completed]
LAST_COMPLETED_LINE = line number of the last completed phase heading
RESUME_PHASE = first phase with Status != completed (or no status entry = never started)
RESUME_LINE = line number of RESUME_PHASE heading (or last completed + 1)
```

#### Step R2: Targeted read of progress.md tail

Read only from the resume point to get context about the last completed phase
and whatever was in progress:
```
Read: {plan_dir}/progress.md (offset: LAST_COMPLETED_LINE, limit: 60)
```

This captures:
- What the last phase did (actions, commits, files modified)
- Any learnings or gotchas logged (the "5-Question Reboot Check" if present)
- The state of the in-progress phase (if partially done)

#### Step R2.5: Read handoff.md (cross-reference actual state)

If the plan has a workflow.md with a handoff file configured:
```
HANDOFF_PATH = resolve(WORKFLOW.handoff.file, git_root)
if exists(HANDOFF_PATH):
    Read: HANDOFF_PATH  # Always small (<150 lines), TLDR won't trigger
    # Extract:
    #   - last_completed_phase.name, summary, key_files
    #   - workspace_state.branch, last_commit, build_status
    #   - architecture_decisions (carry forward to next phase)
    # Cross-reference with progress.md:
    #   - If handoff says Phase 2 completed but progress.md says Phase 1 was last
    #     → handoff is more recent (progress.md may be stale). Trust handoff.
    #   - If progress.md is more recent → trust progress.md (handoff may not have been written)
```

Without workflow.md, skip this step (backward compatible).

#### Step R3: Read next work unit

**If MODE == "task-based"** (existing behavior):

Use TLDR on task_plan.md to get its ToC with line numbers:
```
Read: {plan_dir}/task_plan.md  → TLDR gives ToC with phase line numbers
```

Find the line range for RESUME_PHASE onward:
```
PHASE_START = line number of "## Phase {RESUME_PHASE}" in ToC
PHASE_END = line number of next major section after last pending phase (or EOF)
Read: {plan_dir}/task_plan.md (offset: PHASE_START, limit: PHASE_END - PHASE_START)
```

Parse only these remaining phases.

**If MODE == "spec-driven"** (fast path):

This is where spec-driven planning pays off. Instead of TLDR-scanning and
offset-reading a large plan file, do 3 small reads:

```
# 1. Read manifest.md (small file, typically <50 lines)
Read: {plan_dir}/manifest.md

# 2. Parse the Phase/Sprint/Spec Map table
#    Cross-reference with progress.md status from Step R1/R2:
#    - Specs marked completed in progress.md → skip
#    - Specs marked in_progress → resume
#    - Specs with status=draft AND all depends_on completed → ready
NEXT_SPEC = first spec that is (in_progress OR (draft AND unblocked))

# 3. Read ONLY the next spec file
Read: {plan_dir}/specs/{NEXT_SPEC}-spec.md
```

That's it. Three reads, all small files. Total context budget: ~110-140 lines.

**Git-log cross-reference** (fallback when progress.md is stale or missing):

```bash
# Check what specs have been committed and merged
git log --oneline --grep="orchestrate(" --format="%s"
```

This returns spec names from commit messages like:
  `orchestrate(phase-1/sprint-1): data-model-spec — schema and migrations`

Compare against manifest to find the first uncommitted, unblocked spec.
This handles the case where progress.md wasn't updated (crash, /clear
before finisher wrote to progress, etc.). Git log can't lie about what
was actually committed.

**Priority:** Use progress.md as primary source. Fall back to git log only if
progress.md is missing, empty, or doesn't have status entries for specs that
appear in committed git history.

#### Step R4: Skip findings.md and project docs

- `findings.md` was already consumed during the first run's review stage. Skip it.
- Project docs (CLAUDE.md, CODEBASE_MAP, docs/) are stable between phases. Skip them.
- **Exception:** If progress.md tail mentions new files created by previous phases
  (look for "Files created:" or "Files modified:" entries), read only those specific
  new files if they're docs that could affect subsequent phases.
- **Spec-driven additional skips:**
  - Skip all spec files except NEXT_SPEC
  - Do NOT re-read completed specs — their work is already committed
  - If NEXT_SPEC has a "Needs from" section referencing specific types/interfaces
    from a completed dependency spec, that dependency's "Provides to" section was
    already fulfilled in code. The agent working on NEXT_SPEC will find those
    types in the actual codebase, not in the spec file.

#### Step R5: Output (filtered)

Return the same format as Full Ingest, but:
- Phase array contains only pending/in_progress phases
- Context object is minimal (no full project doc re-scan):

```json
{
  "resumeFrom": {"phase": 3, "title": "AI Drafter"},
  "completedPhases": [1, 1.5, 2],
  "lastCompletedContext": "...tail of progress.md with learnings...",
  "phases": [
    {
      "phase": 3,
      "title": "AI Drafter",
      "tasks": [...],
      "status": "pending"
    }
  ],
  "projectDocs": [],
  "docsSkipped": true,
  "workflow": WORKFLOW or null,
  "skipReason": "resume mode — docs unchanged since initial run"
}
```

**Spec-driven resume output:**

```json
{
  "mode": "spec-driven",
  "resumeFrom": {"spec": "api-spec", "phase": 1, "sprint": 2},
  "completedSpecs": ["root-spec", "data-model-spec", "auth-spec"],
  "nextSpec": {
    "name": "api-spec",
    "phase": 1,
    "sprint": 2,
    "tasks": [...],
    "files": [...],
    "acceptanceCriteria": [...],
    "dependsOn": ["data-model-spec", "auth-spec"],
    "dependenciesMet": true
  },
  "phases": [
    {
      "phase": 1,
      "title": "Phase 1",
      "mode": "spec-driven",
      "specs": [{"name": "api-spec", "sprint": 2, ...}],
      "tasks": [...flattened from api-spec only...]
    }
  ],
  "projectDocs": [],
  "docsSkipped": true,
  "workflow": WORKFLOW or null,
  "skipReason": "resume mode — manifest + single spec read only"
}
```

Key difference from task-based resume: the `phases` array contains only the
specs that are actionable right now, not all remaining phases. After this spec
completes and the orchestrator resumes again, it reads the NEXT spec.
Each resume cycle's context budget stays at one spec file.

---

## Notes

- Do NOT restructure or judge the plan — that's the Plan Reviewer's job
- Do NOT resolve skills — that's the Skill Matcher's job
- Pure extraction and structuring only
- If neither manifest.md nor task_plan.md exists, report error and stop
- On resume, trust the TLDR ToC as your index — don't read past it unless you need content
