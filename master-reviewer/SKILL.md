---
name: master-reviewer
description: >-
  Run a three-round master review loop on an open GitHub PR: thorough review,
  wait for the user to report that findings were tackled, verify fixes, then
  deepen the next pass. After the third pass is verified, declare merge-ready
  or not. Invokes the agent's built-in /review (or equivalent) skill when
  available; otherwise reviews for bugs, security, and effectiveness with
  actionable fix guidance on every finding. Always auto-submits PR reviews
  (never leaves PENDING) and posts verification outcomes both in chat and on
  the PR. Use when the user runs /master-reviewer, asks for a "master review",
  "three-round PR review", "multi-pass PR review", or "review this PR
  thoroughly three times".
argument-hint: "<pr-number-or-url>"
---

# Master Reviewer

You are the **orchestrator** for a fixed three-round PR review process. You do
not invent a parallel review engine when the agent already has one: you
**invoke the built-in review skill** (or the closest equivalent) for each
pass, then manage wait → verify → next-pass sequencing and the final merge
gate.

## Goals

1. Conduct **exactly three** thorough review rounds on an **open** GitHub PR.
2. After each review is **submitted and published on the PR** (never left
   PENDING for the user to submit manually), **stop and wait** for the user to
   say the review has been tackled (or equivalent).
3. Before rounds **2** and **3**, **verify** that previously highlighted issues
   were addressed; only then start the next full review.
4. After the user reports that **round 3** is tackled, **verify again** and
   either give **merge go-ahead** or list remaining blockers.
5. Every finding must include a **concrete, best-path suggestion** so whoever
   fixes the PR can implement a strong solution—not just "please fix this".
6. Each successive review is **more detailed and thorough** than the last so
   the PR ends neat, safe, and maintainable.
7. **Dual-channel outcomes:** every full-review summary and every verification
   outcome must appear **in chat** and **on the GitHub PR** as a submitted
   review (and residual fix guidance when anything is still open), so anyone
   working from the PR alone knows exactly what to do next.

## Invocation

```
/master-reviewer <pr-number-or-url>
/master-reviewer https://github.com/owner/repo/pull/N
/master-reviewer 8
```

### Argument parsing

1. Require a PR number or URL. If missing, ask for one and stop.
2. Accept:
   - Full URL matching `https://github.com/<owner>/<repo>/pull/<n>`
   - Bare digits or `#N` as PR number in the current repo
3. Resolve with `gh pr view <target> --json number,title,url,state,headRefOid,baseRefOid,headRefName,baseRefName`.
4. If `state` is not `OPEN`, stop: master-reviewer only runs on open PRs.
5. Store for the whole run: `pr_number`, `pr_url`, `owner`, `repo`, `pr_title`.

## Durable session state

Keep a short state block in your working memory (and restate it to the user
after each phase) so multi-turn waits resume cleanly:

```
MASTER_REVIEW_STATE
- PR: #<n> <url>
- Round completed: 0|1|2|3
- Phase: review | wait_user | verify | merge_gate | done
- Last head SHA: <sha>
- Open issues from last review: <count> (bugs / suggestions / nits)
- Outstanding prior issues: <short bullets or "none">
- Last published review id / url: <if any>
```

Do **not** invent a separate on-disk protocol unless the user asks; conversation
state plus GitHub PR head SHA and prior review comments are enough.

## Auto-submit reviews (mandatory)

**Never leave a review in PENDING for the user to submit manually.** After every
inspection that produces a GitHub PR review (full round, verification, or merge
gate), the review must be **published immediately**.

### How to publish

Use `event: "COMMENT"` for master-reviewer reviews unless the user explicitly
asks for `REQUEST_CHANGES`. Do **not** use `APPROVE` for rounds 1–3 findings
reviews. Only the final merge-gate message may recommend merge in prose; it
still posts as `COMMENT` (do not auto-approve unless the user explicitly asks).

**Preferred create path** (single shot, already submitted):

```bash
gh api "repos/${owner}/${repo}/pulls/${pr_number}/reviews" -X POST --input payload.json
```

Payload must include:

```json
{
  "commit_id": "<head_sha>",
  "body": "<markdown summary>",
  "event": "COMMENT",
  "comments": [ /* optional inline comments */ ]
}
```

**If a helper skill creates a PENDING review** (no `event` field — e.g. the
bundled `/review` skill):

1. Capture the returned review `id` from the create response.
2. Immediately submit it:

```bash
gh api "repos/${owner}/${repo}/pulls/${pr_number}/reviews/${review_id}/events" \
  -X POST -f event=COMMENT
```

3. Confirm the review `state` is not `PENDING` (expect `COMMENTED` or equivalent).
4. If submit fails, report the error, keep local notes, and retry once or post a
   fresh review with `event: "COMMENT"` so the PR is not left without a public
   trail.

**Override for built-in review skill:** When orchestrating `/review` (or similar)
under master-reviewer, treat its "user submits PENDING" rule as **superseded**.
Master-reviewer always auto-submits. Tell the user the review is already live on
the PR; do not ask them to click "Submit review".

After publish, include in chat:

- Review id
- Link to the PR Conversation or review (`html_url` from the API is fine)
- That the review is **already submitted** (visible to collaborators)

## Review engine selection (required)

Before round 1, pick how reviews are performed:

### A) Preferred — built-in review skill

If this agent has a PR review skill (commonly `/review`, `review`, or a
bundled "review" skill that posts GitHub PR reviews):

1. **Read and follow that skill** for each full review pass (load its
   `SKILL.md` and execute its PR mode against the target PR), **except** for
   submit behavior (see Auto-submit above).
2. Pass framing into that skill's flow so the pass depth and suggestion rules
   below still apply (e.g. include them in the reviewer prompt / task context
   when the skill allows).
3. After the skill posts (PENDING or otherwise), **auto-submit** so the review
   is published with `event: COMMENT`. Do not wait for manual UI submit.

### B) Fallback — no review skill available

If no review skill exists:

1. Fetch the PR diff with `gh pr diff` and metadata with `gh pr view`.
2. Read changed files and surrounding call sites.
3. Review with this priority order:
   1. **Correctness / bugs** (logic errors, bad defaults, race conditions,
      invalid state, broken edge cases)
   2. **Security** (authz/authn, secrets, injection, open webhooks, insecure
      defaults, multi-tenant leaks)
   3. **Effectiveness & efficiency** (wrong abstraction, N+1, full scans,
      missing validation, operability, clarity)
4. Post findings as a GitHub PR review via `gh api` when authenticated with
   **`event: "COMMENT"`** (auto-submitted), with actionable suggestions on
   every item. If posting is impossible, write a clear markdown report in chat
   with the same structure and note that the PR could not be updated.

## Round depth ladder (mandatory)

Each full review must be **deeper** than the previous one. Do not repeat the
same shallow pass.

| Round | Name | Focus |
|------:|------|--------|
| 1 | **Foundation** | Correctness, security, data integrity, auth, obvious breakage. Catch merge-blockers and high-severity bugs first. |
| 2 | **Hardening** | Re-verify round-1 fixes; dig into edge cases, error paths, multi-tenant isolation, API contracts, race/consistency, incomplete validation, deploy/config footguns related to the change. |
| 3 | **Polish & merge-bar** | Re-verify all prior fixes; maintainability, naming consistency with codebase, tests/docs gaps that risk regression, operability (logs, health, env), dead paths, nits that still matter for a clean merge. Be stricter and more thorough than round 2. |

If a round finds **zero issues**, still post/report a short **submitted** summary
stating that the round is clean at that depth, then wait for the user before
continuing (except after final verify—see merge gate).

## Suggestion quality (every finding)

For **every** issue in **every** round, include:

1. **What is wrong** (specific, with file and line when possible)
2. **Why it matters** (user impact, security, correctness, ops)
3. **Best way to tackle it** — preferred fix path, not a vague "fix this":
   - Preferred approach (concrete)
   - What to avoid (common weak fix)
   - Acceptance check (how to know it is fixed)

Example shape inside a review comment or issue block:

```text
**[bug]** Description of the defect.

**Why it matters:** …

**Best fix:** …
**Avoid:** …
**Done when:** …
```

When using the built-in review skill's `Suggestion:` field, put the full
best-fix / avoid / done-when content there so fixers are guided.

## Main loop

```
resolve PR
pick review engine (A or B)
round = 1
while round <= 3:
    run full review for this round (depth ladder)
    publish review on PR with event COMMENT (auto-submit; never leave PENDING)
    report summary to user in chat + link to live review
    set phase = wait_user
    STOP and wait for user signal that the round was tackled
    on user "tackled" / "fixed" / "addressed" / similar:
        run VERIFY for issues from the last completed round
        publish verification outcomes on the PR (submitted COMMENT review)
        report the same verification table + residual guidance in chat
        if verify finds prior issues still open:
            residual gaps + best-fix must appear on PR and in chat
            stay in wait_user for this round (do not advance)
            continue waiting
        if round < 3:
            round += 1
            run next full review
        else:
            run MERGE GATE (final verification)
            publish merge-gate outcome on PR + chat
            end
```

### User wait rules

- After publishing a round, clearly tell the user:
  - Round number completed
  - Issue counts by severity
  - Link to the **already submitted** review
  - **Exact next step:** e.g. "When you have addressed these findings, reply
    that the review has been tackled (e.g. `review tackled` or
    `round 1 done`)."
- Do **not** start the next review or verify until the user indicates the
  current round's findings were tackled (or they explicitly ask to continue /
  re-check).
- Phrases that count as go-ahead to verify: "tackled", "addressed", "fixed",
  "handled", "done", "ready for next", "verify", etc. If ambiguous, ask once.
- Do **not** ask the user to submit a PENDING review in the GitHub UI.

### Verify phase (before rounds 2 and 3, and after round 3)

Verification is **not** a full new depth review. It is a targeted check of
prior findings. Outcomes must be dual-channel: **chat + PR**.

1. Refresh PR head (`gh pr view` → `headRefOid`). Note if SHA changed.
2. Collect prior open findings from the last review (chat summary + GitHub
   review comments if needed).
3. For each prior finding, inspect current code/diff and mark:
   - **Fixed**
   - **Partially fixed** (what remains)
   - **Still open**
   - **Regressed / new break from the fix**
4. Report a verification table to the user **in chat**.
5. **Also publish a submitted PR review** (`event: "COMMENT"`) so anyone
   working from the PR can act without reading chat. See
   [Verification review body](#verification-review-body-template) below.
6. **Advance only if** there are no remaining **bugs** from the prior round
   (and no partials that still break correctness/security). Pure nits may be
   carried into the next deeper round if the user wants speed—but default is:
   **prior bugs must be fixed before the next full review.**
7. If still broken, do not start the next full review; wait again after listing
   what remains and refreshed best-fix guidance **on the PR and in chat**.

#### Verification review body (template)

Post one top-level review body (inline comments optional if a residual maps
cleanly to a diff line). Title the review clearly, e.g.
`Master review — Round N verification`.

```markdown
## Master review — Round <N> verification

**PR head:** `<sha>`
**Result:** PASS (advance) | FAIL (do not advance) | PASS WITH CARRY (nits only)

### Checklist

| # | Severity | Finding (short) | Verdict | Notes |
|---|----------|-----------------|---------|-------|
| 1 | bug | … | Fixed / Partially fixed / Still open / Regressed | … |

### Still open / partial — exact next actions

For each non-Fixed item, repeat full guidance:

**[severity] Title**
- **What remains:** …
- **Why it matters:** …
- **Best fix:** …
- **Avoid:** …
- **Done when:** …

### Fixed this pass

- Short bullets of what was confirmed fixed (optional but helpful).

### Orchestrator decision

- Advancing to Round <N+1> | Waiting for remaining bug fixes | Merge gate: READY / NOT READY
```

Anyone reading only the PR must be able to implement remaining work from this
body alone (no dependency on chat history).

Optional: for residual issues whose `(path, line)` is still on the current
diff, add inline comments with the same best-fix block so they show on Files
changed.

### Full review phase (rounds 1–3)

1. Announce: `Master review — Round N/3 (<name>) on PR #…`
2. Invoke the selected review engine with **round-specific depth** and
   **suggestion quality** requirements injected into context.
3. For rounds 2–3, also inject:
   - List of prior findings and verify results
   - Instruction: do not re-open items marked Fixed unless still broken;
     hunt for **new** issues at this deeper level and any regressions
4. Ensure the review is **published** with `event: COMMENT` (auto-submit).
5. Summarize in chat: counts, top issues, best-fix highlights, live review
   link, wait prompt. Do not instruct manual GitHub submit.

### Merge gate (after round 3 is tackled + verified)

1. Run a final verify over **all** issues raised across rounds 1–3 (or at
   least everything still open after round 3).
2. Optionally skim CI (`gh pr checks`) and conflict state
   (`gh pr view --json mergeable,mergeStateStatus`).
3. Decide:
   - **MERGE READY** — no open bugs/security issues; remaining nits are
     optional or none. Explicitly say: **Go ahead and merge.**
   - **NOT MERGE READY** — list blockers with best-fix guidance; stay in
     wait_user until the user tackles again, then re-run merge gate only
     (no fourth full review unless the user asks to restart).
4. Publish that decision as a **submitted** `COMMENT` review on the PR (same
   dual-channel rule) in addition to the chat summary.

## Resume behavior

If the user returns mid-loop (new message after wait):

1. Restate `MASTER_REVIEW_STATE`.
2. If they signal tackled → verify → publish verification on PR → advance as above.
3. If they ask "status" → report state without starting a new review.
4. If they ask to skip a round → refuse by default; only skip if they
   explicitly override ("skip to merge gate" / "skip round 2").
5. If the PR was closed/merged mid-loop → stop and report.

## Output standards

- No emojis required.
- Be direct and specific; severity-first ordering (bugs → suggestions → nits).
- Do not flood with low-value nits in round 1; deepen over rounds.
- Never modify PR code yourself unless the user separately asks you to fix.
- Never approve merge after round 1 or 2; only the **merge gate** after
  round 3 verification may say **Go ahead and merge.**
- Always dual-channel: chat summary + submitted PR review for full rounds,
  verifications, and merge gate.
- Always auto-submit; never leave PENDING reviews for manual UI submit.

## Quick checklist (orchestrator)

- [ ] PR open and identified
- [ ] Review engine selected (built-in skill preferred)
- [ ] Round 1 full review **published** (COMMENT, not PENDING) + chat summary + user wait
- [ ] User tackled → verify → **publish verification on PR** + chat → only then Round 2 if bugs clear
- [ ] Round 2 full review **published** + user wait
- [ ] User tackled → verify → **publish verification on PR** + chat → only then Round 3 if bugs clear
- [ ] Round 3 full review **published** + user wait
- [ ] User tackled → final verify → **publish merge gate on PR** + chat → **merge go-ahead or blockers**
- [ ] Every finding carried a best-fix / avoid / done-when style suggestion
- [ ] No review left PENDING; no "please submit in the UI" instructions
