---
name: feature-intake-and-plan
description: "Guide new feature development from idea to build-ready plan: structured summary → multiple-choice questions → PLAN.md collaboration → handoff to plan and subagent-driven-development skills."
version: 1.0.0
author: psam-717
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [planning, feature, intake, workflow, design, brainstorm]
    related_skills: [plan, subagent-driven-development, test-driven-development, writing-plans]
---

# Feature Intake & Plan

## Overview

This skill encodes the process used for the psamvault-mCP hackathon features: a structured, question-driven approach to turning a feature idea into a build-ready PLAN.md.

It covers two phases before any code is written:

1. **Intake** — Summarize the feature proposal, provide key decision points with multiple-choice answers
2. **Collaborative Planning** — Continuously update PLAN.md as we brainstorm, let the user ask clarifying questions

Then hands off to `plan` and `subagent-driven-development` for execution.

## When to Use

- User says "I have an idea for a feature" or "Let's add X"
- A feature request is vague and needs structured exploration
- You need to document trade-offs and get the user's input on design decisions
- Before any implementation that affects multiple files or has design choices

**Do not use for:**
- Simple bug fixes (use `systematic-debugging`)
- One-file changes with no design decisions (just implement)
- Tasks already fully specified (skip straight to `writing-plans`)

## The Process

### Phase 1: Intake

When the user proposes a feature idea:

#### Step 1: Write a structured summary

Create a PLAN.md at the project root (or `hackathon/PLAN.md` if the project has one):

```markdown
# [Feature Name]

**Status:** 🟡 EXPLORING

**Proposed by:** User
**Date:** YYYY-MM-DD

---

## Summary

[A 3-5 sentence description. What it does, why it's needed, who it's for.]

## Key Points

1. **Point A:** [One sentence]
2. **Point B:** [One sentence]
3. **Point C:** [One sentence]

## Key Decisions Needed

### Decision 1: [Title]

**Context:** [2-3 sentences explaining the trade-off]

| # | Option | Pros | Cons |
|---|--------|------|------|
| 1 | **[Option A]** | [Key pro] | [Key con] |
| 2 | **[Option B]** | [Key pro] | [Key con] |
| 3 | **[Option C]** | [Key pro] | [Key con] |

## Open Questions

- [ ] Question 1?
- [ ] Question 2?
```

#### Step 2: Present each decision with 3+ choices

Structure each choice with what it is, when it makes sense, and the trade-off:

```
**Decision: Storage approach**

Where should the data live?

1. **In-memory** — Simplest, never persists. ✓ Fast to build ✗ Lost on restart
2. **SQLite** — Persistent, zero setup. ✓ Simple ✗ Won't scale horizontally
3. **PostgreSQL** — Production-grade. ✓ Scalable ✗ Requires a database server

Which approach fits best?
```

#### Step 3: Let the user ask questions

```
"These are the key design decisions I see. You can:
- Pick an option for any decision
- Ask me questions about the trade-offs
- Suggest a different approach I hadn't considered

What would you like to start with?"
```

### Phase 2: Collaborative Planning

#### Step 1: Update PLAN.md immediately

Every time a decision is made or the scope changes, update PLAN.md:

```markdown
## Decisions Made

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Storage | SQLite | No need for server, data must persist |

## Open Questions

- [x] ~~Storage approach?~~ → SQLite
- [ ] Auth method?
```

Use `patch` for targeted edits rather than rewriting the whole file each time.

#### Step 2: When all decisions are made, transition to build-ready

```markdown
## Build Order

| Step | Feature | Depends On | Est. Time | Status |
|------|---------|-----------|-----------|--------|
| 1 | [Smallest unit] | Nothing | X hours | 🔴 |
| 2 | [Next step] | Step 1 | X hours | 🔴 |
| ... | ... | ... | ... | ... |

## Files Likely to Change

- `src/path/to/new_file.py` — [What it does]
- `src/path/to/existing.py` — [What changes]
- `tests/path/to/test.py` — [New tests]

## Acceptance Criteria

- [ ] Criterion 1
- [ ] Criterion 2

## Risks & Mitigations

- [Risk 1] — [Mitigation]
- [Risk 2] — [Mitigation]
```

#### Step 3: Offer execution

```
"The plan is ready. Want me to:
1. **Build it now** — Follow the plan step by step
2. **Save it** — You review first, we build later
3. **Refine** — Anything to change in the plan?
"
```

### Phase 3: Handoff to Build

When confirmed, load and follow:
1. `plan` — For detailed task-level implementation plans
2. `subagent-driven-development` — For executing tasks with TDD + 2-stage review
3. `test-driven-development` — For the TDD cycle within each task

## Principles

- **Always present multiple choices** — 3+ options help the user see trade-offs they hadn't considered
- **Update PLAN.md immediately** — Every decision goes in right away, not batch-updated later
- **Let the user drive** — Present options and trade-offs, recommend when asked, but the user decides
- **Document rejected alternatives** — Note what was rejected and why, prevents re-litigation
- **Respect existing architecture** — Prefer solutions aligned with project patterns over new frameworks

## Common Pitfalls

1. **Yes/no questions** — Always provide 3+ options with pros/cons
2. **Dumping all decisions at once** — Present 2-3 decisions at a time
3. **Ignoring implementation cost** — Include estimated effort for each option
4. **Skipping the PLAN.md update** — The file is the source of truth
5. **Moving to build too early** — All decisions should be resolved first
6. **Vague PLAN.md** — Must have exact file paths, test targets, acceptance criteria

## Verification Checklist

- [ ] PLAN.md created with summary and key decisions
- [ ] Each decision has 3+ options with pros/cons
- [ ] User invited to ask questions or pick options
- [ ] PLAN.md updated after every decision/conversation
- [ ] All decisions resolved before build phase
- [ ] Build order, file paths, and acceptance criteria documented
- [ ] Rejected alternatives noted with rationale
- [ ] Handoff offered (build, save, or refine)
