---
name: release
description: "Release a shihwesley plugin — validates, tests, generates changelog, tags, creates GitHub Release, updates version files and marketplace.json. Industry-standard semver flow."
---

# Release: $ARGUMENTS

Full release pipeline for shihwesley plugins. Follows semver, conventional commits, and GitHub Release best practices.

## Usage

```
/release [patch|minor|major] [--dry-run]
```

- **patch** (default): bug fixes → `v2.1.0` → `v2.1.1`
- **minor**: new features, backward compatible → `v2.1.0` → `v2.2.0`
- **major**: breaking changes → `v2.1.0` → `v3.0.0`
- **--dry-run**: run all checks, show what would happen, push nothing

## Conventional Commits

Version auto-detection uses conventional commit prefixes, not keyword guessing:

| Prefix | Bump | Example |
|--------|------|---------|
| `fix:` `fix(scope):` | patch | `fix: handle empty query in BM25 search` |
| `feat:` `feat(scope):` | minor | `feat: add Phase 0 knowledge store search` |
| `BREAKING CHANGE:` in body, or `!` after type | major | `feat!: rewrite search API` |
| `docs:` `chore:` `refactor:` `test:` `ci:` | no bump | maintenance, skip for version calc |

If commits don't follow conventional format, fall back to asking the user.

---

## Step 1: Identify the plugin

```bash
PLUGIN_DIR=$(pwd)
REMOTE=$(git remote get-url origin 2>/dev/null)
```

Match `$REMOTE` against `repository` fields in:
`/Users/quartershots/Source/shihwesley-plugins/.claude-plugin/marketplace.json`

Extract: `PLUGIN_NAME`, `CURRENT_VERSION`, `REPO_URL`.

If no match → error: "Not in a recognized shihwesley plugin repo. Run from a plugin directory."

---

## Step 2: Validate release readiness

All checks must pass:

```bash
# 1. On main branch
BRANCH=$(git branch --show-current)
[ "$BRANCH" = "main" ] || error "Not on main branch."

# 2. Clean working tree (no uncommitted changes)
[ -z "$(git status --porcelain)" ] || error "Uncommitted changes. Commit or stash first."

# 3. Synced with remote
git fetch origin main --quiet
LOCAL=$(git rev-parse HEAD)
REMOTE_HEAD=$(git rev-parse origin/main)
[ "$LOCAL" = "$REMOTE_HEAD" ] || error "Local differs from origin/main. Pull or push first."

# 4. Commits exist since last tag
LAST_TAG=$(git tag --sort=-v:refname | head -1)
COMMIT_COUNT=$(git rev-list "$LAST_TAG"..HEAD --count)
[ "$COMMIT_COUNT" -gt 0 ] || error "No commits since $LAST_TAG. Nothing to release."

# 5. HEAD not already tagged
TAGGED=$(git tag --points-at HEAD)
[ -z "$TAGGED" ] || error "HEAD is already tagged $TAGGED."
```

---

## Step 3: Run tests

Detect project type and run the appropriate test suite:

```bash
# Python (pyproject.toml or setup.py)
if [ -f pyproject.toml ] || [ -f setup.py ]; then
    pytest tests/ -x --tb=short
fi

# Node.js (package.json with test script)
if [ -f package.json ] && grep -q '"test"' package.json; then
    npm test
fi

# Swift (Package.swift or .xcodeproj)
if [ -f Package.swift ]; then
    swift test
elif ls *.xcodeproj 1>/dev/null 2>&1; then
    set -o pipefail && xcodebuild test -scheme "$(xcodebuild -list -json | python3 -c 'import sys,json; print(json.load(sys.stdin)["project"]["schemes"][0])')" 2>&1 | xcbeautify
fi
```

If tests fail → abort release. Show failure output.

If no test runner detected → warn "No test suite found. Proceeding without tests." and ask user to confirm.

---

## Step 4: Determine next version

### Parse conventional commits

```bash
git log "$LAST_TAG"..HEAD --pretty=format:"%s" --no-merges
```

Scan each commit subject:
- `^feat!:` or body contains `BREAKING CHANGE:` → **major**
- `^feat(\(.*\))?:` → **minor**
- `^fix(\(.*\))?:` → **patch**
- `^docs:|^chore:|^refactor:|^test:|^ci:|^style:|^perf:` → no bump (but include in changelog)

Highest bump wins. If no conventional commits found, ask the user to pick patch/minor/major.

Override: if `$ARGUMENTS` explicitly says `patch`, `minor`, or `major`, use that regardless of commit analysis.

### Compute version

Parse `LAST_TAG` (strip `v` prefix), split on `.`, bump the right segment, reset lower segments to 0.

Example: `v2.1.3` + minor → `v2.2.0`

---

## Step 5: Generate CHANGELOG entry

Build the entry from conventional commits, grouped by type:

```markdown
## [X.Y.Z] - YYYY-MM-DD

### Added
- Phase 0 knowledge store search (#commit-hash)

### Fixed
- BM25 multi-word queries returning 0 results (#commit-hash)

### Changed
- Research pipeline now runs 6 phases instead of 5

### Breaking Changes
- (only if major bump)
```

Mapping:
- `feat:` → **Added**
- `fix:` → **Fixed**
- `refactor:` `perf:` → **Changed**
- `BREAKING CHANGE:` → **Breaking Changes**
- `docs:` `chore:` `test:` `ci:` → omit from changelog (internal)

Non-conventional commits go under **Other** with their full subject line.

### Write CHANGELOG.md

If `CHANGELOG.md` exists, prepend the new entry after the `# Changelog` header.
If it doesn't exist, create it with a header and this first entry.

Stage the file: `git add CHANGELOG.md`

---

## Step 6: Update version files

Detect and update version strings in project files:

```bash
# Python: pyproject.toml
if [ -f pyproject.toml ]; then
    # Update: version = "X.Y.Z" under [project] or [tool.poetry]
fi

# Node.js: package.json
if [ -f package.json ]; then
    # Update: "version": "X.Y.Z"
fi

# Claude plugin: .claude-plugin/plugin.json or similar
if [ -f .claude-plugin/plugin.json ]; then
    # Update version field
fi
```

Stage changed files: `git add pyproject.toml package.json` (whichever exist)

---

## Step 7: Confirm with user

Use AskUserQuestion:
- Header: "Release"
- Question: "Release $PLUGIN_NAME $NEXT_VERSION?"
- Show: commit count, changelog preview (first 10 lines), files that will change
- Options:
  1. "Release $NEXT_VERSION" — proceed
  2. "Change version" — let user type a version
  3. "Abort" — cancel, unstage everything

---

## Step 8: Commit, tag, and push

```bash
# Commit version bump + changelog (if any files were staged)
if ! git diff --cached --quiet; then
    git commit -m "chore: release $NEXT_VERSION"
fi

# Tag
git tag -a "$NEXT_VERSION" -m "Release $NEXT_VERSION"

# Push commit + tag
git push origin main
git push origin "$NEXT_VERSION"
```

---

## Step 9: Create GitHub Release

```bash
# Generate release notes from changelog entry
gh release create "$NEXT_VERSION" \
  --title "$PLUGIN_NAME $NEXT_VERSION_NO_V" \
  --notes-file <(extract changelog entry for this version) \
  --latest
```

If `gh` is not installed or auth fails → warn and skip. The tag is already pushed; a GitHub Release can be created manually.

---

## Step 10: Update marketplace.json

```bash
MARKETPLACE="/Users/quartershots/Source/shihwesley-plugins/.claude-plugin/marketplace.json"
ORIGINAL_DIR=$(pwd)
```

Read marketplace.json. Find plugin entry by `name == $PLUGIN_NAME`. Update `"version"` to `$NEXT_VERSION` (without `v` prefix).

Write the file.

```bash
cd /Users/quartershots/Source/shihwesley-plugins
git add .claude-plugin/marketplace.json
git commit -m "chore: bump $PLUGIN_NAME to $NEXT_VERSION_NO_V"
git push origin main
cd "$ORIGINAL_DIR"
```

If shihwesley-plugins has uncommitted changes → `git stash` before, `git stash pop` after.

---

## Step 11: Report

```
Released: $PLUGIN_NAME $NEXT_VERSION

Tag:        $NEXT_VERSION → $REPO_URL
GitHub:     https://github.com/shihwesley/$PLUGIN_NAME/releases/tag/$NEXT_VERSION
Changelog:  CHANGELOG.md updated
Marketplace: shihwesley-plugins bumped to $NEXT_VERSION_NO_V
Commits:    $COMMIT_COUNT included

Reinstall:  claude plugins update shihwesley-plugins/$PLUGIN_NAME
```

---

## Dry Run Mode

If `--dry-run` in `$ARGUMENTS`:
- Run Steps 1-5 (validate, test, detect version, generate changelog preview)
- Show what would be written to CHANGELOG.md
- Show what version files would change
- Show the `gh release create` command that would run
- Skip Steps 6-10 (no writes, no tags, no pushes)
- Report: "Dry run complete. No changes made."

---

## Edge Cases

- **No conventional commits:** ask user for bump type, generate changelog from raw subjects
- **marketplace.json not found:** warn, skip marketplace update, still tag and release
- **Plugin not in marketplace:** warn, tag pushed but marketplace not updated
- **Tests fail:** abort entirely, show failure output
- **gh CLI not installed:** skip GitHub Release, warn user
- **Multiple tags at different commits:** use most recent semver tag only
- **Pre-release versions (alpha/beta):** not supported yet — use manual tagging

## Rules

- Never force-push
- Never skip hooks (no `--no-verify`)
- Always run tests before tagging
- Always confirm with user before tagging
- Always return to original directory after marketplace update
- Changelog follows [Keep a Changelog](https://keepachangelog.com/) format
- Versions follow [Semantic Versioning](https://semver.org/) strictly
