<purpose>

Start a new milestone by creating a dedicated branch and setting up the milestone structure. This creates the foundation for the next shippable increment (v1.0, v1.1, v2.0).

GSD handles branch creation. Developer handles merging when complete.

</purpose>

<required_reading>

**Read these files NOW:**

1. references/branch-namespacing.md
2. templates/milestone.md
3. `.planning/MILESTONES.md` (if exists)
4. `.planning/PROJECT.md`

</required_reading>

<process>

<step name="validate_context">

Verify GSD project exists and check current state:

```bash
# Check .planning/ exists
ls .planning/PROJECT.md 2>/dev/null || echo "NO_PROJECT"

# Check current branch
git branch --show-current

# Check if on a gsd/ branch already
git branch --show-current | grep -q "^gsd/" && echo "ON_GSD_BRANCH" || echo "NOT_ON_GSD_BRANCH"

# Check for existing milestones
ls .planning/MILESTONES.md 2>/dev/null || echo "NO_MILESTONES"
```

**If NO_PROJECT:**

```
This doesn't appear to be a GSD project.
Run `/gsd:new-project` first to initialize.
```

Stop workflow.

**If ON_GSD_BRANCH:**

```
You're currently on branch: [branch-name]

This appears to be an active milestone branch.
Complete the current milestone first with `/gsd:complete-milestone`.

Options:
1. Complete current milestone first (recommended)
2. Abandon current milestone and start fresh
3. Cancel
```

Wait for user choice. If "abandon", proceed. If "complete" or "cancel", stop.

**If NOT_ON_GSD_BRANCH:**

Proceed to next step.

</step>

<step name="gather_milestone_info">

Collect milestone details from user:

```
## New Milestone Setup

What version is this milestone?
Examples: v0.2.0, v1.0, v1.1, v2.0

What's the milestone name/theme?
Examples: "Team Extension", "MVP", "Security Hardening"
```

<config-check>

```bash
cat .planning/config.json 2>/dev/null
```

</config-check>

<if mode="yolo">

If user provided version and name in initial message, use those directly.
Otherwise prompt once for both.

```
What version and name for this milestone?
Format: v[X.Y] [Name]
Example: v1.0 MVP
```

</if>

<if mode="interactive" OR="custom">

Ask version and name separately, confirm before proceeding.

</if>

**Capture:**
- VERSION: e.g., "0.2.0" (without v prefix for internal use)
- NAME: e.g., "Team Extension"
- SLUG: generated from NAME (lowercase, kebab-case)

**Slug generation:**

```bash
# Example: "Team Extension" -> "team-extension"
SLUG=$(echo "[NAME]" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9 ]//g' | sed 's/  */ /g' | sed 's/ /-/g')
```

**Branch name:** `gsd/v[VERSION]-[SLUG]`

Present confirmation:

```
Milestone: v[VERSION] [NAME]
Branch: gsd/v[VERSION]-[SLUG]

Ready to create? (yes/no)
```

</step>

<step name="create_branch">

Create the milestone branch:

```bash
# Ensure we're starting from main with latest
CURRENT_BRANCH=$(git branch --show-current)

# Check if main exists
git rev-parse --verify main >/dev/null 2>&1 && BASE_BRANCH="main" || BASE_BRANCH="$CURRENT_BRANCH"

# Create new branch
git checkout -b gsd/v[VERSION]-[SLUG]

# Verify
git branch --show-current
```

Report:

```
Created branch: gsd/v[VERSION]-[SLUG]
Based on: [BASE_BRANCH]
```

**If branch creation fails** (branch exists, git error):

```
Branch creation failed: [error]

Options:
1. Use existing branch (if it exists)
2. Choose different version/name
3. Cancel
```

</step>

<step name="setup_milestone_structure">

Initialize or update planning files for new milestone.

**Check for existing REQUIREMENTS.md:**

```bash
ls .planning/REQUIREMENTS.md 2>/dev/null || echo "NO_REQUIREMENTS"
```

**If NO_REQUIREMENTS or empty:**

Create `.planning/REQUIREMENTS.md`:

```markdown
# Requirements: v[VERSION] [NAME]

## Overview

[Brief description of what this milestone delivers]

## Requirements

### Must Have (P0)

- [ ] [Requirement 1]
- [ ] [Requirement 2]

### Should Have (P1)

- [ ] [Requirement 1]

### Nice to Have (P2)

- [ ] [Requirement 1]

## Acceptance Criteria

[How we know the milestone is complete]

## Out of Scope

[What this milestone explicitly does NOT include]

---
*Milestone: v[VERSION] [NAME]*
*Created: [DATE]*
```

Prompt user:

```
REQUIREMENTS.md created with template.
Please fill in requirements before planning phases.

Options:
1. Fill requirements now (recommended)
2. Continue with placeholder requirements
```

**If REQUIREMENTS.md exists:**

```
Existing REQUIREMENTS.md found.
This may be from a previous milestone that wasn't properly archived.

Options:
1. Archive existing and create fresh (recommended)
2. Keep existing requirements
3. View existing requirements first
```

If "archive": Move to `.planning/milestones/[old-version]-REQUIREMENTS.md`

**Update ROADMAP.md milestone header:**

If ROADMAP.md exists, add milestone header if not present:

```bash
cat .planning/ROADMAP.md
```

Add to Overview section if needed:

```markdown
## Current Milestone

**v[VERSION] [NAME]** - [STATUS: Planning/In Progress]
Branch: `gsd/v[VERSION]-[SLUG]`
```

**Initialize STATE.md for new milestone:**

Update `.planning/STATE.md`:

```markdown
## Project Reference

See: .planning/PROJECT.md (updated [DATE])

**Core value:** [from PROJECT.md]
**Current focus:** v[VERSION] [NAME] - Phase planning

## Current Position

Phase: [Next phase number] of [Total]
Plan: Not started
Status: Ready to plan
Last activity: [DATE] - Started v[VERSION] [NAME] milestone

Progress: [progress bar based on phases]
```

</step>

<step name="update_project">

Update PROJECT.md with new milestone context:

```bash
cat .planning/PROJECT.md
```

**Add to Active Requirements section** (if using that pattern):

Any new requirements for this milestone should be captured in the Active section.

**Update Context section:**

```markdown
## Context

Currently working on: v[VERSION] [NAME]
Branch: gsd/v[VERSION]-[SLUG]
```

**Update "Last updated" footer:**

```markdown
---
*Last updated: [DATE] after starting v[VERSION] [NAME]*
```

</step>

<step name="commit_milestone_setup">

Commit the milestone initialization:

```bash
# Stage planning files
git add .planning/REQUIREMENTS.md
git add .planning/ROADMAP.md
git add .planning/STATE.md
git add .planning/PROJECT.md

# Commit
git commit -m "$(cat <<'EOF'
chore: initialize v[VERSION] [NAME] milestone

- Created branch gsd/v[VERSION]-[SLUG]
- Set up REQUIREMENTS.md for milestone
- Updated STATE.md with milestone context
- Updated PROJECT.md with current focus

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

</step>

<step name="present_next_steps">

```
## Milestone Ready

**v[VERSION] [NAME]** initialized

Branch: gsd/v[VERSION]-[SLUG]
Requirements: .planning/REQUIREMENTS.md

---

## Next Steps

1. **Define requirements** - Complete REQUIREMENTS.md with P0/P1/P2 items
2. **Plan phases** - `/gsd:create-roadmap` or manually update ROADMAP.md
3. **Start execution** - `/gsd:plan-phase` for first phase

---

Quick start:
- If requirements are clear: `/gsd:create-roadmap`
- If phases are already defined: `/gsd:plan-phase`
- If more context needed: `/gsd:discuss-milestone`

---
```

</step>

</process>

<first_milestone_behavior>

When this is the first milestone of a project (no prior milestones):

- No warning about completing previous milestone
- VERSION typically starts at v1.0 or v0.1.0
- Branch created from current HEAD (usually main after `/gsd:new-project`)
- REQUIREMENTS.md created fresh (no archival needed)

</first_milestone_behavior>

<branching_strategy>

**Branch creation:**
- Always use `git checkout -b` (creates and switches)
- Prefer branching from `main` if it exists
- Fall back to current branch if `main` doesn't exist

**Branch naming:**
- Prefix: `gsd/`
- Version: `v[X.Y.Z]` or `v[X.Y]`
- Slug: kebab-case from milestone name
- Full pattern: `gsd/v{version}-{feature-slug}`

**No merge automation:**
- This workflow creates branches only
- Merging is handled by `/gsd:complete-milestone` (Phase 7)
- Developer controls merge timing and strategy

</branching_strategy>

<success_criteria>

Milestone initialization is successful when:

- [ ] Branch `gsd/v[VERSION]-[SLUG]` created and checked out
- [ ] REQUIREMENTS.md exists with milestone template
- [ ] STATE.md updated with milestone context
- [ ] PROJECT.md updated with current focus
- [ ] Initialization commit created on new branch
- [ ] User knows next step (requirements → roadmap → phases)

</success_criteria>
