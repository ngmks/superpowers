---
name: using-git-worktrees
description: Use when starting feature work that needs isolation from current workspace or before executing implementation plans - creates isolated git worktrees with smart directory selection and safety verification
---

# Using Git Worktrees

## Overview

Git worktrees create isolated workspaces sharing the same repository, allowing work on multiple branches simultaneously without switching.

**Core principle:** Systematic directory selection + safety verification = reliable isolation.

**Announce at start:** "I'm using the using-git-worktrees skill to set up an isolated workspace."

## ⚠️ Critical: Working Directory Behavior

**Claude Code's PWD does NOT change between tool calls.**

**The scenario:** Claude Code starts in the main project directory. You create a worktree at `.worktrees/feat-auth`. Running `cd` does NOT move you to the worktree for future tool calls.

**The solution:** Chain commands with `&&` to ensure they run in the worktree directory.

```bash
# ✅ Correct - chain commands together
cd "$WORKTREE_PATH" && npm install && npm test

# ❌ Wrong - cd doesn't persist
cd "$WORKTREE_PATH"
npm install  # Runs in original PWD, not worktree!
```

**For file operations (Read/Edit/Write tools):**
- Use absolute paths: `$WORKTREE_PATH/src/file.js`
- Or use relative paths if you understand where Claude Code's PWD is

**Remember:** You are always in Claude Code's original PWD unless you chain commands with `&&`.

## Pre-Flight Checks

**CRITICAL: Always run these checks before creating a worktree.**

### 1. Detect If Already In A Worktree

```bash
# Check if in a worktree by comparing git-dir vs git-common-dir
# Works reliably from any subdirectory, not just repo root
git_dir=$(git rev-parse --path-format=absolute --git-dir 2>/dev/null)
git_common_dir=$(git rev-parse --path-format=absolute --git-common-dir 2>/dev/null)

if [[ -n "$git_dir" && -n "$git_common_dir" && "$git_dir" != "$git_common_dir" ]]; then
    echo "⚠️  Already in a worktree. Navigating to main repository..."
    main_repo=$(echo "$git_common_dir" | sed 's|/.git$||')
    cd "$main_repo"
    echo "Now in: $(pwd)"
fi
```

**Why critical:** Creating a worktree from within another worktree causes nested creation at wrong paths.

**Why this method:** Comparing `git-dir` vs `git-common-dir` works from any subdirectory. When in a worktree, these differ; in the main repository, they're identical. The old `[ -f .git ]` check only worked at the repository root.

### 2. Detect Repository Type

```bash
# Check if this is a bare repository
is_bare=$(git config --get core.bare 2>/dev/null)
if [[ "$is_bare" == "true" ]]; then
    echo "Note: This is a bare repository. All work must happen in worktrees."
    echo "Primary workspace should be: .worktrees/main"
fi
```

## Directory Selection Process

Follow this priority order:

### 1. Check Existing Directories

```bash
# Check in priority order
ls -d .worktrees 2>/dev/null     # Preferred (hidden)
ls -d worktrees 2>/dev/null      # Alternative
```

**If found:** Use that directory. If both exist, `.worktrees` wins.

### 2. Check CLAUDE.md

```bash
grep -i "worktree.*director" CLAUDE.md 2>/dev/null
```

**If preference specified:** Use it without asking.

### 3. Ask User

If no directory exists and no CLAUDE.md preference:

```
No worktree directory found. Where should I create worktrees?

1. .worktrees/ (project-local, hidden)
2. ~/.config/superpowers/worktrees/<project-name>/ (global location)

Which would you prefer?
```

## Safety Verification

### For Project-Local Directories (.worktrees or worktrees)

**MUST verify directory is ignored before creating worktree:**

```bash
# Check if directory is ignored (respects local, global, and system gitignore)
git check-ignore -q .worktrees 2>/dev/null || git check-ignore -q worktrees 2>/dev/null
```

**If NOT ignored:**

Per Jesse's rule "Fix broken things immediately":
1. Add appropriate line to .gitignore
2. Commit the change
3. Proceed with worktree creation

**Why critical:** Prevents accidentally committing worktree contents to repository.

### For Global Directory (~/.config/superpowers/worktrees)

No .gitignore verification needed - outside project entirely.

## Creation Steps

### 1. Detect Project Name

```bash
project=$(basename "$(git rev-parse --show-toplevel)")
```

### 2. Create Worktree and Capture Path

**CRITICAL: Always use absolute paths to prevent nested worktree creation.**

```bash
# Get absolute path to main repository
main_repo=$(git rev-parse --show-toplevel)

# Determine full worktree path (always absolute) and store in WORKTREE_PATH
case $LOCATION in
  .worktrees|worktrees)
    WORKTREE_PATH="$main_repo/$LOCATION/$BRANCH_NAME"
    ;;
  ~/.config/superpowers/worktrees/*)
    WORKTREE_PATH="$HOME/.config/superpowers/worktrees/$project/$BRANCH_NAME"
    ;;
esac

# Create worktree with new branch using absolute path
git worktree add "$WORKTREE_PATH" -b "$BRANCH_NAME"
```

**Why absolute paths:** Relative paths like `../.worktrees/` can create worktrees at wrong locations when already inside a worktree.

### 3. Run Project Setup (Chained Commands)

Auto-detect and run appropriate setup using `&&` to chain commands:

```bash
# Node.js
cd "$WORKTREE_PATH" && [ -f package.json ] && npm install

# Rust
cd "$WORKTREE_PATH" && [ -f Cargo.toml ] && cargo build

# Python
cd "$WORKTREE_PATH" && [ -f requirements.txt ] && pip install -r requirements.txt
cd "$WORKTREE_PATH" && [ -f pyproject.toml ] && poetry install

# Go
cd "$WORKTREE_PATH" && [ -f go.mod ] && go mod download
```

### 4. Verify Clean Baseline (Chained Commands)

Run tests to ensure worktree starts clean:

```bash
# Examples - use project-appropriate command (always chain with cd)
cd "$WORKTREE_PATH" && npm test
cd "$WORKTREE_PATH" && cargo test
cd "$WORKTREE_PATH" && pytest
cd "$WORKTREE_PATH" && go test ./...
```

**If tests fail:** Report failures, ask whether to proceed or investigate.

**If tests pass:** Report ready.

### 5. Report Location

```
Worktree ready at <full-path>
Tests passing (<N> tests, 0 failures)
Ready to implement <feature-name>
```

## Quick Reference

| Situation | Action |
|-----------|--------|
| `.worktrees/` exists | Use it (verify ignored) |
| `worktrees/` exists | Use it (verify ignored) |
| Both exist | Use `.worktrees/` |
| Neither exists | Check CLAUDE.md → Ask user |
| Directory not ignored | Add to .gitignore + commit |
| Tests fail during baseline | Report failures + ask |
| No package.json/Cargo.toml | Skip dependency install |

## Common Mistakes

### Using relative paths for file operations

- **Problem:** `Read: src/file.js` assumes you're "in" the worktree (you're not)
- **Fix:** Always use absolute paths: `Read: $WORKTREE_PATH/src/file.js`

### Expecting `cd` to persist across tool calls

- **Problem:** Running `cd "$WORKTREE_PATH"` in one tool call doesn't affect the next
- **Fix:** Chain commands: `cd "$WORKTREE_PATH" && npm test`

### Not capturing worktree path in a variable

- **Problem:** Hard to use absolute paths consistently without storing the path
- **Fix:** Store path in `$WORKTREE_PATH` variable immediately after creation

### Skipping ignore verification

- **Problem:** Worktree contents get tracked, pollute git status
- **Fix:** Always use `git check-ignore` before creating project-local worktree

### Assuming directory location

- **Problem:** Creates inconsistency, violates project conventions
- **Fix:** Follow priority: existing > CLAUDE.md > ask

### Proceeding with failing tests

- **Problem:** Can't distinguish new bugs from pre-existing issues
- **Fix:** Report failures, get explicit permission to proceed

### Hardcoding setup commands

- **Problem:** Breaks on projects using different tools
- **Fix:** Auto-detect from project files (package.json, etc.)

## Example Workflow

```
You: I'm using the using-git-worktrees skill to set up an isolated workspace.

[Check .worktrees/ - exists]
[Verify ignored - git check-ignore confirms .worktrees/ is ignored]
[Create worktree: git worktree add .worktrees/auth -b feature/auth]
[Run npm install]
[Run npm test - 47 passing]

Worktree ready at /Users/jesse/myproject/.worktrees/auth
Tests passing (47 tests, 0 failures)
Ready to implement auth feature
```

## Red Flags

**Never:**
- Use relative paths for file operations (assumes you're "in" the worktree)
- Expect `cd` to persist across tool calls
- Create worktree from within another worktree (detect and navigate to main repo first)
- Use relative paths like `../.worktrees/` for worktree operations (always use absolute paths)
- Skip capturing worktree path in `$WORKTREE_PATH` variable
- Create worktree without verifying it's ignored (project-local)
- Skip baseline test verification
- Proceed with failing tests without asking
- Assume directory location when ambiguous
- Skip CLAUDE.md check

**Always:**
- Run pre-flight checks: detect if in worktree, detect if bare repo
- Use `$WORKTREE_PATH` for all file operations (Read/Edit/Write)
- Chain commands with `&&` when running in worktree: `cd "$WORKTREE_PATH" && command`
- Use absolute paths for all worktree creation
- Capture worktree path in `$WORKTREE_PATH` variable immediately after creation
- Follow directory priority: existing > CLAUDE.md > ask
- Verify directory is ignored for project-local
- Auto-detect and run project setup
- Verify clean test baseline

## Integration

**Called by:**
- **brainstorming** (Phase 4) - OPTIONAL when isolation is needed for implementation
- **subagent-driven-development** - OPTIONAL when isolation is required
- **executing-plans** - OPTIONAL when isolation is required
- Any skill needing isolated workspace under the same gate

**Pairs with:**
- **finishing-a-development-branch** - REQUIRED for cleanup after work complete (if worktree was created)

**Note:** Worktrees are now opt-in. Use when you need branch isolation. Skip for simple changes on main branch with explicit user consent.
