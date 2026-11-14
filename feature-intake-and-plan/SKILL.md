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

#### Step 0: Map the design tree, then compute the frontier

Before writing anything down, model the work as a **design tree**: every decision
branches into the decisions that hang off it. The **frontier** is every decision
whose prerequisites are already settled — the questions that can be asked *now*
without guessing at an answer you have not heard yet.

- Ask the whole frontier. Never ask a question whose answer depends on one that is
  still open; that question belongs to a **later round**.
- Recompute the frontier after every round: settled answers push it outward.
- **Facts are your job, decisions are the user's.** Anything discoverable from the
  repo, the docs, an installed skill, or the web gets looked up (delegate it if it
  is big); never spend a round asking the user something you can find. A lookup
  still running is an unsettled prerequisite, so only the questions downstream of
  it wait — ask the rest of the frontier now.
- Track the tree in PLAN.md (open / settled / queued) so the frontier is visible
  rather than remembered.

Step 1 writes that tree into PLAN.md: the summary, the decisions currently on the
frontier, and the questions queued behind them.

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

#### Step 2: Ask one round — the frontier, numbered, each with a recommendation

A round is the current frontier presented together and numbered — then **a stop**.
Do not open the next round, and do not start building, until the user has answered.

- Every question carries **3+ options**, each with what it is, when it makes sense,
  and the trade-off (the psam convention).
- Every question carries **your recommendation**, marked inline. Word the question
  so a bare "yes" accepts it.
- Cap a round at about five questions, grouped by decision area, shallowest first.
  If the frontier is larger, say which questions are queued behind these: that is a
  density limit on presentation, never permission to ask a dependent question early.
- Answer with a compact table per question, not a paragraph per option.

Structure each choice with what it is, when it makes sense, and the trade-off:

```
**Decision: Storage approach**

Where should the data live?

1. **In-memory** — Simplest, never persists. ✓ Fast to build ✗ Lost on restart
2. **SQLite** — Persistent, zero setup. ✓ Simple ✗ Won't scale horizontally
3. **PostgreSQL** — Production-grade. ✓ Scalable ✗ Requires a database server

Which approach fits best?
```

#### Step 3: Close every fact gap, then let the user answer

Before a round goes out, close the fact gaps it exposes: read the file, check the
installed skill, search the docs, or delegate the lookup. Present decisions only.
Then close the round with an invitation that covers all three:

> These are the key design decisions I see. You can pick an option for any of them,
> ask about the trade-offs, or suggest an approach I had not considered. What would
> you like to start with?

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
- **Facts are yours, decisions are the user's** — look up anything the environment
  can answer rather than making the user your search engine
- **One round at a time** — the frontier goes out numbered, with a recommendation per
  question, and then you wait; questions downstream of an open answer belong to the
  next round
- **Done is mechanical** — intake ends when the frontier is empty: every branch of the
  design tree visited, nothing silently assumed, and the user confirms the shared
  understanding. "All decisions resolved" is not a test; an empty frontier is
- **Document rejected alternatives** — Note what was rejected and why, prevents re-litigation
- **Respect existing architecture** — Prefer solutions aligned with project patterns over new frameworks
- **Ship the asked-for scope; extras stay visibly optional** — when an agreed sentence carries an
  optional add-on ("and optionally a <second destination>"), implement the core as the deliverable and
  either leave the add-on switchable or name it as extra when you report. Silently folding it in makes
  the user audit something they never asked for.
- **A question is not a change request** — "why is X included?" asks for the reason, nothing more.
  Reply with the reason and stop; never flip behaviour, disable a feature, or open a patch on the
  strength of a question. Wait for an instruction.

## Common Pitfalls

1. **Yes/no questions** — Always provide 3+ options with pros/cons
2. **Asking the whole tree at once** — Ask the *frontier* only: a question whose
   answer depends on an open one belongs to the next round. Dependency sets the
   order; density is only a presentation cap (~5 per round, grouped)
3. **Being the user's search engine** — a question you could have answered by reading
   the repo, a skill, or a doc is a wasted round
4. **Ignoring implementation cost** — Include estimated effort for each option
5. **Skipping the PLAN.md update** — The file is the source of truth
6. **Moving to build too early** — The frontier must be empty first, not "most decisions done"
7. **Vague PLAN.md** — Must have exact file paths, test targets, acceptance criteria

## Verification Checklist

- [ ] Frontier is empty: every branch of the design tree visited, nothing silently assumed
- [ ] PLAN.md created with summary and key decisions
- [ ] Each decision has 3+ options with pros/cons
- [ ] User invited to ask questions or pick options
- [ ] PLAN.md updated after every decision/conversation
- [ ] All decisions resolved before build phase
- [ ] Build order, file paths, and acceptance criteria documented
- [ ] Rejected alternatives noted with rationale
- [ ] Handoff offered (build, save, or refine)
