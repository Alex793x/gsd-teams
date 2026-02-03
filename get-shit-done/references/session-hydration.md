# Session Hydration

Hydration is the process of reconstructing ephemeral task state from persistent files when a GSD session starts. This enables session continuity across Claude context resets and terminal restarts.

## 1. Hydration Triggers

### When Hydration Runs

Hydration occurs at **session start** — when Claude begins working on a GSD command after a context reset.

| Trigger | Description |
|---------|-------------|
| New Claude conversation | Fresh context, no memory of prior work |
| `/clear` command | Context explicitly reset by user |
| Context window overflow | Claude auto-summarizes, losing working state |
| Terminal restart | New session, prior in-memory state lost |

### Commands That Trigger Hydration

| Command | Hydration Behavior |
|---------|-------------------|
| `/gsd:execute-plan` | Hydrates to find resume point in current plan |
| `/gsd:resume-work` | Explicitly requests hydration and resume |
| `/gsd:progress` | Hydrates to report accurate task completion |
| `/gsd:verify-work` | Hydrates to know what was built |
| `/gsd:plan-phase` | No hydration (planning doesn't need execution state) |
| `/gsd:new-project` | No hydration (creates fresh project) |

## 2. Hydration Algorithm

### Step-by-Step Process

```
1. Detect user identity
   └─ Call get_session_user() to get sanitized username

2. Locate session file
   └─ Path: .planning/sessions/{user}/current-plan.md

3. Check session file exists
   ├─ If missing → Fresh session bootstrap (Section 6)
   └─ If exists → Continue to step 4

4. Validate session file
   ├─ Parse header comments for plan path
   ├─ Compare with current phase in ROADMAP.md
   ├─ If stale → Fresh session bootstrap (Section 6)
   └─ If valid → Continue to step 5

5. Parse task status attributes
   └─ Extract status="complete|in-progress|pending" from <task> elements

6. Determine resume point
   ├─ If all complete → Plan finished, offer next
   ├─ If none started → Start from task 1
   └─ Otherwise → Resume from first non-complete task

7. Reconstruct STATE.md if needed
   └─ Calculate position from ROADMAP.md and phase SUMMARY.md files
```

### Shell Implementation

```bash
# Full hydration sequence
hydrate_session() {
  local user plan_path session_file current_phase plan_phase

  # Step 1: Get user identity
  user=$(get_session_user)

  # Step 2: Locate session file
  session_file=".planning/sessions/${user}/current-plan.md"

  # Step 3: Check existence
  if [ ! -f "$session_file" ]; then
    echo "NO_SESSION"
    return 1
  fi

  # Step 4: Validate - extract plan path from header
  plan_path=$(grep -m1 '^<!-- Plan:' "$session_file" | sed 's/.*Plan: \(.*\) -->/\1/')

  # Get current phase from ROADMAP.md
  current_phase=$(grep -E '^\|.*In progress' .planning/ROADMAP.md | head -1 | grep -oE '[0-9]+' | head -1)

  # Get phase from session file path
  plan_phase=$(echo "$plan_path" | grep -oE '[0-9]+-' | head -1 | tr -d '-')

  # Check if stale (session references different phase than current)
  if [ "$plan_phase" != "$current_phase" ]; then
    echo "STALE"
    return 2
  fi

  # Step 5-6: Parse and return resume point
  parse_resume_point "$session_file"
}
```

## 3. Parsing current-plan.md

### XML Structure with Status Attributes

```xml
<!-- Session progress for john-doe -->
<!-- Plan: .planning/phases/03-session-folder-structure/03-01-PLAN.md -->
<!-- Started: 2026-02-03T10:00:00Z -->

<tasks>
<task type="auto" status="complete">
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

### Status Values

| Status | Meaning |
|--------|---------|
| `complete` | Task finished, committed |
| `in-progress` | Task started but not finished (interrupted) |
| `pending` | Task not yet started |

### Parsing Pattern (Grep/Sed)

```bash
# Extract all task statuses
parse_task_statuses() {
  local file="$1"
  grep -E '<task[^>]*status=' "$file" \
    | sed 's/.*status="\([^"]*\)".*/\1/'
}

# Example output:
# complete
# in-progress
# pending
```

### Extract Task Names with Status

```bash
# Get task names and their statuses
parse_tasks_with_status() {
  local file="$1"

  # Find task elements and extract both name and status
  grep -E '<task[^>]*status=' "$file" -A5 \
    | grep -E '(status=|<name>)' \
    | paste - - \
    | sed 's/.*status="\([^"]*\)".*<name>\(.*\)<\/name>.*/\2: \1/'
}

# Example output:
# Task 1: Create schema: complete
# Task 2: Create API endpoints: in-progress
# Task 3: Add tests: pending
```

### Count Tasks by Status

```bash
# Count tasks in each state
count_task_statuses() {
  local file="$1"

  echo "Complete: $(grep -c 'status="complete"' "$file")"
  echo "In progress: $(grep -c 'status="in-progress"' "$file")"
  echo "Pending: $(grep -c 'status="pending"' "$file")"
}
```

## 4. Resume Logic

### Finding the Resume Point

```bash
# Find first task to execute
parse_resume_point() {
  local file="$1"
  local task_num=0
  local resume_task=""

  # Read task statuses in order
  while IFS= read -r status; do
    task_num=$((task_num + 1))

    case "$status" in
      complete)
        # Task done, continue
        ;;
      in-progress|pending)
        # Found resume point
        resume_task="$task_num"
        break
        ;;
    esac
  done < <(parse_task_statuses "$file")

  if [ -z "$resume_task" ]; then
    echo "ALL_COMPLETE"
  else
    echo "$resume_task"
  fi
}
```

### Resume States

| State | Action |
|-------|--------|
| `ALL_COMPLETE` | Plan finished — check for next plan in phase |
| Task number (e.g., "3") | Resume from task 3 |
| `1` (all pending) | Fresh start — begin from task 1 |

### Handling In-Progress Tasks

If a task is marked `in-progress`, it was interrupted mid-execution:

```bash
# Check if task was interrupted
check_interrupted_task() {
  local file="$1"

  if grep -q 'status="in-progress"' "$file"; then
    # Extract the in-progress task name
    local task_name
    task_name=$(grep -B5 'status="in-progress"' "$file" \
      | grep '<name>' \
      | sed 's/.*<name>\(.*\)<\/name>.*/\1/')

    echo "INTERRUPTED: $task_name"
    return 0
  fi

  return 1
}
```

**Interrupted task handling:**
- Re-execute from the beginning of that task
- Task work may be partially complete
- Verification will determine if task needs full re-execution

## 5. Stale Session Detection

### What Makes a Session Stale

A session is stale when `current-plan.md` references a plan that is no longer active:

| Scenario | Detection | Action |
|----------|-----------|--------|
| Plan completed since last session | SUMMARY.md exists for the plan | Session is stale |
| Phase advanced | ROADMAP.md shows different "In progress" phase | Session is stale |
| Plan path changed | Header doesn't match any current PLAN.md | Session is stale |
| Task count mismatch | current-plan.md has different task count than PLAN.md | Session is stale |

### Shell Implementation

```bash
# Check if session is stale
is_session_stale() {
  local session_file="$1"
  local plan_path started_ts

  # Extract header info
  plan_path=$(grep -m1 '^<!-- Plan:' "$session_file" | sed 's/.*Plan: \(.*\) -->/\1/')
  started_ts=$(grep -m1 '^<!-- Started:' "$session_file" | sed 's/.*Started: \(.*\) -->/\1/')

  # Check 1: Does the plan file still exist?
  if [ ! -f "$plan_path" ]; then
    echo "STALE: Plan file no longer exists"
    return 0
  fi

  # Check 2: Does a SUMMARY already exist for this plan?
  local summary_path="${plan_path/PLAN/SUMMARY}"
  if [ -f "$summary_path" ]; then
    echo "STALE: Plan already completed (SUMMARY exists)"
    return 0
  fi

  # Check 3: Is this plan in the current phase?
  local plan_phase current_phase
  plan_phase=$(echo "$plan_path" | grep -oE '/[0-9]+-' | head -1 | tr -d '/-')
  current_phase=$(grep -E '^\|.*In progress' .planning/ROADMAP.md 2>/dev/null \
    | head -1 | grep -oE '[0-9]+' | head -1)

  if [ -n "$current_phase" ] && [ "$plan_phase" != "$current_phase" ]; then
    echo "STALE: Phase has advanced (was $plan_phase, now $current_phase)"
    return 0
  fi

  # Check 4: Task count matches?
  local session_tasks plan_tasks
  session_tasks=$(grep -c '<task[^>]*type=' "$session_file")
  plan_tasks=$(grep -c '<task[^>]*type=' "$plan_path")

  if [ "$session_tasks" != "$plan_tasks" ]; then
    echo "STALE: Task count mismatch ($session_tasks vs $plan_tasks)"
    return 0
  fi

  # Session is valid
  return 1
}
```

### Stale Session Handling

```bash
# Handle stale session
handle_stale_session() {
  local session_dir="$1"
  local session_file="${session_dir}/current-plan.md"

  # Archive stale session (optional — could just delete)
  local archive_name="current-plan.$(date +%Y%m%d%H%M%S).stale"
  mv "$session_file" "${session_dir}/${archive_name}"

  echo "Archived stale session to ${archive_name}"
  echo "Ready for fresh bootstrap"
}
```

## 6. Fresh Session Bootstrap

### When Fresh Bootstrap Occurs

- No `current-plan.md` exists for user
- Existing `current-plan.md` is stale
- User explicitly requests fresh start

### Bootstrap Process

```bash
# Bootstrap fresh session from PLAN.md
bootstrap_session() {
  local plan_path="$1"    # Path to PLAN.md
  local session_dir="$2"  # User's session directory

  local user session_file timestamp

  user=$(get_session_user)
  session_file="${session_dir}/current-plan.md"
  timestamp=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

  # Create session directory if needed
  mkdir -p "$session_dir"

  # Create header
  cat > "$session_file" << EOF
<!-- Session progress for ${user} -->
<!-- Plan: ${plan_path} -->
<!-- Started: ${timestamp} -->

EOF

  # Copy plan content with status="pending" injected
  # Transform: <task type="auto"> → <task type="auto" status="pending">
  sed 's/<task type="\([^"]*\)"/<task type="\1" status="pending"/g' \
    "$plan_path" >> "$session_file"

  echo "Bootstrapped session: $session_file"
}
```

### Initial Status Injection

Transform plan tasks to include status attributes:

```bash
# Original PLAN.md
<task type="auto">
  <name>Task 1: Create schema</name>
</task>

# After bootstrap in current-plan.md
<task type="auto" status="pending">
  <name>Task 1: Create schema</name>
</task>
```

### Full Bootstrap Shell Function

```bash
# Complete bootstrap with validation
create_fresh_session() {
  local plan_path="$1"
  local user session_dir

  user=$(get_session_user)
  session_dir=".planning/sessions/${user}"

  # Validate plan exists
  if [ ! -f "$plan_path" ]; then
    echo "ERROR: Plan not found: $plan_path"
    return 1
  fi

  # Create session directory
  mkdir -p "$session_dir"

  # Bootstrap the session file
  bootstrap_session "$plan_path" "$session_dir"

  # Return session file path
  echo "${session_dir}/current-plan.md"
}
```

## 7. Edge Cases

### Session Directory Missing

```bash
# Handled by mkdir -p in all creation functions
mkdir -p ".planning/sessions/$(get_session_user)"
# Creates full path, no error if exists
```

### current-plan.md Corrupted

**Detection:**
```bash
# Check for valid structure
validate_session_file() {
  local file="$1"

  # Must have header comment
  if ! grep -q '^<!-- Plan:' "$file"; then
    return 1
  fi

  # Must have at least one task
  if ! grep -q '<task' "$file"; then
    return 1
  fi

  # Must have status attributes (bootstrapped correctly)
  if ! grep -q 'status=' "$file"; then
    return 1
  fi

  return 0
}
```

**Handling:** Treat as missing — bootstrap fresh.

### Task Count Mismatch

If PLAN.md has been edited (tasks added/removed) after session started:

```bash
# Detect mismatch
check_task_count() {
  local session_file="$1"
  local plan_path

  plan_path=$(grep -m1 '^<!-- Plan:' "$session_file" | sed 's/.*Plan: \(.*\) -->/\1/')

  local session_count plan_count
  session_count=$(grep -c '<task[^>]*type=' "$session_file")
  plan_count=$(grep -c '<task[^>]*type=' "$plan_path")

  if [ "$session_count" != "$plan_count" ]; then
    echo "MISMATCH: session has $session_count tasks, plan has $plan_count"
    return 1
  fi

  return 0
}
```

**Handling:** Treat as stale — bootstrap fresh from updated PLAN.md.

### Plan File Path Changed

Plans can be renamed or moved. The header stores the original path:

```bash
# Check if plan path is still valid
check_plan_path() {
  local session_file="$1"
  local plan_path

  plan_path=$(grep -m1 '^<!-- Plan:' "$session_file" | sed 's/.*Plan: \(.*\) -->/\1/')

  if [ ! -f "$plan_path" ]; then
    echo "INVALID: Plan no longer at $plan_path"
    return 1
  fi

  return 0
}
```

**Handling:** Treat as stale — user must run execute-plan with new path.

### Multiple Users, Same Name

If two developers have identical sanitized names (e.g., both "john-doe"):

- They share session state (same directory)
- Last writer wins
- Not a bug — rare edge case
- Solution: one developer changes git user.name

### Session File Locked

On some systems, file locking may prevent writes:

```bash
# Atomic write pattern
update_session_file() {
  local session_file="$1"
  local temp_file="${session_file}.tmp"

  # Write to temp file
  # ... modifications ...

  # Atomic move (rename is atomic on POSIX)
  mv "$temp_file" "$session_file"
}
```

## 8. Updating Task Status

### After Task Completion

```bash
# Mark a task as complete
mark_task_complete() {
  local session_file="$1"
  local task_num="$2"

  # Use sed to update the Nth task element
  # This finds the Nth occurrence of status="pending" or status="in-progress"
  # and changes it to status="complete"

  local count=0
  local temp_file="${session_file}.tmp"

  while IFS= read -r line; do
    count_tasks=$((count_tasks + 1))
    if [ "$count_tasks" -eq "$task_num" ]; then
      # Update this task's status
      echo "$line" | sed 's/status="[^"]*"/status="complete"/'
    else
      echo "$line"
    fi
  done < "$session_file" > "$temp_file"

  mv "$temp_file" "$session_file"
}
```

### Mark Task In-Progress

```bash
# Mark a task as in-progress (starting execution)
mark_task_in_progress() {
  local session_file="$1"
  local task_num="$2"

  # Similar pattern to mark_task_complete
  # Changes status="pending" to status="in-progress"
  sed -i.bak "s/\(<task[^>]*\)status=\"pending\"/\1status=\"in-progress\"/" \
    "$session_file"
}
```

### Simplified Update Pattern

```bash
# Update task status by task number (1-indexed)
update_task_status() {
  local session_file="$1"
  local task_num="$2"
  local new_status="$3"  # complete, in-progress, pending

  # Use awk for precise Nth-occurrence replacement
  awk -v n="$task_num" -v status="$new_status" '
    /<task[^>]*type=/ {
      count++
      if (count == n) {
        gsub(/status="[^"]*"/, "status=\"" status "\"")
      }
    }
    { print }
  ' "$session_file" > "${session_file}.tmp"

  mv "${session_file}.tmp" "$session_file"
}
```

## Cross-References

- **Session structure:** See `get-shit-done/references/session-structure.md` for current-plan.md format
- **State architecture:** See `get-shit-done/references/state-architecture.md` for global vs ephemeral split
- **User identity:** See `get-shit-done/references/user-identity.md` for get_session_user() implementation
- **Workflow integration:** See Phase 6 plans for implementation in execute-plan workflow

---

**Consistency verified (Phase 04-01, 2026-02-03):**
- `current-plan.md` status attribute format (`status="complete|in-progress|pending"`) matches session-structure.md
- Header comment format (`<!-- Plan: ... -->`, `<!-- Started: ... -->`) matches session-structure.md
- `get_session_user()` function matches user-identity.md exactly
- Hydration concept matches state-architecture.md definition
- Shell snippets use identical patterns from prior phase documents
