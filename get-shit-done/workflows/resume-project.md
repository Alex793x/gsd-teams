<trigger>
Use this workflow when:
- Starting a new session on an existing project
- User says "continue", "what's next", "where were we", "resume"
- Any planning operation when .planning/ already exists
- User returns after time away from project
</trigger>

<purpose>
Instantly restore full project context so "Where were we?" has an immediate, complete answer.
</purpose>

<required_reading>
@~/.claude/get-shit-done/references/continuation-format.md
</required_reading>

<process>

<step name="detect_existing_project">
Check if this is an existing project:

```bash
ls .planning/STATE.md 2>/dev/null && echo "Project exists"
ls .planning/ROADMAP.md 2>/dev/null && echo "Roadmap exists"
ls .planning/PROJECT.md 2>/dev/null && echo "Project file exists"
```

**If STATE.md exists:** Proceed to load_state
**If only ROADMAP.md/PROJECT.md exist:** Offer to reconstruct STATE.md
**If .planning/ doesn't exist:** This is a new project - route to /gsd:new-project
</step>

<step name="load_state">

Read and parse STATE.md, then PROJECT.md:

```bash
cat .planning/STATE.md
cat .planning/PROJECT.md
```

**From STATE.md extract:**

- **Project Reference**: Core value and current focus
- **Current Position**: Phase X of Y, Plan A of B, Status
- **Progress**: Visual progress bar
- **Recent Decisions**: Key decisions affecting current work
- **Pending Todos**: Ideas captured during sessions
- **Blockers/Concerns**: Issues carried forward
- **Session Continuity**: Where we left off, any resume files

**From PROJECT.md extract:**

- **What This Is**: Current accurate description
- **Requirements**: Validated, Active, Out of Scope
- **Key Decisions**: Full decision log with outcomes
- **Constraints**: Hard limits on implementation

</step>

<step name="check_incomplete_work">
Look for incomplete work that needs attention:

```bash
# Session integration: See get-shit-done/references/session-hydration.md
# get_session_user() - from user-identity.md
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

# Check for continue-here files (mid-plan resumption)
ls .planning/phases/*/.continue-here*.md 2>/dev/null

# Check for plans without summaries (incomplete execution)
for plan in .planning/phases/*/*-PLAN.md; do
  summary="${plan/PLAN/SUMMARY}"
  [ ! -f "$summary" ] && echo "Incomplete: $plan"
done 2>/dev/null

# Check for interrupted agents
if [ -f .planning/current-agent-id.txt ] && [ -s .planning/current-agent-id.txt ]; then
  AGENT_ID=$(cat .planning/current-agent-id.txt | tr -d '\n')
  echo "Interrupted agent: $AGENT_ID"
fi

# Check for session file with in-progress work
user=$(get_session_user)
session_file=".planning/sessions/${user}/current-plan.md"
if [ -f "$session_file" ]; then
  # Parse task statuses
  in_progress=$(grep -c 'status="in-progress"' "$session_file" 2>/dev/null || echo 0)
  pending=$(grep -c 'status="pending"' "$session_file" 2>/dev/null || echo 0)
  complete=$(grep -c 'status="complete"' "$session_file" 2>/dev/null || echo 0)
  echo "Session: $complete complete, $in_progress in-progress, $pending pending"
fi
```

**If .continue-here file exists:**

- This is a mid-plan resumption point
- Read the file for specific resumption context
- Flag: "Found mid-plan checkpoint"

**If PLAN without SUMMARY exists:**

- Execution was started but not completed
- Flag: "Found incomplete plan execution"

**If interrupted agent found:**

- Subagent was spawned but session ended before completion
- Read agent-history.json for task details
- Flag: "Found interrupted agent"

**If session file exists with in-progress tasks:**

- Task was interrupted mid-execution
- Flag: "Found interrupted task in session"

**If session file exists with pending tasks:**

- Session started but not completed
- Flag: "Found incomplete session"
  </step>

<step name="present_status">
Present complete project status to user:

```
╔══════════════════════════════════════════════════════════════╗
║  PROJECT STATUS                                               ║
╠══════════════════════════════════════════════════════════════╣
║  Building: [one-liner from PROJECT.md "What This Is"]         ║
║                                                               ║
║  Phase: [X] of [Y] - [Phase name]                            ║
║  Plan:  [A] of [B] - [Status]                                ║
║  Progress: [██████░░░░] XX%                                  ║
║                                                               ║
[If session file exists:]
║  Session: [X] complete, [Y] pending, [Z] in-progress         ║
║  Plan: [plan path from session header]                        ║
║                                                               ║
║  Last activity: [date] - [what happened]                     ║
╚══════════════════════════════════════════════════════════════╝

[If incomplete work found:]
⚠️  Incomplete work detected:
    - [.continue-here file or incomplete plan]

[If interrupted agent found:]
⚠️  Interrupted agent detected:
    Agent ID: [id]
    Task: [task description from agent-history.json]
    Interrupted: [timestamp]

    Resume with: Task tool (resume parameter with agent ID)

[If session is stale:]
⚠️  Stale session detected:
    - Session references: [old plan path]
    - Current phase: [current phase from ROADMAP.md]
    Will archive and start fresh.

[If in-progress task found in session:]
⚠️  Interrupted task detected:
    Task: [task name from session file]
    Will resume from this task.

[If pending todos exist:]
📋 [N] pending todos — /gsd:check-todos to review

[If blockers exist:]
⚠️  Carried concerns:
    - [blocker 1]
    - [blocker 2]

[If alignment is not ✓:]
⚠️  Brief alignment: [status] - [assessment]
```

**Session display parsing:**

```bash
# Parse session info for display (use same get_session_user from check_incomplete_work)
user=$(get_session_user)
session_file=".planning/sessions/${user}/current-plan.md"

if [ -f "$session_file" ]; then
  # Task counts
  in_progress=$(grep -c 'status="in-progress"' "$session_file" 2>/dev/null || echo 0)
  pending=$(grep -c 'status="pending"' "$session_file" 2>/dev/null || echo 0)
  complete=$(grep -c 'status="complete"' "$session_file" 2>/dev/null || echo 0)

  # Plan path from header
  plan_path=$(grep -m1 '^<!-- Plan:' "$session_file" | sed 's/.*Plan: \(.*\) -->/\1/')

  # Check for in-progress (interrupted) task
  if [ "$in_progress" -gt 0 ]; then
    interrupted_task=$(grep -B5 'status="in-progress"' "$session_file" \
      | grep '<name>' | sed 's/.*<name>\(.*\)<\/name>.*/\1/' | head -1)
    echo "Interrupted task: $interrupted_task"
  fi
fi
```

</step>

<step name="determine_next_action">
Based on project state, determine the most logical next action:

**If interrupted agent exists:**
→ Primary: Resume interrupted agent (Task tool with resume parameter)
→ Option: Start fresh (abandon agent work)

**If .continue-here file exists:**
→ Primary: Resume from checkpoint
→ Option: Start fresh on current plan

**If session file exists with incomplete work:**

First check if session is stale using is_session_stale() 4-check algorithm:

```bash
# Check if session is stale
is_session_stale() {
  local session_file="$1"
  local plan_path started_ts

  # Extract header info
  plan_path=$(grep -m1 '^<!-- Plan:' "$session_file" | sed 's/.*Plan: \(.*\) -->/\1/')

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

# Find resume point from session file
parse_resume_point() {
  local file="$1"
  local task_num=0
  local resume_task=""

  # Read task statuses in order
  while IFS= read -r status; do
    task_num=$((task_num + 1))
    case "$status" in
      complete) ;;  # Task done, continue
      in-progress|pending)
        resume_task="$task_num"
        break
        ;;
    esac
  done < <(grep -E '<task[^>]*status=' "$file" | sed 's/.*status="\([^"]*\)".*/\1/')

  if [ -z "$resume_task" ]; then
    echo "ALL_COMPLETE"
  else
    echo "$resume_task"
  fi
}
```

**Session routing logic:**
- If session is stale → Archive stale session, start fresh from current PLAN.md
- If session valid with in-progress/pending tasks:
  → Primary: Resume session (continue from last task)
  → Parse resume point using parse_resume_point()
  → Display: "Resume from task {N} of {total}?"
  → Option: Start fresh (abandon session progress)

**If incomplete plan (PLAN without SUMMARY):**
→ Primary: Complete the incomplete plan
→ Option: Abandon and move on

**If phase in progress, all plans complete:**
→ Primary: Transition to next phase
→ Option: Review completed work

**If phase ready to plan:**
→ Check if CONTEXT.md exists for this phase:

- If CONTEXT.md missing:
  → Primary: Discuss phase vision (how user imagines it working)
  → Secondary: Plan directly (skip context gathering)
- If CONTEXT.md exists:
  → Primary: Plan the phase
  → Option: Review roadmap

**If phase ready to execute:**
→ Primary: Execute next plan
→ Option: Review the plan first
</step>

<step name="offer_options">
Present contextual options based on project state:

```
What would you like to do?

[Primary action based on state - e.g.:]
1. Resume interrupted agent [if interrupted agent found]
   OR
1. Execute phase (/gsd:execute-phase {phase})
   OR
1. Discuss Phase 3 context (/gsd:discuss-phase 3) [if CONTEXT.md missing]
   OR
1. Plan Phase 3 (/gsd:plan-phase 3) [if CONTEXT.md exists or discuss option declined]

[Secondary options:]
2. Review current phase status
3. Check pending todos ([N] pending)
4. Review brief alignment
5. Something else
```

**Note:** When offering phase planning, check for CONTEXT.md existence first:

```bash
ls .planning/phases/XX-name/*-CONTEXT.md 2>/dev/null
```

If missing, suggest discuss-phase before plan. If exists, offer plan directly.

Wait for user selection.
</step>

<step name="route_to_workflow">
Based on user selection, route to appropriate workflow:

- **Execute plan** → Show command for user to run after clearing:
  ```
  ---

  ## ▶ Next Up

  **{phase}-{plan}: [Plan Name]** — [objective from PLAN.md]

  `/gsd:execute-phase {phase}`

  <sub>`/clear` first → fresh context window</sub>

  ---
  ```
- **Plan phase** → Show command for user to run after clearing:
  ```
  ---

  ## ▶ Next Up

  **Phase [N]: [Name]** — [Goal from ROADMAP.md]

  `/gsd:plan-phase [phase-number]`

  <sub>`/clear` first → fresh context window</sub>

  ---

  **Also available:**
  - `/gsd:discuss-phase [N]` — gather context first
  - `/gsd:research-phase [N]` — investigate unknowns

  ---
  ```
- **Transition** → ./transition.md
- **Check todos** → Read .planning/todos/pending/, present summary
- **Review alignment** → Read PROJECT.md, compare to current state
- **Something else** → Ask what they need
</step>

<step name="update_session">
Before proceeding to routed workflow, update session continuity:

Update STATE.md:

```markdown
## Session Continuity

Last session: [now]
Stopped at: Session resumed, proceeding to [action]
Resume file: [updated if applicable]
```

This ensures if session ends unexpectedly, next resume knows the state.
</step>

</process>

<reconstruction>
If STATE.md is missing but other artifacts exist:

"STATE.md missing. Reconstructing from artifacts..."

1. Read PROJECT.md → Extract "What This Is" and Core Value
2. Read ROADMAP.md → Determine phases, find current position
3. Scan \*-SUMMARY.md files → Extract decisions, concerns
4. Count pending todos in .planning/todos/pending/
5. Check for .continue-here files → Session continuity

Reconstruct and write STATE.md, then proceed normally.

This handles cases where:

- Project predates STATE.md introduction
- File was accidentally deleted
- Cloning repo without full .planning/ state
  </reconstruction>

<quick_resume>
If user says "continue" or "go":
- Load state silently
- Determine primary action
- Execute immediately without presenting options

"Continuing from [state]... [action]"
</quick_resume>

<success_criteria>
Resume is complete when:

- [ ] STATE.md loaded (or reconstructed)
- [ ] Incomplete work detected and flagged
- [ ] Clear status presented to user
- [ ] Contextual next actions offered
- [ ] User knows exactly where project stands
- [ ] Session continuity updated
      </success_criteria>
