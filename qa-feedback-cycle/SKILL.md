---
name: qa-feedback-cycle
description: >
  QA feedback collection loop — user tests the app, reports issues verbally, and the
  agent verifies, categorises, and logs them to a FEEDBACK.md file for batch processing.
  Designed for manual app testing workflows.
---

# QA Feedback Cycle

Use when the user is manually testing an app and reporting issues, suggestions, or observations verbally. This skill codifies the workflow: verify → log → batch → fix.

## Workflow

### Step 1 — User reports a finding
The user says something like:
- "I noticed that when I do X, Y happens"
- "Can we make X better?"
- "There's a problem with..."

### Step 2 — Verify & gather evidence
Before logging, verify the claim by checking the relevant source files. Use `search_files()` and `read_file()` to find the relevant code or configuration. Confirm the problem is real before logging.

### Step 3 — Categorise
Each entry gets a category:
- 🐛 **Bug** — Something broken or not working as expected
- 💡 **Suggestion** — Idea for improvement
- 🎨 **UI/UX** — Visual or usability feedback
- ⚡ **Performance** — Speed or reliability
- 🔒 **Security** — Security concern

### Step 4 — Log to FEEDBACK.md
Add a new entry to the project's `FEEDBACK.md` with:
- Entry number (sequential)
- Category
- Date filed
- Problem description in user-friendly terms
- Evidence (file paths + line numbers + relevant code snippets)
- Scope of fix (what needs to change)

### Step 5 — Continue collecting
Don't fix anything yet. Just log. Tell the user the entry has been filed and keep collecting.

### Step 6 — When user says "I'm done"
The user signals testing is complete. At that point:
1. Push FEEDBACK.md to GitHub so the user can review everything
2. Present the full list of entries with their categories
3. Ask if they want to start tackling them

## Entry Format

Each entry follows this structure in FEEDBACK.md:

```
## Entry N — Short descriptive title

- **Category:** 🐛 Bug / 💡 Suggestion / 🎨 UI/UX / ⚡ Performance / 🔒 Security
- **Filed:** YYYY-MM-DD
- **Problem:** Description of the issue in plain language.

**Evidence:**
- `path/to/file.ext` line N: relevant code/snippet

**Scope of fix:**
1. Step one
2. Step two
...
```

## Important Rules
- Do NOT fix anything during the feedback collection phase. Log only.
- Verify every claim before logging — don't take the user's word at face value, check the code.
- Be specific with file paths and line numbers in the evidence section.
- If the user reports something that's already logged, say so and avoid duplicates.
- When the user says they're done testing, push FEEDBACK.md to GitHub and prepare the list for review.
