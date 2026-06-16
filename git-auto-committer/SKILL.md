---
name: git-auto-committer
description: >-
  Scans the current git repo for all untracked and modified files, examines
  their contents/diffs, generates meaningful commit messages, presents them
  to the user for verification, then stages and commits each file individually.
  After committing, optionally creates a new branch and opens a PR for major
  updates (seeks user approval first).
  Supports both interactive mode (user-confirmation required) and cron/
  automated mode (no interaction, commits autonomously).
  Trigger phrases: "stage and commit", "commit my changes", "commit all files",
  "git commit", "commit my modified files", "create a commit message",
  "auto-commit cron job".

  Works on Windows (git-bash/PowerShell) and POSIX.
---

# Git Auto-Committer

You are an expert git workflow specialist. When the user asks to stage and commit their changes, follow this process strictly.

This skill has two modes:
- **Interactive mode** (default) — shows changes, waits for user confirmation, then commits
- **Cron / Automated mode** — runs autonomously, commits without user interaction, designed for `cronjob(action='create')`

Check the prompt or context to determine the mode. If the task involves `cronjob`, `cron`, `schedule`, or `automated`, use Cron mode. Otherwise, use Interactive mode.

---

## Phase 1 — Identify All Uncommitted Files

Run from the project root:

```bash
git status --short
```

Parse the output:
- `?? <file>` → **untracked** (new files)
- ` M <file>` → **modified** (staged changes already? Check if ` M` vs `M ` — first column is staging area)
- `M  <file>` → **staged** (already in staging area — skip these)
- `MM <file>` → **modified and staged** (partially staged — treat as modified)

Collect the full list of files that need processing.

---

## Phase 2 — Examine Each File

For each untracked or modified file, analyze its content:

**For modified files:**
```bash
git diff <filepath>
```
Read the diff to understand what changed. Look for patterns: bug fixes, new features, refactoring, config changes, documentation updates.

**For untracked files:**
Read the file contents directly to understand its purpose:
```bash
cat <filepath>       # or use read_file tool
```
Look at filename, directory context, and content to classify the change.

**Skip generated directories** — `node_modules/`, `__pycache__/`, `.git/`, `build/`, `dist/`, `.dart_tool/`, `.packages/`, `.flutter-plugins*`, `*.g.dart`, `*.freezed.dart`, `pubspec.lock`, `venv/`, `cli_venv/`, `mcp_venv/`, `.pytest_cache/`, `.egg-info/`, `*.pyc`, `*.min.js`.

**Honour user-specified exclusions** — If the user explicitly says "exclude these folders" or "skip files in <path>", parse that instruction and exclude the named paths from both the file listing and any commit candidates. Do not second-guess — exclude exactly what they said.

---

## Phase 3 — Build a Verification Table (INTERACTIVE MODE ONLY)

Present the user with a clear table showing every file found and your proposed commit message:

```
🔍 Found N uncommitted file(s):

  # │ TYPE      │ FILE                  │ PROPOSED COMMIT MESSAGE
  ───┼──────────┼───────────────────────┼──────────────────────────────────
  1  │ Modified  │ lib/auth.dart        │ fix: validate token expiry in login handler
  2  │ Untracked │ test/auth_test.dart  │ test: add edge-case tests for token refresh
  3  │ Modified  │ README.md            │ docs: update installation instructions
```

Ask using the `clarify` tool with choices:

> **"Here are the changes I found. Shall I proceed with committing all of them, or would you like to skip any?"**

Provide choices like `["All, proceed with all commits", "Skip file 1, 3", "Cancel"]`. If the user responds with numbers (e.g. "skip 1, 3"), those files are skipped. If "all" or affirmative, all files are committed.

**STOP and wait for user confirmation.** Do NOT stage or commit anything until the user explicitly confirms.

---

## Phase 4 — Stage and Commit One by One

Once the user confirms (interactive) or directly after Phase 2 (cron mode), process each file individually:

1. **Stage:**
   ```bash
   git add <filepath>
   ```

2. **Verify staging:**
   ```bash
   git status
   ```
   Confirm only the intended file is staged.

3. **Commit:**
   ```bash
   git commit -m "type(scope): description"
   ```

4. **Verify:**
   Check that the commit completed successfully before moving to the next file.

Each file gets its own individual commit with a unique, meaningful message.

---

## Cron / Automated Mode

Use this mode when the task is packaged as a `cronjob(action='create')` or any autonomous schedule.

**Key differences from Interactive mode:**
- **Skip Phase 3 entirely** — no verification table, no waiting for approval
- **Generate commit messages** from Phases 1–2 analysis alone; use the Commit Message Guidelines below
- **Commit immediately** after Phase 2 for each file, one by one
- **Deliver a summary** at the end — use the final response as the Telegram deliverable:

```
🔔 Auto-Commiter Report — <repo-name>

Committed <N> file(s):
  • <file1> → feat(scope): description
  • <file2> → fix(scope): description
  • <file3> → docs(scope): description

Summary:
  git log --oneline -n <N>
```

**Setting up the cron job:**
- The cron job's `workdir` MUST be set to the absolute path of the git repo root (e.g., `D:\Projects\servia-updated\servipac_flutter`)
- The cron job's `workdir` value ensures AGENTS.md / CLAUDE.md / .cursorrules from the project are injected as context
- The `prompt` field should include the trigger phrase like "auto-commit my changes" or "run git-auto-committer"
- The `skills` field must include `git-auto-committer`
- Example cron job setup:
  ```
  cronjob(
    action='create',
    name='Daily Auto-Commit - Project Name',
    schedule='0 21 * * *',         # 9 PM daily
    prompt='Auto-commit my uncommitted changes in this repo',
    skills=['git-auto-committer'],
    workdir='D:\\Projects\\<repo-name>',
    deliver='origin'
  )
  ```

**When no changes exist:**
If `git status --short` returns empty, respond with:
```
🔔 Auto-Commiter Report — <repo-name>

No uncommitted changes found.
```

---

## Phase 5 — Offer PR Creation (INTERACTIVE MODE ONLY)

**After all commits are done**, assess if the changes qualify as a **major update**.
A change is "major" when it involves:
- New features (`feat` type commits)
- Significant refactoring or restructuring
- API changes or breaking changes
- Cross-cutting changes spanning multiple modules
- A feature that would benefit from code review before merging

**If the changes are minor** (simple docs fixes, typo corrections, single-file chores, style-only changes), skip this phase entirely — just deliver the commit summary and finish.

**If the changes are major**, transition to PR workflow by asking the user:

> **"These changes look significant. Would you like me to create a new branch and open a GitHub PR for review?"**

Use the `clarify` tool to ask for approval. **STOP and wait for user confirmation.** Do NOT create any branch or PR until the user explicitly says yes.

### PR Creation Steps

When the user confirms, follow the `github-pr-workflow` skill (load it with `skill_view`):

1. **Create a new branch** from the current state:
   ```bash
   git checkout -b feat/<short-description>
   ```

2. **Push the branch** to GitHub:
   ```bash
   git push -u origin HEAD
   ```

3. **Create the PR** using `gh pr create` with:
   - A conventional-commit style title
   - A body summarizing what changed and why
   - Base branch set to the repo's default (usually `main`)

4. **Show the PR URL** to the user:
   ```
   ✅ PR created: https://github.com/<owner>/<repo>/pull/<number>
   ```

If `gh` is not available, fall back to `curl` + `GITHUB_TOKEN` (see `github-pr-workflow` skill).

---

## Important Rules

- **Interactive mode:** NEVER commit before getting user confirmation.
- **Cron mode:** Confirm user explicitly requested an autonomous schedule before skipping Phase 3. If ambiguous, default to interactive.
- Each file gets its own separate commit — never batch multiple files in one commit.
- Skip already-staged files — only process untracked and modified-but-not-staged files.
- If a file's changes are ambiguous, do your best to classify it. In cron mode, use sensible defaults (e.g., `chore` for unclear changes).
- After all commits, show a summary:
  ```bash
  git log --oneline -n <count>
  ```

### Important

When the user says "commit and create a PR", do NOT commit to `main` first and then create a branch. The correct sequence is:

1. **Commit on the current branch** (usually `main` for convenience)
2. **Create a new feature branch** from the current state
3. **Reset `main` back** to the last clean merge commit
4. **Force-push `main`**
5. **Push the feature branch**
6. **Create PR** from feature branch → main

This ensures the PR is the source of truth and `main` stays clean. See `psamvault-release` skill's reference `main-push-rescue-example.md` for a concrete rescue example.

---

## Commit Message Guidelines

Use conventional commit format: `type(scope): description`

| Type | When to use |
|------|-------------|
| `feat` | New feature or addition |
| `fix` | Bug fix |
| `refactor` | Code reorganization, no behaviour change |
| `docs` | Documentation only |
| `test` | Adding or updating tests |
| `chore` | Maintenance, config, dependencies, tooling |
| `style` | Formatting, linting, whitespace |

**Scope:** the filename or module area affected (e.g., `auth`, `api_client`, `README`).

**Description:** concise, present tense, lowercase, under 72 characters.

**Bad:** `"update files"` or `"make changes"`
**Good:** `"refactor: extract password validation into helper function"`

For files that span multiple concerns, use the primary/dominant change type.

---

### Windows-specific pitfalls

- **SRE module mismatch bug**: On this Windows machine, the uv-managed Python may crash with `"AssertionError: SRE module mismatch"` when importing `re`. This can affect scripts that parse `git diff` output or run Python-based tools.

  **Workaround:** Set `PYTHONHOME=""` before running Python commands, or create a fresh uv venv and use its Python:
  ```bash
  uv venv .venv
  .venv/Scripts/python.exe <script>
  ```

- **Patch tool indentation warping**: The `patch` tool can mangle indentation on Python files when `old_string`/`new_string` contain indented blocks (function bodies, nested conditionals). This happens because the fuzzy matcher re-aligns indent levels during replacement. Symptoms: `IndentationError` after the edit.

  **Workaround:** When editing large blocks of indented Python code (whole functions, multi-line conditionals):
  1. If possible, keep replacements small (one or two lines) to avoid triggering the fuzzy indent re-alignment
  2. If the patch tool returns an indentation error, fall back to reading the full file with `read_file`, then using `write_file` to rewrite the entire file with the correct indentation
