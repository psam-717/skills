---
name: feature-intake-plan-loop
description: "Agentic loop around feature-intake-and-plan — autonomously iterate Sense → Plan → Act → Evaluate cycles to converge on a build-ready PLAN.md without manual steering per step."
version: 1.0.0
author: psam-717
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [planning, agentic-loop, feature, intake, autonomous, workflow]
    related_skills: [feature-intake-and-plan, plan, subagent-driven-development]
---

# Feature Intake Plan Loop

## Overview

This is an **agentic loop** wrapper around the `feature-intake-and-plan` skill. Instead of me following a linear checklist, I iterate through autonomous cycles:

```
[F] Sense → [F] Plan → [F] Act → [F] Evaluate → LOOP or EXIT
```

Each cycle checks if we're converging on a build-ready PLAN.md. If not, I re-plan and loop. If stuck, I re-research and re-prompt. If done, I hand off to execution.

**Scope:** the loop logic is project-agnostic. Concrete examples throughout (psamvault-cli's `ak-*` command groups, `api_key_commands.py`, Alembic migrations) are illustrative — mirror whatever conventions the *target* project already has, not psamvault-cli's.

## When to Use

- User says "plan this feature" or "let's design X"
- A vaguely-defined feature request lands
- You want the agent to drive the intake process autonomously
- Before any implementation work

**Load this skill alongside `feature-intake-and-plan`.** This skill is the loop controller; `feature-intake-and-plan` provides the intake templates and decision-making conventions.

---

## The Loop: 4 Stages

### Stage 1: SENSE

Gather everything I need before planning:

1. **Read the project structure** — `find . -maxdepth 2 -type f -name "*.py" | sort`, `pyproject.toml`, README, and key source files in the relevant directory
2. **Read the user's intent** — what feature? what problem does it solve?
3. **Read existing entry types as templates** — if the new feature mirrors an existing one (e.g., a new command group that should follow the pattern of an existing group in *this* project — psamvault-cli's `ak-add`/`ak-list` family is one such example), read the existing command file, its crypto helpers, its API client functions, and its schema/model line-by-line. **The existing type IS the spec** — don't design from scratch. Note the exact fields, response shapes, naming conventions, and error handling patterns.
4. **Check memory + holographic facts** — has the user expressed preferences that affect this?
5. **Produce a context snapshot**: stack, patterns, feature goal, constraints

**Self-check:**
- [ ] Do I understand the feature goal well enough to summarise it in 3 sentences?
- [ ] Do I know the project's language, framework, and key patterns?
- [ ] Have I read the source files of an existing comparable feature to understand the pattern?
- [ ] Are there any obvious constraints (budget, deadline, deployment target)?

If **NO** to any → **Loop back**: ask the user a clarifying question before proceeding.

---

### Stage 2: PLAN

Design what I'll do this cycle:

1. **Identify open decisions** — what design choices are unresolved?
2. **Determine knowledge gaps** — what options do I not fully understand?
3. **Decide the next action:**
   - *Research* — if I need to learn about an option (search docs, read source, check alternatives)
   - *Present* — if I have enough info to offer the user a decision
   - *Update PLAN.md* — if a decision was just made
   - *Handoff* — if all decisions are resolved

**Self-check:**
- [ ] Is the action concrete enough to execute in one cycle?
- [ ] Have I verified this action isn't already done in a previous cycle?
- [ ] Am I researching something I'll actually put to use, or am I stalling?

If the action is vague → **Loop back** and break it down into a smaller next step.

---

### Stage 3: ACT

Execute the planned action:

| Planned Action | What I Do |
|---|---|
| **Research** | `web_search`, `web_extract`, `read_file`, `search_files` to fill knowledge gaps. Or `delegate_task` with research context. |
| **Present** | Offer 3+ options with pros/cons + my recommendation (using the templates from `feature-intake-and-plan`). Use `clarify(multiple_choice)` to let the user pick. |
| **Update PLAN.md** | `patch` the file with the new decision, add completed items, update status. |
| **Handoff** | Transition to `plan` skill for subagent execution. |

**Self-check:**
- [ ] Did the action succeed? (Research returned useful info? User responded to the prompt?)
- [ ] Did I get what I needed, or do I need to iterate?

If the action failed or returned nothing useful → **Mark as failed**, **Loop back** to PLAN and try a different approach.

---

### Stage 4: EVALUATE

Assess where we are and whether to loop again:

**Convergence check:**
- [ ] All design decisions resolved?
- [ ] PLAN.md has build order, file paths, acceptance criteria?
- [ ] Open questions list empty?
- [ ] User has seen and agreed to the plan?

**Loop condition logic:**

| State | Next Action |
|---|---|
| All resolved, user confirmed | ✅ **Exit loop** → Handoff to build |
| Some decisions still open | 🔄 **Loop back** → SENSE what's left, PLAN a presentation |
| User asked a question mid-cycle | 🔄 **Loop back** → answer, then re-EVALUATE |
| We've been on the same decision for 3+ cycles | ⚠️ **Escalate** → Tell the user "We seem stuck on X. Can you pick a direction or shall I recommend one?" |
| Research returned nothing or error | 🔄 **Loop back** → try a different search, or ask the user for a source |
| I proposed options and the user said "I don't know, you decide" | → **Use my recommendation** from the presentation, update PLAN.md, continue looping |

**Token budget check:**
- If this loop has run 8+ cycles, summarise what's been done and ask the user if they want to continue or wrap up with what we have.

---

## Build Phase (Post-Planning Continuation)

When planning is done and the user confirms "build it," **do not stop**. Transition into build execution:

1. **Read the PLAN.md build order** — pull the files likely to change and acceptance criteria
2. **Read an existing comparable feature in full** — the existing pattern IS the spec. Read every line of a similar feature in *this* project (e.g., the sibling command group the new one should mirror — in psamvault-cli that's `api_key_commands.py` as the template for `note_commands.py`), its crypto helpers, API client functions, and wiring in `main.py`. Copy-paste the whole file mentally before writing anything new.
3. **Build layer by layer:**
   - **Backend first** (if applicable): model → schema → CRUD → routes → migration → wire into main.py
   - **Then client:** crypto helpers → API client functions → command file → wire into main.py → update list/search/export/import
4. **Verify each layer** — syntax check, import check, file parse check after each batch of changes
5. **Git workflow** — after completing a layer, commit it to a feature branch with a conventional commit message. Don't wait until all layers are done to branch/commit.
   - `git checkout -b feature/<feature-name>`
   - Commit with conventional message: `feat: add <what> — <detail>`
   - **Push with token auth** when HTTPS interactive auth fails (`No such device or address`):
     - `git push https://psam-717:$(gh auth token)@github.com/<owner>/<repo>.git <branch>`
     - Or set up the remote once: `git remote set-url origin https://<user>:<token>@github.com/<owner>/<repo>.git`
   - Create PR: `gh pr create --base main --head <branch> --title "..." --body "..."`
   - **Do not wait to be asked** — the PR is part of the deliverable, not an optional follow-up. If the user has to ask "did you create a PR?" you missed a step. Push and create the PR immediately after committing the feature branch, before reporting what was built.
6. **Update PLAN.md** — mark each step ✅ as you complete it
7. **Report progress** — after each layer, tell the user what was built, the PR link, and what's next

**Key pattern for new command groups:** model new files after an existing type in the same project. In psamvault-cli, for example, `ak-add` / `ak-list` / `ak-get` / `ak-delete` / `ak-update` are the template for any new command group, `encrypt_api_key` / `decrypt_api_key` are the template for any new encryption helper, and `add_api_key_entry` / `list_api_key_entries` are the template for any new API client functions. Whatever the project, find the equivalent existing group and mirror it exactly — never import psamvault-cli conventions wholesale.

## Loop Termination Conditions

The loop exits naturally when:

1. **All decisions resolved** — every open question is marked [x] with a documented answer
2. **User explicitly says "go ahead" / "build it" / "implement"** — this transitions to the Build Phase instead of terminating
3. **User says "stop" or "save it for later"**
4. **8 cycles reached** — pause and ask if the user wants to continue
5. **All remaining decisions are truly the user's preference** and you've already recommended — present the final state and exit

When exiting without building, update the PLAN.md status to **🟢 READY** or **🟡 PAUSED** accordingly.

---

## Integration with feature-intake-and-plan

This loop **does not replace** `feature-intake-and-plan`. It orchestrates it:

1. **Load this skill** to activate the loop controller
2. **Load `feature-intake-and-plan`** for the templates, conventions, and principles
3. The loop calls the intake templates from `feature-intake-and-plan` during the ACT phase
4. The loop uses the same PLAN.md format and build-order structure
5. The Build Phase uses the build order and acceptance criteria from PLAN.md to drive implementation

**Recommended load order:**
```yaml
skills:
  - feature-intake-plan-loop
  - feature-intake-and-plan
```

---

## Common Pitfalls

1. **Researching forever** — Limit research to 2 cycles per open decision. If you can't find what you need, ask the user.
2. **Presenting too many decisions at once** — Max 3 per cycle unless the user asks for the full picture.
3. **Forgetting to update PLAN.md** — Every decision must be written to the file before the next cycle.
4. **Not tracking what cycle you're on** — Keep an internal count. At cycle 8, stop and check with the user.
5. **Ignoring the user's signal to stop** — If the user says "let's just start with option 1", accept it. Don't keep looping for completeness.
6. **Re-asking already-resolved decisions** — Check PLAN.md's "Decisions Made" section before presenting again.
7. **Re-searching something you already know** — If you already have the answer from memory or context, don't search; just present. Search only for genuine gaps.
8. **Dense presentation when presenting many decisions** — When you have 3+ decisions with 3+ options each, the output becomes a wall of text the user can't digest. Default to presenting 1–2 decisions per cycle. If you must present many at once, structure each decision as a compact table with a title line, not a paragraph per option. See `references/presentation-format.md` for the lean format pattern.
9. **Stopping after planning** — The user often wants you to keep building after the plan is done. Don't ask "do you want me to build it now?" — they already said yes by picking the feature. Transition directly into the Build Phase.
10. **Designing new patterns from scratch** — When adding a feature that mirrors an existing type (new command group, new entry type, new encryption helper), read the existing implementation in full first. Do not invent new patterns for naming, error handling, pagination, response shapes, or API conventions. Mirror what's already there — the user's project has established conventions.
11. **Building backend and client out of order** — Always build backend first (model → schema → CRUD → routes → migration → wiring), then client (crypto → API client → commands → wiring → list/search/export updates). This order lets you verify the backend before the client depends on it, and the backend endpoints define the contract the client must match.
12. **Forgetting to create the PR** — Building the feature is only half the work. After committing the feature branch, push and create the PR immediately. If the user has to ask "did you create a PR?", that's a missed step. The PR link is part of the deliverable summary.
13. **Alembic migration parented to wrong revision** (only applies when the target project uses Alembic for DB migrations) — When creating a new Alembic migration, always check what the CURRENT head of the chain is, not just any parent in the chain. Run `ls migrations/versions/` and read the `revision` field of the most recent file. Two migrations that both point to the same parent create a fork — Alembic refuses to run `upgrade head` with multiple heads. The new migration's `down_revision` must be the LATEST revision ID, not an intermediate one. If in doubt, find the head by checking which revision is NOT referenced as `down_revision` by any other migration.
14. **`write_file` overwrites the entire file — do not use it for appending** — `write_file` always replaces the full file content. If you need to add a function to an existing file, use `patch` with `old_string`/`new_string` or `execute_code` for multi-step edits. The only safe uses of `write_file` on existing files are: (a) creating a new file, or (b) replacing the entire content of a file you just read in full (not via offset/limit pagination). If you accidentally clobber a file, restore from git: `git checkout main -- <file>`.
15. **Avoid `patch` for inserting multi-line blocks with escaped backslashes** — When editing Python multi-line strings (f-strings with `\n`, `\\n`), the `patch` tool can mangle escaping and indentation. After any patch that touches string literals with backslash escapes, immediately syntax-check the file. If the escaping broke, restore from git and write the affected function as a complete block instead of a targeted patch.
16. **`web_extract` fails silently with DuckDuckGo backend** — When `web_extract` says "DuckDuckGo is a search-only backend", do NOT keep retrying with different URLs. Switch to `curl` via `terminal()` or use the `browser` tools instead. Three consecutive retries of the same failing tool is the tool-loop warning threshold — detect it earlier.

## Verification Checklist

At each EVALUATE stage, run this:

- [ ] PLAN.md exists and is up to date
- [ ] Decisions Made section reflects all user answers so far
- [ ] Open questions are decreasing (not growing) every 2 cycles
- [ ] Every open question has at least 3 options defined (or a clear reason it can't)
- [ ] Build order has been drafted (even partially)
- [ ] Acceptance criteria exist (even draft form)
- [ ] No more than 2 consecutive research cycles without presenting to the user
- [ ] Token budget: cycles < 8
