# Branch Namespacing

GSD's team extension uses **milestone-level branches** following industry-standard feature branch patterns. When a developer starts working on a new milestone (like v0.2.0), GSD creates a dedicated branch automatically.

## Naming Convention

Branch names follow the pattern: `gsd/v{version}-{feature-slug}`

| Component | Description | Example |
|-----------|-------------|---------|
| `gsd/` | Prefix identifying GSD-managed branches | `gsd/` |
| `v{version}` | Milestone version number | `v0.2.0`, `v1.0`, `v1.1` |
| `-{feature-slug}` | Kebab-case descriptor from milestone name | `-team-extension`, `-mvp`, `-security-hardening` |

### Examples

| Milestone | Branch Name |
|-----------|-------------|
| v0.2.0 Team Extension | `gsd/v0.2.0-team-extension` |
| v1.0 MVP | `gsd/v1.0-mvp` |
| v1.1 Security Hardening | `gsd/v1.1-security-hardening` |
| v2.0 iOS Launch | `gsd/v2.0-ios-launch` |

### Slug Generation Rules

The feature slug is generated from the milestone name:
1. Convert to lowercase
2. Replace spaces with hyphens
3. Strip special characters (keep only alphanumeric and hyphens)
4. Collapse multiple hyphens to single hyphen
5. Trim leading/trailing hyphens

| Milestone Name | Generated Slug |
|----------------|----------------|
| "Team Extension" | `team-extension` |
| "MVP" | `mvp` |
| "Security & Hardening" | `security-hardening` |
| "iOS Launch (Beta)" | `ios-launch-beta` |

## Rationale

### Why Milestone-Level, Not Phase-Level?

**Industry standard feature branch pattern:**
- Teams typically use one branch per deliverable/release
- Milestone = a shippable increment (v1.0, v1.1, v2.0)
- Phases are internal execution units, not deliverables

**Longer-lived, meaningful branches:**
- A milestone might span 4-8 phases over days or weeks
- Branches represent coherent units of work the team understands
- Branch names map to version numbers and release notes

**Avoids micro-branch proliferation:**
- Phase-level would create many short-lived branches (hours, not days)
- More branch management overhead
- Less alignment with standard git workflows

**Matches team workflow expectations:**
- Most teams already think in terms of "the v1.1 branch"
- PR reviews happen at milestone boundaries, not phase boundaries
- CI/CD typically triggers on branches, not internal phases

### Why Not Branch-per-Developer?

Developer isolation is achieved through **ephemeral session state**, not branches:
- `.planning/sessions/{user}/` provides per-developer working context
- Branches isolate code changes (milestone-scoped), not session state
- Multiple developers can work on the same milestone branch

## Usage Patterns

### Branch Creation

Branches are created automatically by `/gsd:new-milestone`:

```bash
# When starting milestone v0.2.0 Team Extension
git checkout -b gsd/v0.2.0-team-extension

# From main branch (typical case)
git checkout main
git pull origin main
git checkout -b gsd/v0.2.0-team-extension
```

### All Phases on Same Branch

All phases within a milestone execute on the same branch:

```
gsd/v0.2.0-team-extension
  ├── Phase 1: State Architecture (complete)
  ├── Phase 2: User Identity (complete)
  ├── Phase 3: Session Structure (complete)
  ├── Phase 4: Session Hydration (in progress)
  └── ... remaining phases
```

### Developer Merging

GSD creates branches but **does not automate merging**:
- Developer decides when milestone is ready to merge
- Developer handles merge conflicts with full context
- Developer chooses merge strategy (squash, no-ff, etc.)

**Why no automated merging?**
- Merge conflicts require human judgment
- Merge timing depends on project workflow (PRs, reviews, CI)
- Different teams have different merge policies

**Phase 7 (Squash Merge Workflow)** provides guidance and tooling for clean merges, but the developer controls when and how.

## Best Practices

### One Milestone = One Branch

- Create branch when starting milestone: `/gsd:new-milestone`
- Complete all phases on that branch
- Merge to main when milestone ships: `/gsd:complete-milestone`

### Keep Branches Current

Developer responsibility to stay in sync with main:

```bash
# Periodically sync with main
git fetch origin main
git merge origin/main
# Or: git rebase origin/main (if team prefers)
```

**When to sync:**
- Before starting a new phase
- After team members merge other work to main
- When conflicts are small (don't wait until they're large)

### Use Squash Merge for Clean History

Recommended merge strategy (documented in Phase 7):

```bash
git checkout main
git merge --squash gsd/v0.2.0-team-extension
git commit -m "feat: v0.2.0 Team Extension"
```

**Benefits:**
- Single commit per milestone on main
- No micro-commits from GSD task execution
- Clean, readable project history
- Easy to revert if needed

### Branch Lifecycle

```
1. [CREATE] /gsd:new-milestone creates branch from main
2. [WORK] Execute phases on branch (commits accumulate)
3. [SYNC] Periodically merge main into branch (developer)
4. [VERIFY] /gsd:verify-work confirms milestone ready
5. [MERGE] Squash merge to main (developer)
6. [CLEANUP] Delete feature branch (developer)
```

## Integration with GSD Workflows

| Workflow | Branch Behavior |
|----------|-----------------|
| `/gsd:new-milestone` | Creates branch, prompts for milestone name |
| `/gsd:plan-phase` | Works on current branch |
| `/gsd:execute-plan` | Commits to current branch |
| `/gsd:complete-milestone` | Offers merge guidance, does not auto-merge |
| `/gsd:progress` | Shows current branch in status |

## Edge Cases

### First Milestone of Project

When no prior milestones exist:
- Branch created from current HEAD (usually main after `/gsd:new-project`)
- No warning about completing previous milestone

### Already on a GSD Branch

When user runs `/gsd:new-milestone` while on a `gsd/` branch:
- Warn user: "You're on branch gsd/v0.1.0-initial. Complete current milestone first?"
- Offer options: complete current, abandon current, or cancel

### Multiple Developers on Same Milestone

Supported workflow:
- Both developers work on `gsd/v0.2.0-team-extension`
- Each has their own `.planning/sessions/{user}/` state
- Git handles code merging when both push
- GSD doesn't coordinate concurrent execution (async via git)

## Design Decisions

| Decision | Rationale |
|----------|-----------|
| Milestone-level, not phase-level | Industry standard, avoids micro-branches |
| `gsd/` prefix | Clearly identifies GSD-managed branches |
| Version in branch name | Maps branches to releases |
| Slug from milestone name | Human-readable, descriptive branches |
| No automated merging | Requires human judgment for conflicts |
| Developer controls sync timing | Different teams have different workflows |
