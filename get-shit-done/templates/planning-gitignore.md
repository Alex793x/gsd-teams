# Planning Directory .gitignore Template

Template for `.planning/.gitignore` - excludes ephemeral state from git tracking to enable team collaboration.

## Template Content

Copy this content to `.planning/.gitignore`:

```gitignore
# GSD Ephemeral State
# These files are per-developer and should not be committed
# See: get-shit-done/references/state-architecture.md

# Current session state (rebuilt on hydration)
STATE.md

# Per-developer session directories
sessions/

# Agent execution tracking (session-specific)
agent-history.json
current-agent-id.txt

# Editor/OS artifacts that may appear
.DS_Store
*.swp
*~
```

## When This File Is Created

**New projects with team mode:**
- `/gsd:new-project` creates this file automatically when team collaboration is enabled

**Existing projects adopting team support:**
- Users create manually by copying the template content above
- Run `git rm --cached .planning/STATE.md` to stop tracking existing STATE.md

**Solo projects:**
- This file is optional - solo projects work without it
- Adding it later enables team collaboration without breaking existing workflow

## Pattern Explanations

| Pattern | Purpose |
|---------|---------|
| `STATE.md` | Session position tracking - per-developer, rebuilt on hydration |
| `sessions/` | Per-developer working directories - scratch space and active plan copies |
| `agent-history.json` | Subagent tracking for resume capability - transient execution state |
| `current-agent-id.txt` | Currently running agent ID - only relevant during active execution |
| `.DS_Store` | macOS directory metadata - never belongs in repos |
| `*.swp` | Vim swap files - editor artifacts |
| `*~` | Backup files from various editors |

## What Stays Tracked

These files remain git-tracked (NOT in .gitignore):

| File | Why Tracked |
|------|-------------|
| `PROJECT.md` | Team alignment on project vision |
| `ROADMAP.md` | Shared execution plan |
| `ISSUES.md` | Team visibility into deferred work |
| `config.json` | Consistent workflow settings |
| `phases/*/PLAN.md` | Work specifications |
| `phases/*/SUMMARY.md` | Work record (history) |
| `codebase/*.md` | Shared architecture understanding |

## Usage by Workflows

**`/gsd:new-project` (future update):**
```
1. Create .planning/ directory structure
2. If team mode enabled:
   - Write .planning/.gitignore from this template
   - Initial commit includes .gitignore
3. STATE.md created but not committed (already gitignored)
```

**Hydration workflows:**
```
1. Check if STATE.md exists
2. If not (or stale), reconstruct from ROADMAP.md + latest SUMMARY.md
3. STATE.md is local-only, never needs merge resolution
```

## Backward Compatibility

**Solo projects without .gitignore:**
- STATE.md gets committed (existing behavior)
- No functional difference for solo use
- Can add .gitignore later to enable team collaboration

**Adding .gitignore to existing project:**
1. Create `.planning/.gitignore` with template content
2. Remove STATE.md from git: `git rm --cached .planning/STATE.md`
3. Commit: `git commit -m "chore: enable team collaboration mode"`
4. Each developer's STATE.md is now local
