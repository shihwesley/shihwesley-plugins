---
name: worktree-manager
description: Creates, manages, and cleans up git worktrees for orchestrated phase execution
---

# Worktree Manager

Handles the lifecycle of git worktrees during `/orchestrate` execution. Each phase gets an isolated worktree.

## Prerequisites

- Must be in a git repository
- Git 2.15+ (worktree support)
- No uncommitted changes on current branch (warn if present)

## Worktree Lifecycle

### Create (before phase starts)

```bash
# 1. Determine paths
GIT_ROOT=$(git rev-parse --show-toplevel)
PROJECT_NAME=$(basename "$GIT_ROOT")
PHASE_NUM={phase.phase}
PHASE_SLUG=$(echo "{phase.title}" | tr '[:upper:]' '[:lower:]' | tr ' ' '-' | tr -cd 'a-z0-9-' | head -c 30)
BRANCH_NAME="orchestrate/phase-${PHASE_NUM}-${PHASE_SLUG}"
WORKTREE_PATH="${GIT_ROOT}/../${PROJECT_NAME}-phase-${PHASE_NUM}"

# 2. Check for conflicts
if git worktree list | grep -q "$WORKTREE_PATH"; then
    echo "Worktree already exists at $WORKTREE_PATH"
    echo "Options: resume existing worktree, or remove and recreate"
    # Ask user
fi

if git branch --list "$BRANCH_NAME" | grep -q .; then
    echo "Branch $BRANCH_NAME already exists"
    echo "Options: use existing branch, or delete and recreate"
    # Ask user
fi

# 3. Create
git worktree add "$WORKTREE_PATH" -b "$BRANCH_NAME"
```

### Setup (after create, before agents start)

The worktree needs context files for agents to work properly:

```bash
# Symlink .claude directory so agents have access to skills/commands
# (worktrees share the same .git, but .claude is project-level)
if [ ! -L "${WORKTREE_PATH}/.claude" ] && [ -d "${GIT_ROOT}/.claude" ]; then
    ln -s "${GIT_ROOT}/.claude" "${WORKTREE_PATH}/.claude"
fi

# Copy CLAUDE.md if it exists (agents read this for conventions)
if [ -f "${GIT_ROOT}/CLAUDE.md" ] && [ ! -f "${WORKTREE_PATH}/CLAUDE.md" ]; then
    cp "${GIT_ROOT}/CLAUDE.md" "${WORKTREE_PATH}/CLAUDE.md"
fi
```

**Important**: Worktrees share the same `.git` directory, so:
- Commits in a worktree are visible from the main tree
- Branch operations affect the shared repo
- File changes are isolated to the worktree's working directory

### Verify (health check before dispatching agents)

```bash
# Verify worktree is valid
cd "$WORKTREE_PATH"
git status  # should show clean worktree on the new branch

# Verify branch
CURRENT=$(git branch --show-current)
if [ "$CURRENT" != "$BRANCH_NAME" ]; then
    echo "ERROR: Worktree not on expected branch"
    exit 1
fi
```

### Merge (after phase passes test + review)

```bash
# From the main working tree (not the worktree)
cd "$GIT_ROOT"

# Determine merge target (the branch we started from)
MERGE_TARGET=$(git branch --show-current)

# Merge the phase branch
git merge "$BRANCH_NAME" --no-ff -m "Merge orchestrate phase ${PHASE_NUM}: ${phase.title}"
```

**Conflict handling:**
- If merge conflicts occur → do NOT auto-resolve
- Report the conflicting files to the user
- Ask: "Merge conflict in phase N. Resolve manually, skip phase, or abort orchestration?"
- If user resolves → continue
- If user skips → leave branch unmerged, proceed to next phase

### Cleanup (after successful merge)

```bash
# Remove worktree
git worktree remove "$WORKTREE_PATH" --force

# Delete the phase branch (it's been merged)
git branch -d "$BRANCH_NAME"

# Prune stale worktree entries
git worktree prune
```

**Cleanup on failure:**
- If phase fails and user aborts → still remove worktree but keep branch
- Branch can be manually reviewed or resumed later
- `git worktree remove "$WORKTREE_PATH" --force` (force needed if dirty)

## State Tracking

The worktree manager maintains state in a temporary file:

```json
// /tmp/claude-orchestrate-{session-id}/worktrees.json
{
  "sessionId": "...",
  "gitRoot": "/path/to/project",
  "phases": {
    "1": {
      "branch": "orchestrate/phase-1-setup",
      "worktreePath": "/path/to/project-phase-1",
      "status": "active|merged|failed|cleaned",
      "createdAt": "2026-02-05T12:00:00Z"
    }
  }
}
```

This allows resume support — if orchestration is interrupted, the state file shows which worktrees exist and their status.

## Integration with Worktree Enforcer

Your existing `worktree-enforcer.sh` hook blocks >3 file edits on main/master. The orchestrator's worktrees are on feature branches, so this hook won't trigger. However:

- Agents working in worktrees are on `orchestrate/phase-N-*` branches — safe
- The merge step happens on the main branch — but it's a git merge command, not file edits via Edit/Write tools, so the hook doesn't fire

## Error Scenarios

| Error | Recovery |
|-------|----------|
| `git worktree add` fails (dirty tree) | Stash changes, create worktree, unstash |
| Worktree path already exists (filesystem) | Check if it's a valid worktree → resume. If stale → remove and recreate |
| Branch already exists | Check if it has commits → offer to resume. If empty → delete and recreate |
| Merge conflict | Report to user, don't auto-resolve |
| `git worktree remove` fails | Use `--force`, then `git worktree prune` |
