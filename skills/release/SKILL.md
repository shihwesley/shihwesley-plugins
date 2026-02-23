---
name: release
description: "Release a shihwesley plugin — validates, tags, pushes, and auto-updates marketplace.json. Use when pushing a new version of any plugin to main."
---

# Release: $ARGUMENTS

Automate the full release flow for a shihwesley plugin.

## Usage

```
/release [patch|minor|major] [--dry-run]
```

- **patch** (default): bug fixes, small tweaks → `v2.1.0` → `v2.1.1`
- **minor**: new features, backward compatible → `v2.1.0` → `v2.2.0`
- **major**: breaking changes → `v2.1.0` → `v3.0.0`
- **--dry-run**: show what would happen without pushing anything

If no bump type given, auto-detect from commit messages since last tag:
- Commits containing `fix`, `patch`, `bug` → patch
- Commits containing `add`, `feat`, `new` → minor
- Commits containing `break`, `major`, `rewrite` → major

## Step 1: Identify the plugin

Detect which plugin repo we're in by matching the current git remote against the marketplace registry.

```bash
REMOTE=$(git remote get-url origin 2>/dev/null)
```

Match `$REMOTE` against the `repository` fields in:
`/Users/quartershots/Source/shihwesley-plugins/.claude-plugin/marketplace.json`

If no match → error: "Not in a recognized shihwesley plugin repo."

Extract: `PLUGIN_NAME`, `CURRENT_VERSION`, `REPO_URL`

## Step 2: Validate release readiness

Run these checks. **All must pass before proceeding.**

```bash
# 1. On main branch
BRANCH=$(git branch --show-current)
[ "$BRANCH" = "main" ] || error "Not on main. Switch to main first."

# 2. Clean working tree
[ -z "$(git status --porcelain)" ] || error "Uncommitted changes. Commit or stash first."

# 3. Up to date with remote
git fetch origin main --quiet
LOCAL=$(git rev-parse HEAD)
REMOTE_HEAD=$(git rev-parse origin/main)
[ "$LOCAL" = "$REMOTE_HEAD" ] || warn "Local differs from origin/main."

# 4. Commits exist since last tag
LAST_TAG=$(git tag --sort=-v:refname | head -1)
COMMITS=$(git log "$LAST_TAG"..HEAD --oneline)
[ -n "$COMMITS" ] || error "No commits since $LAST_TAG. Nothing to release."
```

Show the user:
- Current version: `$LAST_TAG`
- Commits since last tag (short log)
- Proposed next version

## Step 3: Determine next version

Parse `$ARGUMENTS` for bump type. If not specified, auto-detect:

```bash
# Count commit keywords
FIXES=$(git log "$LAST_TAG"..HEAD --oneline | grep -iE 'fix|patch|bug|correct' | wc -l)
FEATURES=$(git log "$LAST_TAG"..HEAD --oneline | grep -iE 'add|feat|new|implement' | wc -l)
BREAKING=$(git log "$LAST_TAG"..HEAD --oneline | grep -iE 'break|major|rewrite|remove' | wc -l)
```

Priority: breaking > features > fixes. Default to patch if ambiguous.

Compute `NEXT_VERSION` from `LAST_TAG` by bumping the appropriate segment.

## Step 4: Confirm with user

Use AskUserQuestion:
- Header: "Release"
- Question: "Release $PLUGIN_NAME $NEXT_VERSION? ($COMMIT_COUNT commits since $LAST_TAG)"
- Options:
  1. "Release $NEXT_VERSION" — proceed
  2. "Change version" — let user specify a different version
  3. "Abort" — cancel

## Step 5: Tag and push

```bash
git tag "$NEXT_VERSION"
git push origin main
git push origin "$NEXT_VERSION"
```

Verify:
```bash
git ls-remote --tags origin | grep "$NEXT_VERSION"
```

## Step 6: Update marketplace.json

Read `/Users/quartershots/Source/shihwesley-plugins/.claude-plugin/marketplace.json`.

Find the plugin entry by matching `name` == `$PLUGIN_NAME`.

Update:
- `"version"` → new version (without `v` prefix)
- `"description"` → keep existing, unless `$ARGUMENTS` contains `--desc "new description"`

Write the updated file.

## Step 7: Commit and push marketplace

```bash
cd /Users/quartershots/Source/shihwesley-plugins
git add .claude-plugin/marketplace.json
git commit -m "Bump $PLUGIN_NAME to $NEXT_VERSION_NO_PREFIX"
git push origin main
```

## Step 8: Report

```
Released: $PLUGIN_NAME $NEXT_VERSION
- Tag: $NEXT_VERSION pushed to $REPO_URL
- Marketplace: shihwesley-plugins updated and pushed
- Commits included: $COMMIT_COUNT
- Reinstall: claude plugins update shihwesley-plugins/$PLUGIN_NAME
```

## Dry Run Mode

If `--dry-run` is in `$ARGUMENTS`:
- Run Steps 1-4 (validate, detect version, show plan)
- Skip Steps 5-7 (no tag, no push, no marketplace update)
- Report what would have happened

## Edge Cases

- **marketplace.json not found:** warn and skip marketplace update, still tag and push
- **Already tagged at HEAD:** error "HEAD is already tagged $LAST_TAG. Make commits first."
- **Plugin not in marketplace:** warn "Plugin not found in marketplace.json. Tag pushed but marketplace not updated."
- **shihwesley-plugins has uncommitted changes:** stash before updating, pop after push

## Rules

- Never force-push
- Never skip pre-commit hooks
- Always confirm version with user before tagging
- Always return to the original working directory after marketplace update
