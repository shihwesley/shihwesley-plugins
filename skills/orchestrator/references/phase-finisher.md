---
name: phase-finisher
description: Post-dev pipeline per phase — Orbit test → code-review-pro → incremental commit → merge
---

# Phase Finisher

Runs after all agents in a phase complete. Chains: testing → review → commit → merge → cleanup.

## Input

- Phase manifest with completed agent results
- Worktree path for this phase
- Project context (tech stack, conventions)

## Pipeline

### Step 1: Test via Orbit

After all agents complete, run tests in the phase worktree.

```
# Switch to worktree
cd {worktreePath}

# Run Orbit test (isolated Docker sandbox)
Skill: orbit
Args: test
```

**What happens:**
- Orbit detects the project type (node/python/go/rust)
- Builds Docker image with the worktree's code
- Runs test suite in isolation
- Reports pass/fail with duration

**If tests pass → proceed to Step 2**

**If tests fail:**

```
RETRY_COUNT = 0
MAX_RETRIES = 2

while tests_fail and RETRY_COUNT < MAX_RETRIES:
    RETRY_COUNT += 1

    # Dispatch a fix agent with the test failure output
    Task(
        subagent_type = phase.agentType,  # same skills as the phase
        prompt = """
        Tests failed in phase {N} worktree.

        Test output:
        {test_failure_output}

        Working directory: {worktreePath}
        Phase skills: {phase.skills}

        Fix the failing tests. Do not change test expectations unless
        the test is clearly wrong — fix the implementation code.
        Report what you changed.
        """
    )

    # Re-run tests
    Skill: orbit
    Args: test

if still_failing:
    # Ask user
    AskUserQuestion(
        question="Tests still failing after 2 fix attempts. How to proceed?",
        options=[
            "Skip tests, continue to review",
            "Fix manually (pause orchestration)",
            "Abort this phase"
        ]
    )
```

### Step 2: Code Review via code-review-pro

After tests pass, review the worktree's changes.

```
# In the worktree directory
Skill: code-review-pro
Args: local
```

**What happens:**
- code-review-pro launches up to 7 parallel agents (SOLID, Security, Quality, Git History, etc.)
- Scores findings with confidence levels
- Filters to P0-P3 severity

**If clean (no P0/P1) → proceed to Step 3**

**If P0/P1 issues found:**

```
REVIEW_RETRY = 0
MAX_REVIEW_RETRIES = 1

if has_p0_or_p1:
    REVIEW_RETRY += 1

    # Dispatch fix agent with review findings
    Task(
        subagent_type = phase.agentType,
        prompt = """
        Code review found critical issues in phase {N}:

        P0 (Critical):
        {p0_findings}

        P1 (High):
        {p1_findings}

        Working directory: {worktreePath}

        Fix these issues. Focus on P0 first, then P1.
        Do NOT change code that wasn't flagged.
        Report what you fixed.
        """
    )

    # Re-run code review
    Skill: code-review-pro
    Args: local

    if still_has_p0_p1 and REVIEW_RETRY >= MAX_REVIEW_RETRIES:
        AskUserQuestion(
            question="P0/P1 issues persist after fix attempt. How to proceed?",
            options=[
                "Fix manually (pause orchestration)",
                "Accept and commit anyway",
                "Abort this phase"
            ]
        )
```

**P2/P3 handling:**
- Log P2/P3 issues as follow-ups in progress.md
- Do NOT block the commit
- Include in the phase summary for user awareness

### Step 3: Incremental Commit

After review passes, commit the phase's work.

```bash
cd {worktreePath}

# Check what changed
git status
git diff --stat
```

**Single logical change:**
```bash
git add -A
git commit -m "orchestrate(phase-{N}): {phase.title}

- {task1.title}: {brief summary}
- {task2.title}: {brief summary}

Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>"
```

**Multiple logical changes (>1 distinct feature/module):**
```
# Use commit-split for atomic commits
Skill: commit-split
```

This will analyze the changes and create separate commits per logical unit.

**Spec-driven mode commit message:**
```bash
git commit -m "orchestrate(phase-{N}/sprint-{M}): {spec.name} — {spec.overview}

Spec: docs/plans/specs/{spec.name}-spec.md
Requirements completed: {REQ-1, REQ-2, ...}

Co-Authored-By: Claude <noreply@anthropic.com>"
```

The spec name in the commit message enables `git log --grep` resume recovery.
The `Spec:` trailer provides direct file traceability.

### Step 4: Merge Worktree

After commit(s), merge back to the main branch.

Follow `.claude/skills/orchestrator/worktree-manager.md` merge procedure:

```bash
# Return to main working directory
cd {gitRoot}

# Get the source branch name
MERGE_TARGET=$(git branch --show-current)

# Merge
git merge "{phase.worktreeBranch}" --no-ff \
    -m "Merge orchestrate phase {N}: {phase.title}"
```

**Conflict handling:**
- Do NOT auto-resolve
- Report conflicting files
- Ask user: "Merge conflict. Resolve manually / skip merge / abort?"

### Step 5: Cleanup + Report

```bash
# Remove worktree
git worktree remove "{worktreePath}" --force

# Delete branch (it's merged)
git branch -d "{phase.worktreeBranch}"

# Prune
git worktree prune
```

**Update progress.md:**
```markdown
### Phase {N}: {title}
- **Status:** completed
- **Tests:** {pass/fail} ({X passing, Y failing})
- **Review:** {clean/issues} (P0: {n}, P1: {n}, P2: {n}, P3: {n})
- **Fix attempts:** tests={retry_count}, review={review_retry_count}
- **Commits:** {commit_sha_list}
- **P2/P3 deferred:** {list or "none"}
- **Duration:** {time from worktree create to cleanup}
```

**If spec-driven mode** (manifest.md exists in plan directory), additional updates:

**Update each completed spec's frontmatter:**
```bash
# For each spec completed in this phase:
# Edit: docs/plans/specs/{spec-name}-spec.md
# Change frontmatter: status: in_progress → status: completed
```

**Update manifest.md status column:**
```bash
# Edit: docs/plans/manifest.md
# In the Phase/Sprint/Spec Map table, change the spec's Status to "completed"
# This is a single-line edit per spec
```

**Spec-aware progress.md entry** (replaces the standard entry above when in spec-driven mode):
```markdown
### Phase {N}, Sprint {M}: {spec-name}
- **Status:** completed
- **Spec:** docs/plans/specs/{spec-name}-spec.md
- **Requirements completed:** REQ-1, REQ-2, ...
- **Tests:** {pass/fail} ({X passing, Y failing})
- **Review:** {clean/issues} (P0: {n}, P1: {n}, P2: {n}, P3: {n})
- **Fix attempts:** tests={retry_count}, review={review_retry_count}
- **Commits:** {commit_sha_list}
- **Duration:** {time from worktree create to cleanup}
```

Note: progress.md entries are keyed by spec name in spec-driven mode
(not just phase number). This makes git-log cross-referencing unambiguous
and ensures `/orchestrate --resume` can match progress entries to specs.

**Update TaskUpdate:**
```
TaskUpdate(taskId="{phase_task_id}", status="completed")
```

### Step 6: Phase Transition

After cleanup:
1. Check if there's a next phase → proceed
2. If this was the last phase → run final summary
3. If any P2/P3 were deferred → remind user at the end

**Final summary (after all phases):**
```markdown
## Orchestration Complete

### Results
| Phase | Status | Tests | Review | Commits |
|-------|--------|-------|--------|---------|
| 1 | completed | passed | clean | abc1234 |
| 2 | completed | passed | 2 P2 | def5678, ghi9012 |
| 3 | skipped | — | — | — |

### Deferred Items
- Phase 2: P2 — "Consider extracting helper function" (src/utils.ts:45)
- Phase 2: P3 — "Variable naming could be clearer" (src/models.ts:12)

### Installed Skills (new this session)
- swift-networking (from github.com/example/skills)

### Next Steps
1. Review deferred P2/P3 items
2. Run full test suite: /orbit test
3. Create PR: /commit or manual
```

## Error Recovery Summary

| Scenario | Auto-retry | Max retries | Escalation |
|----------|-----------|-------------|------------|
| Test failure | Yes | 2 | Ask user |
| P0/P1 in review | Yes | 1 | Ask user |
| P2/P3 in review | No (log only) | — | Deferred |
| Merge conflict | No | — | Ask user |
| Agent failure | No | — | Logged, continue |
| Orbit unavailable | No | — | Skip test, warn user |

## Notes

- The phase finisher is the **quality gate** — nothing gets committed without passing tests and review
- If Orbit is not initialized for this project → warn and ask whether to skip testing or init Orbit first
- Each step is independently recoverable — if review fails but tests passed, you don't need to re-run tests
- The finisher runs inside the **phase-runner subagent's** context, not the main orchestrator's context. This means all test output, review findings, and fix agent results stay contained in the phase-runner and are discarded after the phase completes. Only the PHASE_RESULT summary surfaces to the main orchestrator.

## Agent Teams Mode Note

The phase finisher is **mode-agnostic** — it receives the same aggregated result format regardless of whether agents were dispatched via classic Task-based dispatch or Agent Teams.

When team mode is active:
- The team lead collects and aggregates results before passing them to the finisher
- The finisher does NOT need to know about teammates, the lead, or team coordination
- Fix agents dispatched by the finisher (Steps 1-2) use classic `Task` dispatch even in team mode — spinning up a team for a single fix agent is unnecessary overhead
