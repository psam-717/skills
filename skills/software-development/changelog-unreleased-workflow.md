---
name: changelog-unreleased-workflow
description: >-
  Maintain CHANGELOG.unreleased.md in psamvault-cli. When the user opens a PR
  with code changes, read the PR diff, write entries to the unreleased file,
  and open a separate PR for the changelog update so it merges cleanly alongside
  the code changes.
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [psamvault-cli, changelog, git, github, pr-workflow]
    related_skills: [psamvault-release, granular-commits-pr, github-pr-workflow]
---

# Changelog Unreleased Workflow

Keep `CHANGELOG.unreleased.md` up to date with changes merged to `main` but
not yet published to PyPI. The key rule: **changelog updates go through a
separate PR**, never pushed directly to main.

## When to use

- User opens a PR with code changes → you read it and add changelog entries
- User merges code changes and asks you to update the unreleased file
- User asks "what's pending for the next release?"

## Repository

- `psamvault-cli` at `/root/psamvault-cli`
- Unreleased file: `CHANGELOG.unreleased.md`
- Released changelog: `changelog.py` (versioned dict format)

## The problem this solves

Don't push changelog updates directly to main. The correct flow isolates the
changelog update in its own PR so main always gets code + changelog as clean
merge commits.

**Memory aid:** Never `git push origin main` with only a changelog update.
Always push a separate branch and open a PR.

## Workflow

### Flow A — User opens a PR with code changes (recommended)

Steps:

1. **User opens PR** with code changes and sends you the PR number
2. **Read the PR** — use `gh pr view <N>` and `gh pr diff <N>` to understand
   what changed
3. **Sync local main:**
   ```bash
   git checkout main && git pull --ff-only origin main
   ```
4. **Create a changelog branch** named after the PR:
   ```bash
   git checkout -b feat/changelog-pr-<N>
   ```
5. **Update `CHANGELOG.unreleased.md`** — categorize entries under the
   appropriate section headers (Added / Fixed / Changed / Tests / Docs).
   Follow the existing format: `type(scope): description`
6. **Commit and push the changelog branch:**
   ```bash
   git add CHANGELOG.unreleased.md
   git commit -m "docs: add PR #<N> entries to CHANGELOG.unreleased.md"
   git push origin feat/changelog-pr-<N>
   ```
7. **Open a PR** targeting the user's code branch, not main:
   ```bash
   gh pr create \
     --title "docs: add PR #<N> entries to CHANGELOG.unreleased.md" \
     --body "## Summary
   Changelog entries for PR #<N> (the feature/fix branch).

   Merging this after the code PR goes through ensures both arrive
   on main together with clean commit history." \
     --base <user-branch-name> \
     --head feat/changelog-pr-<N>
   ```
8. **Tell the user:** both PRs are ready. User merges code PR first, then
   changelog PR (or squash the changelog into the same PR if user prefers).

### Flow B — Code already merged, catch up

If the user merged the code PR but the unreleased file wasn't updated:

1. **Sync local main:**
   ```bash
   git checkout main && git pull --ff-only origin main
   ```
2. **Create a changelog branch:**
   ```bash
   git checkout -b feat/changelog-pr-<N>-catchup
   ```
3. **Update `CHANGELOG.unreleased.md`** with the PR's changes
4. **Commit and push:**
   ```bash
   git add CHANGELOG.unreleased.md
   git commit -m "docs: add PR #<N> entries to CHANGELOG.unreleased.md"
   git push origin feat/changelog-pr-<N>-catchup
   ```
5. **Open a PR** targeting main:
   ```bash
   gh pr create \
     --title "docs: add PR #<N> entries to CHANGELOG.unreleased.md" \
     --body "## Summary
   Catch-up changelog entries for already-merged PR #<N>." \
     --base main
   ```
6. **Tell the user** to merge the changelog PR

### Flow C — Unreleased file check (read-only)

When the user asks "what's pending?":

```bash
cd /root/psamvault-cli && cat CHANGELOG.unreleased.md
```

Also optionally check git log to see what's on main since the last tag:
```bash
cd /root/psamvault-cli && \
  LAST_TAG=$(git tag --list 'v*' --sort=-version:refname | head -1) && \
  git log --oneline "$LAST_TAG..HEAD" | head -20
```

## Changelog format

Follow this structure:

```markdown
## Added

- feat: login url detection for improved login flow
- feat(scope): description

## Fixed

- fix(scope): description

## Changed

- refactor(scope): description

## Tests

- test(scope): description

## Docs

- docs: description
```

### Mapping rules

When a commit message says:
- `feat:` → **Added**
- `fix:` → **Fixed** (unless it's clearly a refactor)
- `refactor:` → **Changed**
- `test:` → **Tests**
- `docs:` → **Docs**
- `chore:` → skip (or Added if user-facing)
- `ci:` → skip

Deduplicate: if an entry already exists in the unreleased file, don't add it
again. Read the file first.

## Pitfalls

1. **Never push changelog commits directly to main** — always use a branch + PR
2. **Target the user's code branch** (Flow A), not main, so both PRs merge cleanly
3. **Don't forget to pull first** — stale local = merge conflicts
4. **Use `--ff-only`** when pulling to catch divergence early
5. **Read the full diff** — don't guess what changed from the PR title alone
6. **Deduplicate** — check existing entries in `CHANGELOG.unreleased.md` before adding
