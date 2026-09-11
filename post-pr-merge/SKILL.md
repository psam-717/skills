---
name: post-pr-merge
description: "Verify any merged PR end-to-end across projects."
version: 1.1.0
author: Psam
license: MIT
---

# Post-PR-Merge Verification (generic)

## Overview

After the user merges any PR on any project repo, verify end-to-end before they manually test: confirm the merge really landed, sync the local clone, run the project's tests, verify the deploy/live effect (not just trust the deploy trigger), run hygiene checks, clean up merged branches, confirm changelog coverage, and report **verdict-first in structured markdown** (the user's standing requirement).

This is the generic skill. Project-specific deep details live in sibling skills (`whatsapp-sales-post-merge`, `pulse-post-merge`) and in the project profile table below — load the matching project skill when one exists and use it for the detailed steps; this skill keeps the shared skeleton and the cross-project pitfalls.

## When to Use

- User says "PR X is merged" / "I merged it" and expects verification
- User asks "is everything live?", "verify the merge", "confirm it works" after merging
- Post-deploy sanity after any merge on psamvault-cli, psam_vault_backend, pulse, whatsapp-sales, or future projects

**Don't use for:** pre-merge review (`master-reviewer`), changelog updates that are part of the PR flow (`changelog-unreleased-workflow`), releases (`py-publish`, `psamvault-release`), or making code changes — this is verification + hygiene only.

## Project Profiles

| Project | Repo(s) | Local clone | Test command | Deploy + how to verify | Live proof | Specific skill |
|---|---|---|---|---|---|---|
| psamvault-cli | psam-717/psamvault-cli | `D:\Projects\py-projects\psamvault-cli` | `.venv/Scripts/python.exe -m pytest tests -q` (~170 tests; uv venv, Py 3.11) | none (CLI); backend repo deploy if PR touches API behavior | read-only CLI call against real backend, e.g. `whoami`; decode token `exp` for TTL PRs | changelog-unreleased-workflow |
| psam_vault_backend | psam-717/psam_vault_backend (private) | `D:\Projects\py-projects\psam_vault_backend` | none (no test suite) | Render auto-deploy: `mcp__render__list_deploys` (service `srv-d75i5pjuibrs73bocurg`) → latest deploy status **live** + commit == merge SHA | endpoint 200 / token TTL via CLI refresh | — |
| pulse | psam-717/pulse + housebuoy/pulse-mobile, pulse-web | `D:\Projects\pulse\...` | per subproject (see pulse-post-merge) | Render auto-deploy (pulse: `srv-da8a8au7bikc73c5seb0` → https://pulse-o3gj.onrender.com) | endpoint + feature artifact | pulse-post-merge |
| whatsapp-sales-agent | psam-717/whatsapp-sales-agent | `/root/whatsapp-sales-saas` | `scripts/ops/run_pr_tests.sh <N>` targeted; nightly full suite is standing catch-all | webhook → systemd restart (check `ActiveEnterTimestamp` after `mergedAt`) | endpoints `api.lenbow.com/health`, `swarmmi.lenbow.com` | whatsapp-sales-post-merge |

**Onboarding a new project:** add a row — clone path, test command, deploy mechanism + the ONE read you trust to prove deploy (deploy list status, restart timestamp, bundle hash — never the webhook 200 alone), a live proof tied to what PRs change, and any sibling skill.

## Step 1 — Confirm the merge really landed

```bash
gh pr view <N> --json state,mergedAt,mergeCommit --jq '{state, mergedAt, mergeCommit: .mergeCommit.oid}'
# expect state=MERGED + recent mergedAt
```

Never start from "trust me it merged" — verify server-side first.

## Step 2 — Sync local main

```bash
git checkout main && git pull --ff-only
```

**If pull fails or diverges, STOP and report** — never verify a merge on a stale/diverged tree. Confirm the merge commit is on top: `git log --oneline -3` and (if the PR also has a changelog PR) verify both merge commits are present.

## Step 3 — Run the project's tests

- Use the profile's test command. If the project has a targeted-run mapping for merged-PR files (whatsapp-sales `run_pr_tests.sh`), run targeted per merge and reserve the full suite for: cross-cutting PRs, multiple PRs merged close together (run ONE full suite on the combined tree), or standing nightlies.
- **Expect `N passed` — never a fixed number** (suites grow). If failures appear, check known env traps (missing `.env`, real keys, test-DB locks) before suspecting the code.
- Tests that need real credentials only run green on the machine that has them — say so instead of declaring failure.

## Step 4 — Verify the deploy/live effect (do NOT trust the trigger)

For every merge that ships to a deployed target, prove the deploy finished AFTER `mergedAt` with the right commit:

- **Render:** `mcp__render__list_deploys` for the service → newest deploy: `commit.id == merge SHA`, `status == "live"`, `finishedAt > mergedAt`. Render MCP has no session workspace state — pass `workspaceId` per request (`tea-d06ngvpr0fns73fqmlq0`). Optionally `mcp__render__list_logs` for startup errors.
- **systemd/webhook:** `systemctl show <svc> -p ActiveEnterTimestamp` after `mergedAt` + journal for the webhook/restart lines.
- **CI/deploy service:** status page/API for the exact commit.
- A successful webhook/deploy trigger is NOT proof — verify the artifact (deploy status, bundle hash, restart timestamp). Back-to-back merges can 502 the second deploy during the restart gap; GitHub auto-retries, but always verify the real artifact.

## Step 5 — Live spot check (read-only proof tied to the PR)

Pick a read-only check that would fail if the PR's change did not actually land:

```bash
curl -s -o /dev/null -w "%{http_code}\n" --max-time 10 <health_url>   # expect 200
```

- Endpoint/health 200 for deploy PRs.
- For behavior PRs, exercise the changed path once (e.g. a CLI command against the real backend, decode a JWT `exp` for TTL PRs, fetch the CSS bundle and grep a signature class for frontend PRs).
- The check must be the actual user-facing path, not a mock.

## Step 6 — Project-specific integration checks

- **Plugin/SKILL/session-text changes** (agents): restarting the gateway does NOT reset live sessions — changed SKILL/plugin text only enters context at session creation. If the PR touched identity/plugin text, run the project's session sweep + verify pre-sync sessions ended (see whatsapp-sales-post-merge Step 7).
- **Session/token rotation code** (CLIs with servers): a long-lived session can keep serving the OLD code/version after an upgrade — verify with a FRESH process, not the live session.
- **Changelog culture projects** (psamvault-cli): confirm the changelog PR merged (or run `changelog-unreleased-workflow` Flow B catch-up).
- Delegate deep-dive detail to the project's specific skill when it exists.

## Step 7 — Process hygiene (liveness is NOT enough)

"Service active" can hide orphans and crash-loops. Per platform:

- **Linux/systemd:** zombies `ps aux | awk '$8 ~ /Z/'` → 0; exactly ONE of each process; port holder == service MainPID with PPID=1 (exceptions where a child is normal — e.g. a bridge owned by a gateway); no stray background test processes holding a test DB (kill stale ones from prior sessions).
- **Windows:** check via `powershell -NoProfile -Command "Get-CimInstance Win32_Process | ..."` — no orphaned copies of the thing you restarted; old PID gone; expected process count restored.
- After restarts: record PID before, confirm the OLD PID no longer exists after — a surviving old PID means a silent crash-loop (new instance never bound).

## Step 8 — Cleanup + changelog

```bash
git branch -d <feature> <changelog-branch>      # local, after confirmed merged
git push origin --delete <feature> <changelog-branch>
```

- Delete merged branches local AND remote (only after server-side MERGED confirmed).
- **Never delete a branch on a spoken "PR merged".** Ask GitHub for the branch's own state first:
  a deleted head branch makes GitHub **CLOSE** the PR (recovery = restore the branch from its commit
  and `gh pr reopen <N>`), which is silent and easy to miss. Prefer the script
  `HERMES_HOME/scripts/prune-merged-branches.py [repo...] [--dry-run]`: it deletes only branches whose
  tip is already an ancestor of `origin/main` **and** for which `gh` reports a MERGED PR with no OPEN
  one — everything else is reported and left alone. (Lesson, 2026-09-11: a branch was deleted on the
  strength of a spoken "merged" while the PR was still OPEN, closing a mergeable PR.)
- Confirm the changelog entries landed; if not, run the catch-up flow (changelog-unreleased-workflow Flow B). **Verify by grepping the section heading in the file on main** — "changelog PR merged" is NOT proof (a changelog PR based on the code branch merges into that branch; if the code branch already merged to main, the entries strand).
- If the merge was the second PR of a pair (code then changelog), verify BOTH are merged and the content is actually on main before cleanup.

## Final report format

Verdict-first, structured markdown, tables for statuses, never a plain wall:

- Open with the plain-language verdict (`✅ PR #N verified live` / `⛔ blocked because X`)
- Table per area: merge state, tests, deploy, endpoints, hygiene, cleanup
- Bullets for one-line proofs (health codes, artifact signature)
- Notes section for anything unusual (deploy gap, warnings, session sweep)
- If anything failed, say exactly what to do next — never claim done on a plausible subset

## Common Pitfalls

1. **`git pull --ff-only` failing means STOP** — never verify on a diverged tree.
2. **Deploy/webhook 200 ≠ deployed** — verify the artifact (deploy status live + commit SHA, restart timestamp, bundle hash).
3. **Back-to-back merges** — the 2nd deploy can fail/502 in the restart gap; auto-retry usually fixes it, but verify the artifact.
4. **Fixed test counts are lies** — assert `N passed`, not a specific number.
5. **Tests needing real secrets only pass where those secrets exist** — report that, don't declare red.
6. **Long-lived sessions serve old code** — verify versions with a fresh process/session, never the live one.
7. **Plugin/SKILL text needs a session sweep, not just a restart** — context loads at session creation.
8. **Never push to main directly** — branches + PRs; even cleanup deletes use the merged remote ref.
9. **Windows ≠ POSIX process checks** — use CIM/tasklist, not `ps`/`ss`/`systemctl` idioms.
10. **Stale background processes** (test runners from prior sessions) can wedge test DBs / ports for hours — check and kill before your run.
11. **Changelog PR "merged" ≠ on main** — a changelog PR based on the code branch strands if the code branch merged first; grep the file on main for the section heading.

## Verification Checklist

- [ ] `gh pr view` shows MERGED; mergeCommit verified (both PRs if code+changelog pair)
- [ ] `git pull --ff-only` clean; merge commit on top
- [ ] Tests: `N passed` (targeted or full per project policy)
- [ ] Deploy artifact verified: status live + commit == merge SHA (or restart timestamp > mergedAt)
- [ ] Live spot check passes (endpoint / CLI / artifact signature)
- [ ] Project integration checks done (session sweep if SKILL/plugin text changed)
- [ ] Hygiene: no zombies/orphans/duplicates; old PIDs gone
- [ ] Merged branches deleted local + remote
- [ ] Changelog present or catch-up PR opened
- [ ] Verdict-first structured report delivered
