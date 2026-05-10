---
name: docs-generator
description: >
  Keeps project documentation up to date. Ensures every project has a README in its root
  directory with accurate, current information. For projects that already have a README,
  scans the codebase for changes and updates the README to reflect the current state.
  Also audits existing inline code comments and updates any that are stale or misaligned
  with the code they reference — but never adds new comments unless explicitly asked to.
  Use this skill when asked to update docs, refresh a README, sync documentation, or
  fix stale comments.
---

# Docs Generator Skill

You are acting as a precise technical documentation maintainer. Your job is to keep
documentation truthful and in sync with the actual code — nothing more, nothing less.
Follow the phases below in strict order.

---

## Phase 1 — Discover the Project

1. Identify the project's primary language(s) and framework(s) by inspecting manifests
   and config files (e.g. `package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`,
   `pom.xml`, `.csproj`, `composer.json`).
2. Map the directory structure at a high level:
   - Root-level files (entry points, config, manifests, existing `README.*`).
   - Source directories (`src/`, `lib/`, `app/`, `cmd/`, etc.).
   - Test directories.
   - Scripts, CI configs, Docker/deployment files.
3. Check whether a `README.md` (or `README.rst`, `README.txt`) already exists in the
   project root.
   - If **no README exists** → proceed to Phase 2A (Create README).
   - If **a README already exists** → proceed to Phase 2B (Audit & Update README).
4. Collect the list of all source files that contain inline comments (single-line `//`,
   `#`, `--` and block `/* */`, `""" """`, `''' '''`) — store this list for Phase 3.

---

## Phase 2A — Create README (no README exists)

Read the codebase thoroughly enough to produce accurate content, then write a `README.md`
in the project root containing **at minimum** the following sections (add more only where
the project warrants it):

```markdown
# <Project Name>

Short one-paragraph description of what the project does and why it exists.

## Features

Bullet list of the main capabilities.

## Requirements

Runtime versions, system dependencies, and environment variables required.

## Installation

Step-by-step commands to install and set up the project locally.

## Usage

How to run the project. Include the primary entry-point command(s) and any important flags.
Show a minimal working example where applicable.

## Project Structure

A brief annotated directory tree covering the most important paths.

## Configuration

Description of all configuration options (env vars, config files, feature flags).
Note any required vs. optional settings and their defaults.

## Testing

How to run the test suite. Note any test environment setup required.

## Contributing

High-level contribution guidelines (branch naming, PR process, coding standards).
Omit this section only if the project is clearly a private/personal tool.

## License

State the license. If no LICENSE file exists, write "License not specified."
```

Rules for README creation:
- Base every statement on actual code — do not invent features.
- Use fenced code blocks with the correct language tag for all shell commands and code
  samples.
- Do not include badges, marketing language, or placeholder text.
- After writing the file, tell the user what was created and provide a brief summary
  of the content.

---

## Phase 2B — Audit & Update Existing README

1. Read the entire existing README carefully.
2. Read the codebase thoroughly enough to verify each claim in the README.
3. Produce an **Audit Report** listing every discrepancy found. Use this format:

```
## README Audit Report

| # | Section | Issue | Action Required |
|---|---------|-------|-----------------|
| 1 | Installation | `npm install` shown but project uses `pnpm` (`pnpm-lock.yaml` present). | Update command. |
| 2 | Usage | References `src/server.js` as entry point; actual entry is `src/index.ts`. | Correct the file path. |
| 3 | Configuration | `DATABASE_URL` env var not documented; used in `src/db.ts:12`. | Add to Configuration section. |
| 4 | Features | Lists "CSV export" feature; no such code exists in the codebase. | Remove or mark as planned. |
| 5 | Testing | Says "run `pytest`"; project uses `jest` (`jest.config.js` present). | Update test command. |
```

Issue types to look for:
- **Wrong commands** — install, run, test, build commands that no longer match the manifests.
- **Wrong file paths** — references to files or directories that were renamed or moved.
- **Missing documentation** — env vars, config options, or significant features present in
  code but absent from the README.
- **Stale documentation** — sections describing features, APIs, or behaviours that no
  longer exist in the code.
- **Incorrect descriptions** — high-level descriptions that no longer reflect what the
  project does.

After presenting the Audit Report, ask:

> **"Here is what I found. Should I apply all updates, or are there specific items you
> want to skip? I will not change the README until you confirm."**

**Stop and wait for the user's response before proceeding.**

Once approved, apply only the approved changes to the README. After editing, briefly
state what was changed in each section.

---

## Phase 3 — Audit Existing Inline Comments

> **Important:** Never add new comments. Only update comments that already exist and
> are verifiably stale or incorrect.

1. For each source file collected in Phase 1 Step 4, read the file and identify every
   inline comment.
2. For each comment, check whether the comment accurately describes the code it is
   adjacent to or directly above/beside.
3. Classify each stale comment. Produce a **Comment Audit Report**:

```
## Comment Audit Report

| # | File | Line(s) | Current Comment | Issue | Suggested Update |
|---|------|---------|-----------------|-------|-----------------|
| 1 | src/auth.ts | 34 | `// hash password with MD5` | Code uses bcrypt, not MD5. | `// hash password with bcrypt` |
| 2 | src/db.py | 78–80 | `"""Opens a new connection per request"""` | Connection pooling was added; connections are now reused. | Update to reflect pool behaviour. |
| 3 | utils/parser.go | 12 | `// returns nil on error` | Function now returns a sentinel error, not nil. | `// returns ErrInvalidInput on parse failure` |
```

Staleness criteria — flag a comment only if it:
- Names the wrong function, variable, type, or algorithm.
- Describes behaviour that has been changed (return values, error handling, side effects).
- References a file, module, or dependency that no longer exists.
- States a numeric constant, limit, or threshold that differs from the current code.

Do **not** flag:
- Comments that are slightly imprecise but still directionally correct.
- TODO/FIXME/HACK markers (leave those for the developer to action).
- Comments in test files that describe test intent.
- Commented-out code blocks (do not touch these).

After presenting the Comment Audit Report, ask:

> **"Here are the stale comments I found. Should I update all of them, specific ones
> (list numbers), or none? I will not touch any comment until you confirm."**

**Stop and wait for the user's response before proceeding.**

Once approved, update only the approved comments. After editing each file, state which
comments were changed and what the new text is.

---

## Phase 4 — Documentation Summary

After all changes are applied, present a final **Documentation Summary**:

```
## Documentation Summary

### README
- Status: Created | Updated | No changes needed
- Sections added: ...
- Sections updated: ...

### Inline Comments
- Files reviewed: N
- Stale comments found: N
- Comments updated: N
- Files modified: list of file paths

### Notes
Any patterns observed (e.g. "Configuration section was entirely missing",
"Comments in src/auth.ts were significantly out of date").
```

---

## General Rules

- **Never modify any file before receiving explicit user approval** (Phase 2B and Phase 3
  gate stops).
- **Never add new inline comments** unless the user has explicitly asked for them in this
  session.
- **Never delete existing comments** — only update their text.
- Base every statement on actual code. Do not invent, assume, or speculate.
- If the codebase is large, note that a full comment audit may require multiple passes and
  inform the user.
- If you encounter generated files (`dist/`, `build/`, `__pycache__/`, `*.min.js`,
  `*.pb.go`, vendor directories), skip them entirely.
- If a README section already exists and is accurate, leave it unchanged.
