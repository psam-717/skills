---
name: py-publish
description: >-
  Publishes Python projects to TestPyPI and PyPI, then creates a GitHub version
  release. Use this skill when the user wants to publish, release, or ship a
  Python package, or asks to bump a version and push to PyPI. Requires a
  pyproject.toml in the project. Workflow: build → TestPyPI → user tests locally
  → PyPI + GitHub release. Before the version bump, loads the
  granular-commits-pr skill to commit codebase changes individually.
---

# Python Package Publisher

You are acting as a Python package release engineer. You handle the full publish
lifecycle: version bump → build → TestPyPI → confirmation gate → PyPI → GitHub
release. Follow every phase in strict order.

---

## Windows-Specific Release Details

For Windows-specific release pitfalls (SRE module mismatch breaking builds,
uv tool install as pipx replacement, taskkill in MSYS/git-bash, editable
install for dev, and more), see `references/release-windows-pitfalls.md`.

For publishing with psamvault-stored PyPI tokens (zero-knowledge approach —
no `.pypirc` file on disk), see `references/publishing-with-psamvault-credentials.md`.

For a project-specific release workflow example (psamvault CLI), including
the pre-release checklist, security audit flow, dashboard verification, and
GitHub main-push rescue, see the `references/psamvault-*` files in this
skill's references directory.

## Phase 0 — Pre-flight Checks

Before doing anything else, verify the environment is ready:

1. **Locate `pyproject.toml`** in the current working directory or its immediate
   parent. If not found, stop and tell the user:
   > "No `pyproject.toml` found. This skill only supports projects using
   > `pyproject.toml`. Please run from the project root."

2. **Read `pyproject.toml`** and extract:
   - `[project].name` — the package name
   - `[project].version` — the current version string
   - `[build-system]` — the build backend (hatchling, setuptools, flit, etc.)

3. **Check required tools** are available on PATH:
   - `python` or `python3`
   - `pip` / `build` (`python -m build --version`)
   - `twine` (`twine --version`)
   - `gh` (GitHub CLI — `gh --version`)

   If any tool is missing, tell the user exactly which ones to install:
   ```
   pip install build twine
   gh: https://cli.github.com/
   ```
   Stop until all tools are confirmed present.

4. **Check git state:**
   - Confirm the working directory is a git repository (`git rev-parse --git-dir`).
     If not, determine whether the repo needs creating (first release on a new
     package) vs. a standard release. **Do NOT skip this check** — the release
     flow differs for each case.
   - Confirm `gh auth status` passes (authenticated to GitHub).
   - Warn (but do not stop) if there are uncommitted changes — tell the user:
     > "⚠ You have uncommitted changes. Consider committing before releasing."

5. **Check PyPI credentials.** Verify token availability based on strategy:
   - **Classic `.pypirc` strategy:** Check that `~/.pypi` or environment variables
     `TWINE_USERNAME`/`TWINE_PASSWORD` (or a `~/.pypirc` entry for both `pypi` and
     `testpypi`) exist.
   - **psamvault vault strategy:** Check that `psamvault login` has been run
     (keychain has `session.vek` + `session.access_token`) and that API key entries
     named `pypi` and `testpypi` exist.
   
   If credentials are missing, ask which strategy the user prefers. If they choose
   psamvault vault, document the setup steps from
   `references/publishing-with-psamvault-credentials.md`.

6. **Determine release type:**
   Run `git rev-parse --git-dir 2>/dev/null && git log --oneline 2>/dev/null | wc -l` to
   see how many commits exist. Then branch:

   - **Has .git + commits + remote with tags**: standard subsequent release (Phases 1-6).
   - **Has .git + commits + remote, no tags**: first PyPI release on an existing repo.
     Phases 1-5 proceed normally. In Phase 5.2, the release notes should omit the
     `compare/prev_tag...vX.Y.Z` URL since there's no previous tag.
   - **Has .git but no remote**: repo not pushed yet. Run Phase 0-B, then Phases 1-6.
   - **No .git at all**: brand new package. Run Phase 0-A, then Phases 1-6.

### Phase 0-A — First Release: Brand New Package (no .git)

Use when the package directory has no `.git` and you need to create the GitHub
repository from scratch.

```bash
# 1. Create .gitignore
cat > .gitignore << 'EOF'
__pycache__/
*.pyc
*.egg-info/
dist/
build/
.pytest_cache/
.env
EOF

# 2. Git init + first commit
git init
git add .
git commit -m "initial commit: <package-name> v<version>"

# 3. Ask the user: public or private repo?
#    Then create:
gh repo create <org>/<repo-name> --public --description "<description>"

# 4. Push
git remote add origin https://github.com/<org>/<repo-name>.git
git push -u origin master
```

**Decision required:** Ask the user whether the repo should be `--public` or `--private`.

After push, verify: `gh repo view <org>/<repo-name>`. Then proceed to Phase 1.
Since this is the first release, there's no previous version to compare against —
use semver starting at `0.1.0` for initial release.

### Phase 0-B — First Release: Has .git, No Remote

Use when the directory is already a git repository but has no remote configured.

```bash
# 1. Create GitHub repo
gh repo create <org>/<repo-name> --public --description "<description>"

# 2. Push
git remote add origin https://github.com/<org>/<repo-name>.git
git push -u origin $(git branch --show-current)
```

Then proceed to Phase 1.

### Phase 0-C — First Release: Has Remote, No Tags

The repo exists on GitHub but hasn't been published to PyPI yet. Phases 1-5
proceed normally — just note that when Phase 5.2 creates the first tag, the
release notes won't have a `compare/<prev_tag>...v<new_version>` URL. Omit that
line from the release notes when it's the first tag.

## Phase 1 — Pre-Release Prep: README + Changelog

Before bumping the version, update the `README.md` (and any top-level doc files
the project uses) to reflect the current state of the package:

1. **Review what's changed since last release:**
   ```bash
   git log <last-tag>..HEAD --oneline --stat
   ```
   If there are no tags yet, use `git log --oneline -20`.

2. **Update `README.md`** — check these sections are current:
   - Feature list / table of tools
   - Version badge or tagline
   - Any new flags, parameters, or commands added
   - Any deprecation notices
   - Installation instructions (were dependencies added?)

3. **If the project has a `CHANGELOG.md`**, add an entry for the new version.
   Format:
   ```markdown
   ## [vX.Y.Z] — YYYY-MM-DD
   ### Added
   - New feature A
   ### Fixed
   - Bug fix B
   ### Changed
   - Breaking or behavioural change C
   ```

4. **Stage and commit** the doc updates **before** the version bump:
   ```bash
   git add README.md CHANGELOG.md  # or whatever files were updated
   git commit -m "docs: update README and changelog for vX.Y.Z"
   ```

5. **Commit all other codebase changes, one file per commit** — load the
   `granular-commits-pr` skill with `skill_view(name="granular-commits-pr")`
   and follow its workflow to commit each modified/new/deleted file individually
   with conventional commit messages. Do NOT batch all changes into a single
   "release: prepare" commit — granular commits preserve history and make the
   changelog clear.

6. **Verify** `git status --short` is empty before moving to Phase 2.

> **⚠ Important:** Committing code changes in Phase 1 (before the bump) keeps
> the git history clean: the version bump is the last commit before the tag,
> making it easy to see exactly what version each commit belongs to.

> **⚠ Important:** Updating README after every change is critical for PyPI
> because PyPI renders README.md on the project page. A stale README confuses
> users. Do NOT skip this phase — if the user pushes back, explain why it matters.

---

## Phase 2 — Version Bump

> **⚠ IMPORTANT: User must approve every version bump.** Do NOT change the
> version in pyproject.toml without first proposing the bump and getting an
> explicit go-ahead. Violating this rule has been explicitly called out by
> the user as something they decide, not the agent.

### 1.1 — Analyse Changes

Run `git log <last-tag>..HEAD --oneline` (or `git log --oneline -20` if no tags
exist yet) to read recent commit messages. Classify changes using semver rules:

| Change type | Version segment to bump |
|---|---|
| Breaking change / incompatible API change | **MAJOR** (`X+1.0.0`) |
| New feature, backward-compatible | **MINOR** (`x.Y+1.0`) |
| Bug fix, patch, docs, chore, refactor | **PATCH** (`x.y.Z+1`) |

### 1.2 — Propose the Bump

Present your analysis to the user:

```
## 📦 Version Bump Proposal

Current version : 1.2.3
Suggested bump  : PATCH → 1.2.4

Reason: recent commits are bug fixes / chores (list them briefly).

Accept this bump, or choose:
  [1] MAJOR → 2.0.0
  [2] MINOR → 1.3.0
  [3] PATCH → 1.2.4  ← recommended
  [4] Manual — I'll type the version
```

**Stop and wait for the user's confirmation or choice.**

### 1.3 — Apply the Bump

Once confirmed, update `pyproject.toml` in place — change only the `version`
field under `[project]`. Do not touch any other field.

Then commit the version bump:
```bash
git add pyproject.toml
git commit -m "chore: bump version to <new_version>
```

---

## Phase 3 — Build

### 2.1 — Prepare the `dist/` Directory

- If `dist/` does **not** exist: create it (it will be populated by the build).
- If `dist/` **already exists**: delete its contents before building to avoid
  stale artifacts:
  ```bash
  rm -rf dist/*  # POSIX
  ```

### 2.2 — Build the Package

```bash
python -m build
```

This produces both a `.tar.gz` (sdist) and a `.whl` (wheel) in `dist/`.

Verify both files exist after the build. If the build fails, show the full error
output and stop. Do not proceed to publishing.

### 2.3 — Inspect the Build

Run `twine check dist/*` to validate the distribution metadata. If twine reports
errors or warnings, show them and ask the user whether to proceed or fix them
first.

---

## Phase 4 — Publish to TestPyPI

### 3.0 — Credential strategy

Choose one:

| Strategy | How it works | Best for |
|----------|-------------|----------|
| **`.pypirc` / env vars** (classic) | Token in `~/.pypirc` or `TWINE_USERNAME`/`TWINE_PASSWORD` | CI runners, shared machines |
| **psamvault vault** (zero-knowledge) | Token stored as `testpypi` API key, fetched + decrypted at publish time | Dev machines, agent-driven publishing |

If using psamvault-credential strategy, see `references/publishing-with-psamvault-credentials.md`
for the publish script, one-shot pattern, and TestPyPI `--extra-index-url` quirk.

### 3.1 — Classic `.pypirc` upload

```bash
twine upload --repository testpypi dist/*
```

After a successful upload, show the user:

```
## ✅ Published to TestPyPI

Package : <name>
Version : <new_version>
URL     : https://test.pypi.org/project/<name>/<new_version>/

Test it locally with:
  pip install --index-url https://test.pypi.org/simple/ --extra-index-url https://pypi.org/simple/ <name>==<new_version>

> **Pitfall: TestPyPI has no dependency index.** TestPyPI does not mirror PyPI's
> dependency packages. When installing from TestPyPI for smoke-testing, always
> use `--extra-index-url https://pypi.org/simple/` otherwise pip will fail with
> "Could not find a version that satisfies the requirement" for all transitive
> dependencies.

Reply "ship it" (or "ok", "looks good", "publish", "go") when you're happy,
or "abort" to stop without publishing to PyPI.
```

**Stop here. Do not publish to PyPI until the user explicitly confirms.**

---

## Phase 5 — Publish to PyPI

Only proceed after the user signals approval (e.g. "ship it", "ok", "looks good",
"publish", "go").

### Credential strategy

Same two options as Phase 3. If using psamvault vault, the script in
`references/publishing-with-psamvault-credentials.md` accepts a CLI arg:

```bash
python scripts/publish.py pypi   # for real PyPI
```

### Classic `.pypirc` upload

```bash
twine upload dist/*
```

After a successful upload, confirm:

```
## ✅ Published to PyPI

Package : <name>
Version : <new_version>
URL     : https://pypi.org/project/<name>/<new_version>/
```

If the upload fails (e.g. version already exists), show the error clearly and
stop. Do not attempt a GitHub release if PyPI upload failed.

---

## Phase 6 — GitHub Release

### 5.1 — Build the Release Notes

Generate release notes from `git log <previous_tag>..HEAD --oneline` (or all
commits if this is the first release). Group them:

```markdown
## What's Changed

### 🚀 Features
- <commit message>

### 🐛 Bug Fixes
- <commit message>

### 🔧 Chores / Maintenance
- <commit message>

**Full Changelog**: https://github.com/<owner>/<repo>/compare/<prev_tag>...v<new_version>
```

**First release:** Omit the "Full Changelog" line — there's no previous tag to
compare against.

Omit sections that have no entries. Keep messages concise — one line each.

### 5.2 — Create the Git Tag and GitHub Release

```bash
git tag v<new_version>
git push origin v<new_version>

gh release create v<new_version> dist/* \
  --title "v<new_version>" \
  --notes "<release notes>"
```

This attaches the built `.whl` and `.tar.gz` as release assets automatically.

After success, show:

```
## 🎉 GitHub Release Created

Tag     : v<new_version>
Release : https://github.com/<owner>/<repo>/releases/tag/v<new_version>
Assets  : <name>-<new_version>.tar.gz, <name>-<new_version>-py3-none-any.whl
```

> **Pitfall: "Latest" marker fix for multi-release feature branches.** When two
> releases point to the same branch (e.g. v0.5.2 + v0.5.3 both targeting
> `feat/search-command`), GitHub auto-marks the **most recently created** release
> as "Latest" — not the highest version. If you created v0.5.3 first then
> v0.5.2, v0.5.2 gets the "Latest" badge. Fix:
> ```bash
> gh release edit v<highest-version> --latest
> ```

---

## Phase 6.5 — Post-Release: Commit, Tag, Push

After the PyPI publish is confirmed and the GitHub release is created, ensure
all release artifacts are committed and pushed to the remote:

1. **Commit `pyproject.toml`** (the version bump from Phase 2 — the only
   file modified after Phase 1):
   ```bash
   git add pyproject.toml
   git commit -m "chore: bump version to <new_version>"
   ```

2. **Push the commits** to the remote:
   ```bash
   git push origin <current-branch>
   ```

3. **Create and push the tag** (if not already done in Phase 6):
   ```bash
   git tag v<new_version>
   git push origin v<new_version>
   ```

5. **Verify** everything is pushed:
   ```bash
   git log --oneline origin/<current-branch>..HEAD
   # Should return nothing (branch is up to date)
   git ls-remote origin refs/tags/v<new_version>
   # Should show the tag exists on remote
   ```

6. **Create a new feature branch and open a PR to `main`** to bring the
   release changes into the main branch:
   ```bash
   # Create a new branch from current state
   git checkout -b release/v<new_version>
   git push -u origin release/v<new_version>

   gh pr create \
     --base main \
     --head release/v<new_version> \
     --title "release: v<new_version>" \
     --body "## Release v<new_version>

   Published to PyPI: https://pypi.org/project/<package-name>/<new_version>/

   See GitHub release for changelog."
   ```

   This ensures the release commits always flow through a PR for review,
   even if you were already on `main`. The version bump + tag live on the
   release branch, and `main` stays clean until the PR is merged.

> **⚠ Always push commits and tags together.** A tag without its corresponding
> commit on the remote creates a broken release — users who `git clone` or
> `pip install` will get the code but the tag won't point to it. Push commits
> first, then the tag, or use `git push --atomic origin <branch> <tag>` to
> ensure both go together or neither does.

---

## Phase 7 — Release Summary

Present a final summary of everything that was done:

```
## 📋 Release Summary

Package  : <name>
Version  : <old_version> → <new_version>

✅ pyproject.toml bumped and committed
✅ Built: dist/<name>-<new_version>.tar.gz
          dist/<name>-<new_version>-py3-none-any.whl
✅ Published to TestPyPI
✅ Published to PyPI      → https://pypi.org/project/<name>/<new_version>/
✅ GitHub release created → https://github.com/<owner>/<repo>/releases/tag/v<new_version>
```

---

## Appendix: Pre-release Package Management

Use this section when the user asks to yank, hide, unpublish, or remove a
pre-release package from PyPI/TestPyPI while keeping the main package visible.

### What is yanking?

**Yanking** marks a release as "do not install by default" — it disappears from
search results and `pip install <package>` won't pick it, but explicit
`pip install <package>==<version>` still works and existing installations are
unaffected. This is the **recommended** approach.

**Deleting** a release is permanent and irreversible. The filename becomes
blocked forever — you can never upload the same filename again. Avoid this.

### When to yank

- You released a **pre-alpha / MVP** of a companion package (e.g. MCP server)
  but aren't ready to advertise it broadly
- You released code you now consider incomplete or unstable
- You want the package to stay available for existing users but hidden from
  new installations

### How to yank via web UI

1. Go to `https://pypi.org/project/<package-name>/` → click **Releases**
2. Click the version → click **Yank**
3. Add a reason like: "Pre-release — full release coming soon"

### How to yank via CLI

```bash
# Requires GH_AUTH_TOKEN with PyPI scope or PyPI API token
# Currently no standard CLI for this — use the web UI

# After yanking, verify:
pip install <package-name>==<version>          # still works (explicit pin)
pip install <package-name>                     # skips it (not latest anymore)
```

### Reversing a yank

When the package is ready for public use, un-yank the latest version:

1. Go to the release page on PyPI
2. Click **Un-yank**
3. Publish a new, stable version to supersede the pre-release

### Yanking vs. Deleting summary

| Action | Searchable | pip install | pip install==version | Reversible | Filename blocked |
|--------|-----------|-------------|---------------------|------------|------------------|
| **Yank** | Hidden | Skipped | Works | ✅ Un-yank any time | ❌ No |
| **Delete** | Removed | Fails | Fails | ❌ Permanent | ✅ Yes (forever) |

> **Recommendation:** Always yank, never delete. Releases are cheap — deleting
> creates permanent problems for no gain.
