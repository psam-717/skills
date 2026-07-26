---
name: granular-commits-pr
description: Commit each changed file individually with conventional commit messages, create a new feature branch, push, and open a PR. Used when the user wants fine-grained, file-level commits rather than squashing everything into one.
version: 1.0.0
metadata:
  hermes:
    tags: [git, github, conventional-commits, pr-workflow]
---

# Granular File-Level Commits → PR

Commits every modified/new/deleted file with its own conventional commit message,
then creates a fresh feature branch, pushes, and opens a PR.

## When to use

- User asks for individual file commits ("commit each file separately", "I want at least N commit messages")
- User wants a clean, auditable commit history with one file per commit
- User wants a new feature branch created for the PR (not reusing an existing branch)

## Workflow

### Step 1: Audit the working tree

```bash
git status --short
```

Identify:
- `M` — modified files
- `??` — new (untracked) files
- `D` — deleted files

### Step 2: Commit each file individually

For **modified/untracked files**:
```bash
git add <path/to/file>
git commit -m "type(scope): description

Detailed body explaining the change."
```

For **deleted files**:
```bash
git rm <path/to/file>
git commit -m "type: description"
```

**Conventional commit format:**
- `feat:` — new feature or module
- `fix:` — bug fix
- `refactor:` — code change that isn't a fix or feature
- `test:` — test changes
- `docs:` — documentation
- `chore:` — maintenance (deps, version bumps, lockfiles)

**Scope** (optional, in parentheses): the module/area affected, e.g. `refactor(tools):`, `test(conftest):`

**Body:** 1-3 lines explaining WHY the change was made, not just what changed.

### Step 3: Verify commit log

```bash
git log --oneline -N   # N = number of commits just made
```

Confirm all files are committed:
```bash
git status --short   # should be empty
```

### Step 4: Create a fresh feature branch

Never reuse an existing branch. Name format: `feat/<descriptive-kebab-case>`

```bash
git checkout -b feat/v0.4.0-browser-refactor
git push origin feat/v0.4.0-browser-refactor
```

### Step 4.5: Check open PRs before creating (critical)

**Before creating a new PR, always check which PRs are already open on this repo**
to determine the correct `--base` target. Do NOT assume a branch you worked on
before is still open — older PRs may have been merged already.

```bash
gh pr list --json number,headRefName,baseRefName,state,title --limit 10
```

Handle each scenario:

| Scenario | What base to use |
|---|---|
| No open PRs | `--base main` (everything goes to main) |
| Open PRs exist | Target `main` unless your feature explicitly builds on another open PR. If so, explain why to the user first. |
| Your old PR was merged | The old feature branch is already in `main`. Base your new branch on `main` and target `main`. |

### Step 5: Open the PR

Use `gh pr create` with a structured body:

```bash
gh pr create \
  --title "type: short summary of the release" \
  --body "## Summary
One-paragraph overview.

## Changes
### Section 1
- bullet points

### Section 2
- bullet points

### Files
| File | Status |
|------|--------|
| path/to/file.py | Updated |
| old/file.py | Deleted |
| new/file.py | New |

N tests passing" \
  --base main
```

## Common pitfalls

1. **Don't reuse branches** — always create a fresh `feat/*` branch, even if commits are already on another branch. Use `git checkout -b` from the current state.
2. **Close stale PRs** — if a PR was accidentally opened from the wrong branch, close it with `gh pr close <N>` before opening the new one.
3. **Don't push to main** — all release commits go through a feature branch → PR → merge flow.
4. **One file per commit** — resist the urge to group related files. The user wants granularity.
5. **Conventional commit types** — use the right prefix. `fix:` for bugs, `feat:` for new code, `refactor:` for rewrites, `chore:` for boring stuff.
