# Session Structure

Per-developer session directories for isolated working context. Each developer gets their own `.planning/sessions/{user}/` folder that's gitignored.

## Directory Hierarchy

```
.planning/
├── sessions/                           # [ephemeral] Per-developer directories
│   ├── {user}/                        # Sanitized user identity (see user-identity.md)
│   │   ├── current-plan.md            # Active plan copy with progress markers
│   │   └── context.md                 # Session-specific notes and scratch
│   ├── john-doe/                      # Example: "John Doe" → "john-doe"
│   │   ├── current-plan.md
│   │   └── context.md
│   └── alice-smith/                   # Example: "Alice Smith" → "alice-smith"
│       ├── current-plan.md
│       └── context.md
```

The `{user}` directory name is derived from `git config user.name` with sanitization applied. See `get-shit-done/references/user-identity.md` for the full detection and sanitization pattern.

## File Purposes

| File | Purpose | Created | Updated |
|------|---------|---------|---------|
| `current-plan.md` | Active plan copy with task progress markers (`[x]` for completed tasks) | On plan execution start | After each task completes |
| `context.md` | Session-specific notes, scratch space, developer observations | On demand | As needed during session |

### current-plan.md

This is a **working copy** of the current PLAN.md file with added progress markers. It enables:

- **Hydration**: Rebuild task state when session resumes
- **Progress tracking**: See which tasks are complete (`[x]`) vs pending (`[ ]`)
- **Interruption recovery**: If session ends mid-plan, progress is preserved

**Format**: Copy of PLAN.md with task status markers injected:

```markdown
<!-- Session progress for john-doe -->
<!-- Plan: .planning/phases/03-session-folder-structure/03-01-PLAN.md -->
<!-- Started: 2026-02-03T10:00:00Z -->

<tasks>
<task type="auto" status="complete">  <!-- Added status attribute -->
  <name>Task 1: Create schema</name>
  ...
</task>

<task type="auto" status="in-progress">
  <name>Task 2: Create API endpoints</name>
  ...
</task>

<task type="auto" status="pending">
  <name>Task 3: Add tests</name>
  ...
</task>
</tasks>
```

### context.md

Free-form session notes. Optional file for:

- Developer observations during execution
- Notes about decisions not captured in SUMMARY.md
- Scratch space for complex calculations
- Links to external resources consulted

This file is purely optional - sessions work fine without it.

## Creation Pattern

### Shell Snippet

```bash
# Get session directory path
# Uses get_session_user() from user-identity.md

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

# Create session directory
create_session_dir() {
  local session_dir=".planning/sessions/$(get_session_user)"
  mkdir -p "$session_dir"
  echo "$session_dir"
}

# Usage
SESSION_DIR=$(create_session_dir)
echo "Session directory: $SESSION_DIR"
```

### Workflow Integration

Workflows should call `create_session_dir()` at the start of execution:

```bash
# At start of execute-plan workflow
SESSION_DIR=$(create_session_dir)

# Copy current plan with progress markers
PLAN_PATH=".planning/phases/03-feature/03-01-PLAN.md"
cp "$PLAN_PATH" "$SESSION_DIR/current-plan.md"

# After each task completes, update progress markers
# (implementation details in Phase 4: Session Hydration)
```

## Lifecycle

### Creation

Session directories are created **on first GSD command** in a project:

1. User runs any GSD command (e.g., `/gsd:execute-plan`)
2. Workflow calls `create_session_dir()`
3. Directory structure created if it doesn't exist
4. Files created as needed during execution

### Persistence

Session directories persist between Claude sessions:

- Gitignored (never committed)
- Remain on filesystem across terminal sessions
- Enable resumption after Claude context resets

### Cleanup

Session directories are **never automatically cleaned up**:

- They're gitignored, so they don't pollute the repository
- Disk usage is minimal (markdown files only)
- Manual cleanup is optional: `rm -rf .planning/sessions/{user}`
- No reason to clean up - they don't cause problems

## Integration Points

### Phase 4: Session Hydration

Session directories enable hydration - reconstructing task state on session start:

1. Check `.planning/sessions/{user}/current-plan.md`
2. If exists and matches active plan in ROADMAP.md:
   - Parse task progress markers
   - Resume from last completed task
3. If missing or stale:
   - Create fresh from PLAN.md
   - Start from task 1

### Phase 6: Workflow Updates

Workflows will be updated to:

1. Create session directory at execution start
2. Copy PLAN.md to `current-plan.md` with progress markers
3. Update progress markers after each task completes
4. Read progress on session resume for hydration

### Existing Workflows

| Workflow | Session Directory Usage |
|----------|-------------------------|
| `execute-plan` | Creates session dir, maintains current-plan.md progress |
| `resume-work` | Reads current-plan.md for hydration |
| `progress` | Reports progress from current-plan.md |
| `plan-phase` | No session usage (planning doesn't need session isolation) |

## Edge Cases

| Scenario | Handling |
|----------|----------|
| Missing session directory | Create on first access (mkdir -p handles this) |
| Multiple terminals, same user | Same session directory - fine, files are updated atomically |
| User changes git config | New session directory created; old one orphaned (harmless) |
| Session directory deleted | Recreated on next GSD command; progress lost but SUMMARY.md has record |
| Very long username | No truncation; filesystem supports ~255 char paths |
| Non-ASCII characters | Stripped during sanitization (see user-identity.md) |

## Cross-References

- **State classification**: See `get-shit-done/references/state-architecture.md` for global vs ephemeral state split
- **Identity detection**: See `get-shit-done/references/user-identity.md` for user detection and sanitization
- **Gitignore patterns**: See `get-shit-done/templates/planning-gitignore.md` for sessions/ exclusion

**Consistency verified (Phase 03-01):**
- `current-plan.md` and `context.md` match Ephemeral State table in state-architecture.md
- `sessions/` gitignore pattern confirmed in planning-gitignore.md
- Shell snippets use identical detection/sanitization from user-identity.md
