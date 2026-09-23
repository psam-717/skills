---
name: hermes-upstream-pr
description: "Upstream a hermes-agent fix from the live checkout as a PR."
version: 1.0
---

# Hermes Upstream PR

Ship a bug fix developed against the managed install (`$HERMES_HOME/hermes-agent`) to the upstream repo as a PR — patch in place, prove in an isolated clone, push from a fork. Use whenever the fix touches the hermes-agent repo and the user wants it upstreamed (or a local patch that must survive cleanly).

## Always-on rules

- **Read CONTRIBUTING.md + `.github/PULL_REQUEST_TEMPLATE.md` in the repo first** and follow them literally: branch `fix/<desc>`, conventional commit (`fix(desktop): ...`), template sections filled, checklist honest.
- **Prove new invariant tests red on base** — apply the test diff to a clean `origin/main` clone WITHOUT the source fix, run, show failure; then apply the source diff and show green. State both in the PR body.
- **Never branch or stash in the live checkout** — the install guard blocks `git stash`/`git checkout` there (mixing module versions in the running runtime). Edit files freely; do git surgery in the scratch clone.
- **Scope commits by path** (`git add apps/desktop`), never `git add -A` — the live checkout carries unrelated dirty files (local patches) that must not ride along in the PR.

## Procedure

0. **Diagnose on live state first** (desktop-app bugs):
   - The PACKAGED app accepts Chromium's own flag — relaunch it with
     `Start-Process '…\release\win-unpacked\Hermes.exe' -ArgumentList
     '--remote-debugging-port=9222'` (only when the restart is yours to make;
     tell the user) to get a CDP port; relaunch without the flag when done.
   - Reproduce signal-driven reconnect/re-render bugs without a human at the
     window: fire synthetic event storms from `Runtime.evaluate` —
     `window.dispatchEvent(new Event('online'))`, `focus`, and
     `document.dispatchEvent(new Event('visibilitychange'))` in a loop.
   - Churn metrics that prove the bug and the fix: `grep -c "ws closed"
     <profile>/logs/gui.log` per profile (the sick profile has orders of
     magnitude more), and `session.reclaimed|ws_orphan_reap` counts in
     `%LOCALAPPDATA%\hermes\logs\desktop.log`. Report before/after numbers.
   - A backgrounded Electron window throttles renderer timers hard — a
     starved in-page sampler means "not focused", not "idle".
1. **Develop against the live checkout** (edits are fine): patch, `npm run typecheck` (in `apps/desktop` for TS), targeted `npx vitest run <files>`, then the full `npm run test:ui`.
2. **Format before committing**: `npx eslint --fix <changed.ts>` + `npx prettier --check <changed files>` — warnings block a clean PR.
3. **Split the diff**: `git diff -- <source files> > src.patch` and `git diff -- <test files> > tests.patch` (both scoped to the changed subdirectory, excluding unrelated dirty files).
4. **Scratch clone** (shared, real disk — the profile scratch dir, never /tmp):
   ```bash
   git clone --shared "$HERMES_HOME/hermes-agent" <scratch>/clone
   cd <scratch>/clone && git fetch https://github.com/NousResearch/hermes-agent.git main --quiet
   git branch fix/<desc> FETCH_HEAD && git checkout fix/<desc>
   ```
   Do the `git checkout` in a SEPARATE terminal call from any command mentioning the live checkout path — the guard pattern-matches the command line, not the cwd.
5. **Junction node_modules** (no npm install needed) with PowerShell, at BOTH levels:
   ```powershell
   New-Item -ItemType Junction -Path <clone>\node_modules -Target <live>\node_modules
   New-Item -ItemType Junction -Path <clone>\apps\desktop\node_modules -Target <live>\apps\desktop\node_modules
   ```
6. **Red-on-base proof**: `git apply tests.patch` → run the new tests → they must FAIL. Then `git apply src.patch` → run the full targeted suite → green.
7. **Commit** with conventional message + `Fixes #<issue>` in the body, push to a fork (`gh repo fork NousResearch/hermes-agent --clone=false`, `git remote add fork`, push branch).
8. **Open the PR** with `gh pr create --repo NousResearch/hermes-agent --base main --head <user>:<branch> --body-file`. Account has READ on upstream — fork flow is mandatory.
9. **Verify the PR** with `gh pr view` (state, additions/deletions, file list) before reporting the URL.
10. **Review follow-ups**: automated AI reviewers comment on PRs and cite real line-level gaps — read them, address each point with a follow-up commit plus a reply comment explaining the reasoning (including deliberate simplifications). The clone branch already carries the first commit, so COPY the patched files from the live tree into the clone instead of re-applying cumulative patches, and prove the new test red on the PREVIOUS commit before the fix (`git diff -- <src> > fix.patch && git checkout -- <src>` → run new test → RED → `git apply fix.patch` → GREEN).
11. **Rebase round** (when a reviewer says "current main added X after this branch's base" or asks you to rebase):
    - Check what actually changed on main for YOUR files before deciding effort — fetch the file blobs from main via the GitHub API (`contents?ref=main` → base64) and diff against the branch base in Python; the compare API's files list caps at 300 entries on a fast-moving repo and ABSENCE THERE IS NOT EVIDENCE a file is unchanged.
    - Fetching current main into the `--shared` clone can exceed the 180s terminal default — run it in background (`git fetch <upstream> main:upstream-main`) and verify the ref exists before using it.
    - `git checkout -b fix/<desc>-v2 upstream-main && git cherry-pick <sha1> <sha2> …` — resolve the small conflicts, run the targeted suite, then implement the reviewer's ask.
    - When main now HAS the policy/module the reviewer wants mirrored (e.g. a shared liveness/decision module), REUSE it verbatim — import the module, don't re-implement the logic — so a later rebase is trivial and the semantics match upstream by construction.
    - Force-push the v2 branch to the SAME head branch name (`git push fork v2:fix/<desc> --force`) — the PR and its review threads survive; old threads mark outdated, which is expected. Update the PR body (`gh pr edit --body-file`) and reply on each review thread (`gh api repos/<org>/<repo>/pulls/<n>/comments/<id>/replies -f body=…`).
12. **Local build**: for desktop changes, `npm run pack` in `apps/desktop` after killing Hermes.exe (packager keeps a rollback at `release/win-unpacked.bak`). Relaunch and verify live.
13. **After the merge** (maintainers cherry-pick external PRs with authorship intact and close the original in favour of the merged one — verify, don't assume):
    - Confirm your commits on `main` are attributed to the user: `gh api repos/<org>/<repo>/commits/<sha> --jq '{author: .author.login, name: .commit.author.name}'`.
    - The GitHub Contributors graph/cache lags up to ~24h after a merge — absence there is not evidence of lost attribution. Point the user at `…/commits?author=<user>` (works immediately) and their profile's activity feed, and tell them the graph catches up.
14. **Teardown once merged** — safe-delete the scratch clone, junctions FIRST:
    - Delete the two node_modules junctions with PowerShell `[IO.Directory]::Delete('<path>')` (removes only the reparse point) BEFORE any recursive delete — MSYS `rm -rf` FOLLOWS junctions and would wipe the live install's `node_modules`. `cmd //c rmdir` from git-bash silently fails on them.
    - `rm -rf <scratch>` from the profile scratch dir hits `Device or resource busy` while a shell's cwd sits inside the tree — `cd $HOME` first; if an EMPTY dir still won't die, `powershell Remove-Item -Recurse -Force` it. Verify the live install's `node_modules` still lists afterward.
    - Delete the fork's merged head branch: `gh api -X DELETE repos/<user>/<repo>/git/refs/heads/<branch>` (merged commits are safe on upstream).

## Pitfalls
- Whole-file formatters are PR noise: if `prettier --check`/`eslint --fix` touch hunks OUTSIDE your edit regions, first run the check against the HEAD copy of the file — if the committed file already fails it, CI does not enforce it there; rebuild the file (`git checkout -- <file>`, re-apply only your scoped edits) instead of shipping the formatter's unrelated rewrites.
- Never pipe a full suite through `tail` — the failure summary is eaten and you must re-run 15 minutes of tests. Redirect to a file (`npx vitest run --project ui > log 2>&1; grep -E 'Test Files|Tests  |FAIL' log`). vitest v4 has no `--reporter=basic` — it's a startup error; use the default reporter with file redirection.
- Compare lint/prettier results against the HEAD baseline before claiming cleanliness — run the linter on the stashed HEAD copy and diff the counts; only NEW warnings block the PR.
- `cmd //c mklink /J` through git-bash prints a banner and silently creates nothing — use PowerShell `New-Item -ItemType Junction`. Verify the junction exists before running tests.
- Pre-existing test failures in the full suite: rerun the failing file in isolation, then confirm on the clean clone WITHOUT your patches before claiming them in the PR body — "reproduces identically on unpatched main, unrelated to this change" is the honest phrasing reviewers expect.
- Changing shared timing contracts (e.g. adding a grace window to a pruner) breaks every test that asserts immediate disposal — age the fixture past the window (`vi.useFakeTimers({ now: Date.now() + N }); prune(); vi.useRealTimers()`) instead of weakening the assertion; spare-assertions are unaffected by aging.
- Stamp timestamps where the operation completes in the real code path (e.g. after the awaited dial), not in event handlers — the vitest gateway mocks never fire `onState` callbacks, so handler-stamped state is invisible to tests.
- `JsonRpcGatewayError` is constructed `(message, { code, data })` — NOT `(code, message)`. The wrong arg order still passes `instanceof` but never matches the code you assert on, closing a test's "healthy backend" case by accident.
- Per the repo's test rules: ≤2 invariant tests per fix, named by behavior, never change-detectors; when a test harness file already covers the area, extend it rather than creating a near-duplicate file.
- To assert in-flight-work behavior against a gateway mock, hold the entry's request lease via the EXPORTED request path (`requestGatewayForAgent`-style) with a never-settling `socket.request = vi.fn(() => new Promise(() => {}))`, `vi.waitFor` until it dispatched, then SWAP `socket.request` for the probe/timeout mock — the lease stays held while the new mock serves only your probe calls.
- Keep the scratch clone until the PR merges (CI feedback may need a follow-up push); delete it after.
