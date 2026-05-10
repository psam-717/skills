---
name: py-publish
description: >
  Publishes Python projects to TestPyPI and PyPI, then creates a GitHub version
  release. Use this skill when the user wants to publish, release, or ship a
  Python package, or asks to bump a version and push to PyPI. Requires a
  pyproject.toml in the project. Workflow: build → TestPyPI → user tests locally
  → PyPI + GitHub release.
---

# Python Package Publisher

You are acting as a Python package release engineer. You handle the full publish
lifecycle: version bump → build → TestPyPI → confirmation gate → PyPI → GitHub
release. Follow every phase in strict order.

---

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
   - Confirm `gh auth status` passes (authenticated to GitHub).
   - Warn (but do not stop) if there are uncommitted changes — tell the user:
     > "⚠ You have uncommitted changes. Consider committing before releasing."

5. **Check PyPI credentials** by verifying that `~/.pypi` or environment variables
   `TWINE_USERNAME` / `TWINE_PASSWORD` (or a `~/.pypirc` entry for both `pypi`
   and `testpypi`) exist. If missing, explain how to set them up:
   ```
   # ~/.pypirc
   [distutils]
   index-servers = pypi testpypi

   [pypi]
   username = __token__
   password = pypi-<your-token>

   [testpypi]
   username = __token__
   password = pypi-<your-testpypi-token>
   ```
   Stop until credentials are in place.

---

## Phase 1 — Version Bump

Determine the correct new version based on what changed in the project.

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
```
git add pyproject.toml
git commit -m "chore: bump version to <new_version>

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

---

## Phase 2 — Build

### 2.1 — Prepare the `dist/` Directory

- If `dist/` does **not** exist: create it (it will be populated by the build).
- If `dist/` **already exists**: delete its contents before building to avoid
  stale artifacts:
  ```
  Remove-Item dist\* -Recurse -Force   # Windows
  # or
  rm -rf dist/*                         # Unix
  ```

### 2.2 — Build the Package

```
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

## Phase 3 — Publish to TestPyPI

```
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

Reply "ship it" (or "ok", "looks good", "publish", "go") when you're happy,
or "abort" to stop without publishing to PyPI.
```

**Stop here. Do not publish to PyPI until the user explicitly confirms.**

---

## Phase 4 — Publish to PyPI

Only proceed after the user signals approval (e.g. "ship it", "ok", "looks good",
"publish", "go").

```
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

## Phase 5 — GitHub Release

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

Omit sections that have no entries. Keep messages concise — one line each.

### 5.2 — Create the Git Tag and GitHub Release

```
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

---

## Phase 6 — Release Summary

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

## Error Handling Rules

| Situation | Action |
|---|---|
| `pyproject.toml` not found | Stop immediately. Explain and do nothing. |
| Build fails | Show full output. Stop. Do not publish anything. |
| `twine check` reports errors | Show them. Ask user whether to continue or fix first. |
| TestPyPI upload fails | Show error. Stop. Do not proceed to PyPI. |
| PyPI upload fails (e.g. version exists) | Show error. Stop. Do not create GitHub release. |
| GitHub CLI not authenticated | Run `gh auth login` and guide the user through it before continuing. |
| Uncommitted changes detected | Warn but do not stop — the user may have intentional working-tree-only changes. |
| No previous git tag found | Use the entire commit history for release notes. Label it "Initial release". |

---

## Gate Summary (where the skill stops and waits)

```
Phase 0  Pre-flight         ── stops if tools or credentials missing
Phase 1  Version bump       ── stops to get user confirmation of new version
Phase 3  TestPyPI published ── HARD STOP — waits for "ship it" before PyPI
```

Never publish to PyPI or create a GitHub release without explicit user approval
after TestPyPI testing.
