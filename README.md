# Custom AI Coding Skills

A collection of reusable AI agent skills that level up your coding workflow — from test-driven development to automated releases.

[![skills.sh](https://skills.sh/b/psam-717/skills)](https://skills.sh/psam-717/skills)

## Install

This repo is published on the [skills.sh](https://skills.sh) registry. Install any skill using the `npx skills` CLI.

### Install a single skill

```bash
npx skills add psam-717/skills --skill <skill-name>
```

For example, to install just the **master-reviewer**:

```bash
npx skills add psam-717/skills --skill master-reviewer
```

### Install multiple specific skills

```bash
npx skills add psam-717/skills --skill master-reviewer --skill tdd-enforcer
```

### Install all skills

```bash
npx skills add psam-717/skills --skill '*'
```

### Install all skills to a specific agent

```bash
npx skills add psam-717/skills --skill '*' -a claude-code
```

Supported agents include Claude Code, Cursor, Codex, GitHub Copilot, Gemini CLI, and 70+ more. Use `npx skills add psam-717/skills --list` to discover available skills interactively.

### Install globally (available across all projects)

```bash
npx skills add psam-717/skills --global --skill master-reviewer
```

### Non-interactive (CI/CD friendly)

```bash
npx skills add psam-717/skills --skill master-reviewer -g -a claude-code -y
```

### List all available skills without installing

```bash
npx skills add psam-717/skills --list
```

---

## Skills

### 🔍 Code Quality

| Skill | What it does |
|---|---|
| **code-breakdown** | Thorough, beginner-friendly explanation of an entire codebase. Reads every file and walks through structure, data flow, and key code. |
| **code-review-debugger** | Full codebase review and debugging session. Finds bugs, security issues, logic errors, and quality problems — presents a report for approval before touching anything. |
| **security-auditor** | Proactive vulnerability audit. Scans for injection, auth flaws, secrets exposure, dependency risks, and data validation gaps. Prioritized report → approved surgical fixes. |
| **web-vulnerability-analysis** | Paste a website URL → full security analysis. Detects the stack, discovers API endpoints, tests for missing auth & CORS flaws, extracts exposed PII/financial data, and produces a severity-rated vulnerability report (read-only reconnaissance). |
| **master-reviewer** | Three-round master review loop on an open GitHub PR: review → wait for “tackled” → verify → deeper pass. Uses the agent’s built-in review skill when available; ends with merge go-ahead or blockers. |

### 🧪 Testing

| Skill | What it does |
|---|---|
| **tdd-enforcer** | Strict test-driven development cycle: writes failing tests first → waits for confirmation → writes minimal implementation → refactors. Adapts to any language/framework. |

### 🗄️ Data

| Skill | What it does |
|---|---|
| **db-schema** | Designs, extends, and reviews database schemas for Python backends. Handles tables, columns, relationships, indexes, and migrations — adapts to your stack. |

### 📦 Releases & Docs

| Skill | What it does |
|---|---|
| **py-publish** | Full Python package release pipeline: version bump → build → TestPyPI → confirmation gate → PyPI → GitHub release. |
| **docs-generator** | Keeps project documentation current. Updates READMEs to reflect code changes, audits stale comments (never adds unsolicited comments). |

### 🔧 Git Workflow

| Skill | What it does |
|---|---|
| **git-auto-committer** | Scans for modified/untracked files, generates meaningful commit messages, stages and commits each file individually. Optional branch creation + PR. Works in interactive and cron/automated modes. |
| **granular-commits-pr** | Commits every changed file individually with conventional commit messages, creates a fresh feature branch, pushes, and opens a structured PR. |

### 📋 Planning

| Skill | What it does |
|---|---|
| **feature-intake-and-plan** | Guides new feature development from idea to build-ready plan. Structures requirements, identifies constraints, breaks work into actionable tasks, and produces a spec + implementation plan. |
| **changelog-unreleased-workflow** | Manages changelog entries for unreleased work. Tracks in-progress changes, formats conventional commit logs, and prepares release notes. |

---

## Usage

These skills are designed for AI coding agents (Hermes, Claude Code, Codex, Cursor, Goose, Grok, etc.). Each skill is a self-contained `SKILL.md` with a YAML frontmatter description and numbered workflow phases.

To use a skill after installing (or after cloning this repo into your agent’s skills directory), load it and say what you need:

> "Run `security-auditor` on this project"  
> "`tdd-enforcer` — write tests for the auth module"  
> "`py-publish` — bump to v1.2.0 and release"  
> "`/master-reviewer` on PR #8"  
> "Analyze this URL for vulnerabilities: https://example.com"

## Structure

```
skills/
├── README.md
├── package.json
├── .claude-plugin/plugin.json
├── changelog-unreleased-workflow/SKILL.md
├── code-breakdown/SKILL.md
├── code-review-debugger/SKILL.md
├── db-schema/SKILL.md
├── docs-generator/SKILL.md
├── feature-intake-and-plan/SKILL.md
├── git-auto-committer/SKILL.md
├── granular-commits-pr/SKILL.md
├── master-reviewer/SKILL.md
├── py-publish/SKILL.md
├── security-auditor/SKILL.md
├── tdd-enforcer/SKILL.md
└── web-vulnerability-analysis/SKILL.md
    └── references/easevote-case-study.md
```

## Updating

Pull the latest version of any installed skill:

```bash
npx skills update
```

Or update a single skill by name:

```bash
npx skills update tdd-enforcer
```

## Removing

```bash
npx skills remove tdd-enforcer
```

## License

Private collection — personal use unless you open the repo to collaborators.
