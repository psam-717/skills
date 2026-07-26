# Custom AI Coding Skills

A collection of reusable AI agent skills that level up your coding workflow — from test-driven development to automated releases.

## Skills

### 🔍 Code Quality
| Skill | What it does |
|---|---|
| **code-breakdown** | Thorough, beginner-friendly explanation of an entire codebase. Reads every file and walks through structure, data flow, and key code. |
| **code-review-debugger** | Full codebase review and debugging session. Finds bugs, security issues, logic errors, and quality problems — presents a report for approval before touching anything. |
| **security-auditor** | Proactive vulnerability audit. Scans for injection, auth flaws, secrets exposure, dependency risks, and data validation gaps. Prioritized report → approved surgical fixes. |

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

---

## Usage

These skills are designed for AI coding agents (Hermes, Claude Code, Goose, etc.). Each skill is a self-contained `SKILL.md` with a YAML frontmatter description and numbered workflow phases.

To use a skill, load it into your agent and say what you need:

> "Run `security-auditor` on this project"
> "`tdd-enforcer` — write tests for the auth module"
> "`py-publish` — bump to v1.2.0 and release"

## Structure

```
skills/
├── README.md
├── code-breakdown/SKILL.md
├── code-review-debugger/SKILL.md
├── db-schema/SKILL.md
├── docs-generator/SKILL.md
├── git-auto-committer/SKILL.md
├── granular-commits-pr/SKILL.md
├── py-publish/SKILL.md
├── security-auditor/SKILL.md
└── tdd-enforcer/SKILL.md
```

## License

Private collection — personal use.
