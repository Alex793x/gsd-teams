# Squash Merge Workflow

GSD execution creates many commits (one per task, one for metadata per plan). Squash merge consolidates all milestone work into a single clean commit on main, keeping project history readable.

## The Problem

During GSD execution, commits accumulate rapidly:

```
feat(08-03): add password reset endpoint
feat(08-03): create email templates
docs(08-03): complete password reset plan
feat(08-02): implement session management
fix(08-02): correct token expiry handling
docs(08-02): complete session plan
feat(08-01): add user registration
...
```

A single milestone might have 20-50 commits. Merging these to main creates noisy history that obscures the actual project evolution.

## The Solution

Squash all milestone commits into one commit on main:

```
feat: v1.0 MVP

Delivered: Complete authentication system with user management

Key accomplishments:
- User registration with email verification
- Session management with refresh tokens
- Password reset flow
- Protected route middleware

Phases: 1-4 (8 plans, 24 tasks)
Branch: gsd/v1.0-mvp
```

One commit represents one shippable increment.

## When to Squash

Squash merge at **milestone completion**, after `/gsd:complete-milestone`:

| Situation | Action |
|-----------|--------|
| Finishing milestone | Squash merge to main |
| Mid-milestone sync | Merge main into branch (not squash) |
| Abandoning milestone | Delete branch or keep for reference |
| Hotfix on main | Cherry-pick or direct commit, not squash |

## How to Squash Merge

### Step 1: Review Commits Being Squashed

```bash
# See what will be consolidated
git log main..HEAD --oneline

# Example output:
# abc123 docs(04-02): complete polish plan
# def456 feat(04-02): add loading states
# ghi789 feat(04-01): improve error messages
# ... (20 more commits)
```

### Step 2: Perform Squash Merge

```bash
# Switch to main
git checkout main

# Squash merge the milestone branch
git merge --squash gsd/v1.0-mvp
```

This stages all changes from the branch but does NOT create a commit yet.

### Step 3: Create Summary Commit

```bash
git commit -m "$(cat <<'EOF'
feat: v1.0 MVP

Delivered: Complete authentication system with user management

Key accomplishments:
- User registration with email verification
- Session management with refresh tokens
- Password reset flow
- Protected route middleware

Phases: 1-4 (8 plans, 24 tasks)
Branch: gsd/v1.0-mvp

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

### Step 4: Verify

```bash
# Confirm single commit on main
git log main -1 --oneline
# Output: xyz789 feat: v1.0 MVP

# Verify all code is present
git diff gsd/v1.0-mvp main
# Should show no differences
```

## Commit Message Format

Squash commits use a rich format capturing the milestone:

```
feat: v[X.Y] [Milestone Name]

Delivered: [One sentence describing what shipped]

Key accomplishments:
- [From MILESTONES.md entry]
- [Key feature 2]
- [Key feature 3]

Phases: [X]-[Y] ([N] plans, [M] tasks)
Branch: gsd/v[X.Y]-[slug]

Co-Authored-By: Claude <noreply@anthropic.com>
```

| Field | Source |
|-------|--------|
| Version | Milestone version (v1.0, v1.1) |
| Name | Milestone name from roadmap |
| Delivered | From MILESTONES.md "Delivered" field |
| Accomplishments | From MILESTONES.md "Key accomplishments" |
| Phases | Phase range from roadmap |
| Branch | The GSD branch being merged |

## Conflict Handling

### Why Conflicts Occur

Conflicts happen when:
- Main received changes after branch was created
- Same files modified on both branches
- Rebasing wasn't done during development

### Resolution Approach

**Option A: Resolve in Squash Merge**

```bash
git checkout main
git merge --squash gsd/v1.0-mvp

# If conflicts:
# 1. Edit conflicting files to resolve
# 2. Stage resolved files
git add <resolved-files>

# 3. Complete the squash commit
git commit -m "feat: v1.0 MVP ..."
```

**Option B: Rebase First, Then Squash**

```bash
# Switch to feature branch
git checkout gsd/v1.0-mvp

# Rebase onto current main
git rebase main

# Resolve conflicts during rebase (per commit)
# ... resolve ...
git add <resolved>
git rebase --continue

# Now squash merge is clean
git checkout main
git merge --squash gsd/v1.0-mvp
git commit -m "feat: v1.0 MVP ..."
```

**When to use each:**

| Situation | Approach |
|-----------|----------|
| Few conflicts, simple resolution | Resolve in squash merge |
| Many conflicts, want commit-by-commit resolution | Rebase first |
| Branch significantly diverged | Rebase first |
| Urgent merge needed | Resolve in squash merge |

### Aborting a Conflicted Merge

```bash
# Cancel the merge attempt
git merge --abort

# Back to clean main, can try rebase approach
```

## After Squash Merge

### Delete the Branch

```bash
# Local deletion
git branch -d gsd/v1.0-mvp

# Force delete if git complains about unmerged commits
# (safe because we squash merged)
git branch -D gsd/v1.0-mvp

# Remote deletion (if pushed)
git push origin --delete gsd/v1.0-mvp
```

### Tag the Release

```bash
# Create annotated tag
git tag -a v1.0 -m "v1.0 MVP - Complete auth system"

# Push tag
git push origin v1.0
```

### Push Main

```bash
git push origin main
```

## Edge Cases

### Multiple Developers on Same Milestone Branch

When multiple developers worked on the same milestone branch:

1. **All developers sync before squash:** Everyone pulls latest branch state
2. **One developer performs squash:** Typically the one completing the milestone
3. **Notify team:** Branch will be deleted, everyone should switch away

```bash
# Other developers after squash
git checkout main
git pull origin main
git branch -d gsd/v1.0-mvp  # Clean up local copy
```

### Partially Merged Work

If some phases were cherry-picked or merged to main earlier:

1. **Check what's already on main:**
   ```bash
   git log main --oneline --grep="feat(01"
   ```

2. **Squash only remaining work:**
   The squash merge will only include commits not already on main

3. **Adjust commit message:** Note that earlier phases were merged separately

### Interrupted Milestone

If milestone was abandoned or paused:

| Scenario | Action |
|----------|--------|
| Will resume later | Keep branch, document in STATE.md |
| Permanently abandoned | Delete branch, note in MILESTONES.md |
| Partial ship | Squash what's ready, new milestone for rest |

## Integration with /gsd:complete-milestone

The `complete-milestone` workflow uses squash merge:

1. **Pre-squash review:** Shows commits being squashed
2. **Offers merge options:** Squash (recommended), merge with history, or manual
3. **Generates commit message:** From milestone stats
4. **Conflict guidance:** Provides resolution steps if conflicts occur
5. **Post-squash verification:** Confirms clean merge

See `get-shit-done/workflows/complete-milestone.md` step `handle_branches` for implementation.

## Best Practices

### Do

- Squash at milestone boundaries, not phase boundaries
- Include Co-Authored-By for AI-assisted development
- Write descriptive squash commit messages
- Delete branches after successful squash
- Tag releases after squash to main

### Don't

- Squash individual phases (too granular)
- Force push main (breaks team workflow)
- Squash without verifying all tests pass
- Leave stale milestone branches hanging around
- Squash to main without team coordination

## Benefits of Squash Merge

| Benefit | Explanation |
|---------|-------------|
| Clean history | One commit per milestone on main |
| Easy bisect | `git bisect` finds broken milestones, not tasks |
| Simple revert | Revert entire milestone with one command |
| Readable log | `git log main` shows project evolution |
| PR clarity | PR represents shippable increment |
