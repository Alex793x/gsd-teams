# Backward Compatibility

GSD's team extension is designed to be fully backward compatible with existing solo projects. Solo projects continue working unchanged - team features are additive, not breaking.

## Overview

**Key principles:**

1. **Solo projects work unchanged** - No migration required, no breaking changes
2. **Team features are additive** - Adding team support is opt-in
3. **Defensive patterns everywhere** - All session operations handle missing state gracefully

## Compatibility Matrix

| Feature | Solo Project | Team Project |
|---------|--------------|--------------|
| STATE.md tracking | git-tracked (normal commits) | gitignored (per-developer) |
| sessions/ directory | created on first execute | created on first execute |
| .planning/.gitignore | optional (not required) | required for team isolation |
| User identity | works with fallbacks | requires git user.name |
| Session files | created but harmless | created and gitignored |
| Branch namespacing | optional | recommended |

## Defensive Pattern Catalog

All session operations use defensive patterns that handle missing state gracefully. This ensures workflows complete successfully regardless of project setup.

### Directory Creation

```bash
mkdir -p "$SESSION_DIR"
```

**Why:** Creates full path if missing, no error if exists. Safe to call repeatedly.

**Used in:** Session bootstrap, execute-plan workflow

### File Existence Guards

```bash
if [ -f "$file" ]; then
  # Only run if file exists
  content=$(cat "$file")
fi
```

**Why:** Prevents errors when session files don't exist yet.

**Used in:** Session resume detection, stale session checks

### Silent Failure Reads

```bash
git config user.name 2>/dev/null
```

**Why:** Suppresses error output when git config is unset. Returns empty string instead of error.

**Used in:** User identity detection

### Fallback Values

```bash
[ -z "$value" ] && value="default"
```

**Why:** Ensures variables always have usable values even when sources are empty.

**Used in:** User identity chain, session path construction

### User Identity Cascade

```bash
get_session_user() {
  local raw
  raw=$(git config user.name 2>/dev/null)
  [ -z "$raw" ] && raw=$(whoami 2>/dev/null)
  [ -z "$raw" ] && raw=$(hostname -s 2>/dev/null)
  [ -z "$raw" ] && raw="unknown"

  echo "$raw" \
    | tr '[:upper:]' '[:lower:]' \
    | tr ' ' '-' \
    | tr -cd 'a-z0-9-' \
    | sed 's/--*/-/g' \
    | sed 's/^-//;s/-$//'
}
```

**Why:** Guarantees a usable identifier regardless of git configuration. Never returns empty or fails.

**Fallback chain:**
1. `git config user.name` - preferred (human-readable, developer-chosen)
2. `whoami` - system username fallback
3. `hostname -s` - machine identifier fallback
4. `"unknown"` - last resort (never empty)

## What Gets Created

When execute-plan runs, these session artifacts are created:

| Artifact | Path | Purpose |
|----------|------|---------|
| Session directory | `.planning/sessions/{user}/` | Per-developer isolation |
| Session file | `.planning/sessions/{user}/current-plan.md` | Task progress tracking |

**In team projects:** These are gitignored by `.planning/.gitignore`

**In solo projects:** These exist locally but don't affect workflow:
- They're not committed (no .gitignore needed to exclude)
- They don't interfere with git operations
- They enable resume capability if desired
- They can be deleted without consequence

## Migration Paths

### Solo to Team

To enable team collaboration on an existing solo project:

```bash
# 1. Create .planning/.gitignore
cat > .planning/.gitignore << 'EOF'
# Ephemeral state (per-developer)
STATE.md
sessions/
agent-history.json
current-agent-id.txt
EOF

# 2. Remove STATE.md from git tracking (keep local file)
git rm --cached .planning/STATE.md

# 3. Commit the gitignore
git add .planning/.gitignore
git commit -m "chore: add team support with gitignore for ephemeral state"
```

After migration:
- Each developer has their own STATE.md (local, not shared)
- Session files are automatically isolated
- PLAN.md and SUMMARY.md continue to be shared

### Team to Solo

No migration needed. Simply:
- Delete `.planning/.gitignore` (optional)
- Start tracking STATE.md again if desired: `git add .planning/STATE.md`

The workflows work identically either way.

## Why This Works

### Session Bootstrap is Idempotent

The session bootstrap step in execute-plan:

```bash
SESSION_DIR=".planning/sessions/${SESSION_USER}"
mkdir -p "$SESSION_DIR"
```

This runs on **every** execute-plan invocation:
- If directory exists: no-op (mkdir -p succeeds)
- If directory missing: creates it
- No state corruption possible

### Defensive Checks Prevent Errors

Every workflow that reads session state guards against missing files:

```bash
# Check if session file exists before reading
if [ -f "$SESSION_FILE" ]; then
  # Parse session state
else
  # Bootstrap fresh session
fi
```

This means:
- First execution: creates session files
- Subsequent executions: uses existing files
- Missing files: handled gracefully

### User Identity Never Fails

The cascading fallback ensures `get_session_user()` always returns a usable string:

```bash
# This ALWAYS returns something
SESSION_USER=$(get_session_user)
SESSION_DIR=".planning/sessions/${SESSION_USER}"
```

Even if:
- Git is not installed: falls back to whoami
- Git user.name is unset: falls back to whoami
- whoami fails: falls back to hostname
- Everything fails: returns "unknown"

## Design Rationale

| Decision | Rationale |
|----------|-----------|
| Sessions created regardless of mode | Enables resume capability without requiring config |
| mkdir -p over explicit checks | Idempotent, fewer code paths, no race conditions |
| Fallback chains over hard requirements | Workflows complete even in edge cases |
| gitignore is opt-in | Solo projects don't need to change anything |
| Session files are harmless | Creating them costs nothing, provides benefit |

## Cross-References

- **State architecture:** See `get-shit-done/references/state-architecture.md` for global vs ephemeral split
- **User identity:** See `get-shit-done/references/user-identity.md` for detection patterns
- **Session structure:** See `get-shit-done/references/session-structure.md` for file formats
- **Session hydration:** See `get-shit-done/references/session-hydration.md` for resume logic
