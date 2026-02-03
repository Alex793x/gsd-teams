# Backward Compatibility Verification

Manual verification scenarios to confirm GSD team extension maintains backward compatibility with solo projects. These are practical tests to run when validating the implementation.

## Verification Scenarios

### Scenario A: Fresh Solo Project

**Purpose:** Verify new solo projects work without team features.

**Setup:**
```bash
# Create a new directory
mkdir test-solo-project
cd test-solo-project
git init
git config user.name "Test User"
```

**Test steps:**

1. Run `/gsd:new-project` without team mode
2. Check STATE.md is created and git-tracked:
   ```bash
   ls -la .planning/STATE.md
   git status .planning/STATE.md
   # Should show as new file, not gitignored
   ```
3. Verify no `.planning/.gitignore` was created:
   ```bash
   ls .planning/.gitignore
   # Should fail - file doesn't exist
   ```
4. Run `/gsd:execute-plan` on first plan
5. Verify sessions/ was created but workflow completed normally:
   ```bash
   ls -la .planning/sessions/
   # Directory exists with user folder
   ```
6. Verify SUMMARY.md was created for the plan

**Expected result:** All steps complete without errors. Solo workflow is unchanged.

---

### Scenario B: Existing Solo Project (No Sessions)

**Purpose:** Verify existing solo projects get session support seamlessly.

**Setup:**
```bash
# Use an existing .planning/ directory with plans but no sessions/
cd existing-solo-project
ls .planning/
# Should have: PROJECT.md, ROADMAP.md, STATE.md, phases/
ls .planning/sessions/
# Should fail - directory doesn't exist yet
```

**Test steps:**

1. Run `/gsd:progress`
2. Verify session info shows gracefully:
   - Should not error
   - May show "no active session" or empty session info
   - Progress report completes normally
3. Run `/gsd:execute-plan`
4. Verify sessions/ was created:
   ```bash
   ls -la .planning/sessions/
   # Now exists with user directory
   ```
5. Verify workflow completes and SUMMARY.md created

**Expected result:** Missing sessions/ handled gracefully. Created on first execute.

---

### Scenario C: Git user.name Not Set

**Purpose:** Verify fallback identity works when git config is missing.

**Setup:**
```bash
# Temporarily unset git user.name
cd test-project
git config --unset user.name
git config user.name
# Should return empty
```

**Test steps:**

1. Run `/gsd:execute-plan`
2. Verify fallback to whoami works:
   ```bash
   ls .planning/sessions/
   # Should have directory named after whoami output
   whoami
   # Compare - directory name should match (sanitized)
   ```
3. Verify workflow completes with alternate user identity
4. Check SUMMARY.md and commits are created

**Cleanup:**
```bash
# Restore git user.name
git config user.name "Your Name"
```

**Expected result:** Workflow completes using whoami as identity source. No errors.

---

### Scenario D: STATE.md Tracked in Git

**Purpose:** Verify solo projects with committed STATE.md continue working.

**Setup:**
```bash
# Project with STATE.md committed (no .gitignore)
cd solo-project-with-tracked-state
git log --oneline .planning/STATE.md
# Should show commits - STATE.md is tracked
cat .planning/.gitignore 2>/dev/null
# Should fail or not contain STATE.md
```

**Test steps:**

1. Run `/gsd:execute-plan`
2. Verify workflow completes without errors
3. Run `/gsd:progress`
4. Verify progress report works normally
5. Run `/gsd:resume-work` (if applicable)
6. Verify all workflows complete without errors
7. Check STATE.md updates are included in commits:
   ```bash
   git status .planning/STATE.md
   # May show as modified (ready to commit)
   git diff .planning/STATE.md
   # Shows position updates
   ```

**Expected result:** All workflows work. STATE.md changes can be committed normally.

---

## Expected Behaviors

These behaviors MUST hold for backward compatibility:

| Behavior | Description |
|----------|-------------|
| No sessions/ errors | No workflow should error due to missing sessions/ directory |
| No .gitignore errors | No workflow should error due to missing .planning/.gitignore |
| User identity resolves | User identity should ALWAYS resolve (never undefined/empty) |
| Session ops transparent | Session operations create if needed, use if exists |
| STATE.md works both ways | Works whether git-tracked or gitignored |
| Commits succeed | Git operations complete regardless of .gitignore presence |

## Warning Signs (NOT Expected)

If you observe any of these, backward compatibility is broken:

| Warning Sign | What It Indicates |
|--------------|-------------------|
| "session file not found" errors blocking execution | Session operations not defensive enough |
| "user identity undefined" errors | Fallback chain broken |
| Git commit failures due to gitignore conflicts | Gitignore patterns too aggressive |
| Workflow divergence beyond intentional differences | Unintended code path differences |
| "directory not found" errors for sessions/ | mkdir -p not being used |
| Empty user directory names | Sanitization returning empty string |

## Quick Verification Checklist

Run these commands to quickly verify backward compatibility:

```bash
# 1. User identity never fails
get_session_user() {
  local raw
  raw=$(git config user.name 2>/dev/null)
  [ -z "$raw" ] && raw=$(whoami 2>/dev/null)
  [ -z "$raw" ] && raw=$(hostname -s 2>/dev/null)
  [ -z "$raw" ] && raw="unknown"
  echo "$raw" | tr '[:upper:]' '[:lower:]' | tr ' ' '-' | tr -cd 'a-z0-9-' | sed 's/--*/-/g;s/^-//;s/-$//'
}
echo "User: $(get_session_user)"
# Should always print something

# 2. Session directory creation is safe
SESSION_DIR=".planning/sessions/$(get_session_user)"
mkdir -p "$SESSION_DIR"
echo $?
# Should be 0 (success) even if directory exists

# 3. File guards work correctly
SESSION_FILE="${SESSION_DIR}/current-plan.md"
if [ -f "$SESSION_FILE" ]; then
  echo "Session exists"
else
  echo "No session (this is fine)"
fi
# Should not error, regardless of file existence

# 4. Git operations work without .gitignore
git status .planning/ 2>/dev/null
# Should not error even without .gitignore
```

## Cross-References

- **Backward compatibility design:** See `get-shit-done/references/backward-compatibility.md`
- **Defensive patterns:** See `get-shit-done/references/session-hydration.md` for implementation details
- **User identity:** See `get-shit-done/references/user-identity.md` for fallback chain
