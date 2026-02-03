# User Identity Detection

Automatic developer identification for per-user session isolation. Uses git configuration for zero-setup identity detection.

## Primary Detection

```bash
git config user.name
```

Returns the configured git user name. This is the preferred source because:
- Already configured on most developer machines
- Familiar to developers (they chose this name)
- Consistent across sessions on the same machine
- No additional configuration required

## Fallback Chain

If `git config user.name` returns empty or fails, fall back in order:

| Priority | Source | Command | Notes |
|----------|--------|---------|-------|
| 1 | Git config | `git config user.name` | Preferred - developer-chosen name |
| 2 | System user | `whoami` (Unix) / `echo %USERNAME%` (Windows) | System login name |
| 3 | Hostname | `hostname -s` | Last resort - machine identifier |

**Never fail.** The detection chain must always return a usable identifier.

## Sanitization Rules

Git user names may contain characters that are problematic for filesystem paths. Apply these transformations:

| Rule | Example |
|------|---------|
| Lowercase | "John Doe" → "john doe" |
| Spaces → hyphens | "john doe" → "john-doe" |
| Remove non-alphanumeric (except hyphens) | "john@work!" → "johnwork" |
| Collapse multiple hyphens | "john--doe" → "john-doe" |
| Trim leading/trailing hyphens | "-john-" → "john" |

**Examples:**
- "John Doe" → "john-doe"
- "Admin@Work" → "adminwork"
- "José García" → "jos-garca" (accents stripped)
- "user_123" → "user123"
- "ALICE" → "alice"

## Shell Snippets

### Detection with Fallbacks (Bash)

```bash
# Get raw identity (with fallbacks)
get_user_identity() {
  local name
  name=$(git config user.name 2>/dev/null)

  if [ -z "$name" ]; then
    name=$(whoami 2>/dev/null)
  fi

  if [ -z "$name" ]; then
    name=$(hostname -s 2>/dev/null)
  fi

  if [ -z "$name" ]; then
    name="unknown"
  fi

  echo "$name"
}
```

### Sanitization Function (Bash)

```bash
# Sanitize name for filesystem use
sanitize_name() {
  local name="$1"

  echo "$name" \
    | tr '[:upper:]' '[:lower:]' \
    | tr ' ' '-' \
    | tr -cd 'a-z0-9-' \
    | sed 's/--*/-/g' \
    | sed 's/^-//;s/-$//'
}
```

### Combined One-Liner

```bash
# Get sanitized user identity in one command
USER_ID=$(git config user.name 2>/dev/null || whoami || hostname -s || echo unknown | tr '[:upper:]' '[:lower:]' | tr ' ' '-' | tr -cd 'a-z0-9-' | sed 's/--*/-/g;s/^-//;s/-$//')
```

### Full Detection and Sanitization

```bash
# Complete pattern for use in workflows
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

# Usage: Creates session directory with sanitized user name
SESSION_DIR=".planning/sessions/$(get_session_user)"
mkdir -p "$SESSION_DIR"
```

## Design Rationale

### Why git user.name?

| Alternative | Rejected Because |
|-------------|------------------|
| Environment variable (e.g., `$GSD_USER`) | Requires setup, not zero-config |
| Config file in `.planning/` | Per-project setup required |
| SSH key fingerprint | Complex, not human-readable |
| `whoami` only | Not personalized, same across machines |
| Email address | Contains `@` which needs escaping |

`git config user.name` is optimal because:
1. **Zero setup** - Git users already have this configured
2. **Human readable** - Developer chose this name
3. **Consistent** - Same name across all git repos on their machine
4. **Team familiarity** - Appears in git commits, everyone knows it

### Session Path Example

With user "John Doe":
```
.planning/sessions/john-doe/
.planning/sessions/john-doe/current-plan.md
.planning/sessions/john-doe/context.md
```

This creates isolated working space that:
- Won't conflict with other developers
- Is clearly labeled with who owns it
- Can be cleaned up without affecting others
- Gets gitignored (ephemeral state per state-architecture.md)

## Edge Cases

| Scenario | Handling |
|----------|----------|
| Git not installed | Falls back to whoami |
| Empty git user.name | Falls back to whoami |
| All fallbacks empty | Returns "unknown" |
| Non-ASCII characters | Stripped during sanitization |
| Very long names | No truncation (filesystem limit ~255 chars) |
| Same name, different machines | Same session folder (expected behavior) |
