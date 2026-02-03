# State Architecture

GSD's team extension splits `.planning/` into two categories: **global state** (git-tracked, team source of truth) and **ephemeral state** (gitignored, per-developer working context). This enables multiple developers to use GSD simultaneously without conflicts.

## Global State (Git-Tracked)

These files are shared across the team and committed to the repository. Merge conflicts in these files are meaningful and should be resolved deliberately.

| File/Directory | Purpose | Why Global |
|----------------|---------|------------|
| `.planning/PROJECT.md` | Project vision, requirements, constraints | Team alignment on goals |
| `.planning/ROADMAP.md` | Phase breakdown and status | Shared execution plan |
| `.planning/ISSUES.md` | Deferred issues and enhancements | Team visibility into tech debt |
| `.planning/config.json` | Workflow settings (model profile, depth) | Consistent behavior across sessions |
| `.planning/phases/*/PLAN.md` | Execution plans | Work specification (immutable once written) |
| `.planning/phases/*/SUMMARY.md` | Execution results | Work record (history of what was built) |
| `.planning/codebase/` | Codebase analysis documents | Shared understanding of architecture |

**Key principle:** Global state represents the *team's shared understanding* and *work record*. If two developers disagree about a ROADMAP change, that's a real conflict requiring discussion.

## Ephemeral State (Gitignored)

These files are per-developer and should not be committed. They're regenerated automatically when a session starts (hydration).

| File/Directory | Purpose | Why Ephemeral |
|----------------|---------|---------------|
| `.planning/STATE.md` | Current position, session context | Per-developer progress tracking |
| `.planning/sessions/{user}/` | Session-specific working files | Developer isolation |
| `.planning/sessions/{user}/current-plan.md` | Active plan copy with task progress | Working state during execution |
| `.planning/sessions/{user}/context.md` | Session-specific notes | Scratch space |
| `.planning/agent-history.json` | Subagent tracking for resume | Session-specific execution state |
| `.planning/current-agent-id.txt` | Currently running agent ID | Transient execution state |

**Key principle:** Ephemeral state represents *where you are right now* in your work. If two developers are at different points in the roadmap, that's normal - not a conflict.

## Rationale

### Why This Split?

**Without split state:**
- Developer A runs `/gsd:execute-plan` and updates STATE.md
- Developer B runs `/gsd:plan-phase` and updates STATE.md
- Both commit - merge conflict on STATE.md
- Conflict is noise: neither developer's position affects the other

**With split state:**
- Developer A's STATE.md is gitignored - their position is local
- Developer B's STATE.md is gitignored - their position is local
- Both can work independently
- Only meaningful changes (PLAN.md, SUMMARY.md) get committed
- Merge conflicts only occur when they matter

### Why Not Gitignore Everything?

The work record (PLAN.md, SUMMARY.md) must be shared because:
- It documents what was built and why
- It enables team members to understand each other's work
- It provides context for future AI sessions
- It creates an audit trail of project evolution

## Migration

### Existing Solo Projects

**No breaking changes.** Solo projects can continue unchanged:
- STATE.md works exactly as before
- No `.planning/.gitignore` required
- Adding team support is opt-in

### Adopting Team Support

To enable team collaboration on an existing project:

1. Create `.planning/.gitignore` (use template from `/gsd:new-project`)
2. Remove STATE.md from git tracking: `git rm --cached .planning/STATE.md`
3. Commit the .gitignore
4. Each developer's STATE.md will be local from now on

### New Team Projects

`/gsd:new-project` creates `.planning/.gitignore` automatically when team mode is enabled.

## Hydration Pattern

When a GSD session starts, ephemeral state is reconstructed from global state. This is called "hydration."

### How It Works

1. **Session start** - Developer runs a GSD command
2. **Check STATE.md** - If missing or stale, hydrate from global state
3. **Reconstruct position** - Read ROADMAP.md, find latest SUMMARY.md, determine current position
4. **Rebuild context** - Load accumulated decisions and deferred issues from summaries
5. **Ready to work** - STATE.md now reflects current project state

### Why Hydration?

Claude sessions don't persist between conversations. By reconstructing ephemeral state from the persistent work record:
- Any developer can pick up where the team left off
- No "stale session" problems
- Global state is the source of truth
- Ephemeral state is just a cached view of global state

**Implementation details:** See Phase 4 (Session Hydration) in the roadmap.

## Directory Structure

After team support is enabled:

```
.planning/
├── .gitignore              # Excludes ephemeral state
├── PROJECT.md              # [global] Project vision
├── ROADMAP.md              # [global] Phase breakdown
├── ISSUES.md               # [global] Deferred issues
├── config.json             # [global] Workflow settings
├── STATE.md                # [ephemeral] Current position
├── agent-history.json      # [ephemeral] Agent tracking
├── current-agent-id.txt    # [ephemeral] Running agent
├── sessions/               # [ephemeral] Per-developer
│   └── {user}/
│       ├── current-plan.md
│       └── context.md
├── phases/                 # [global] Work record
│   ├── 01-foundation/
│   │   ├── 01-01-PLAN.md
│   │   └── 01-01-SUMMARY.md
│   └── 02-auth/
│       ├── 02-01-PLAN.md
│       └── 02-01-SUMMARY.md
└── codebase/               # [global] Architecture docs
    ├── ARCHITECTURE.md
    └── STRUCTURE.md
```

## Design Decisions

| Decision | Rationale |
|----------|-----------|
| STATE.md is ephemeral | Position tracking is per-developer, not shared |
| PLAN.md is global | Plans are specifications - changing them affects everyone |
| SUMMARY.md is global | Summaries are the work record - history should be shared |
| sessions/ is ephemeral | Per-developer scratch space shouldn't pollute team repo |
| config.json is global | Workflow settings should be consistent across team |
