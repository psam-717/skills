# Running the CI lanes locally (and knowing when not to)

## What a lane is

`.github/scripts/run-workspace-checks.mjs` turns **every `check:*` script of the
changed package** into one CI job. `apps/desktop` declares four:

| Lane | Command | Notes |
|---|---|---|
| `check:lint` | `npm run typecheck && npm run lint` | four `tsc` passes; can V8-abort on memory — run alone, `NODE_OPTIONS=--max-old-space-size=8192` |
| `check:test:ui` | `vitest run --project ui` | ~1,239 test files — see the budget below |
| `check:test:desktop:platforms` | `vitest run --project electron` | ~375 files, ~10 min/tree, host-shaped failures on both trees |
| `check:test:desktop:all` | packages the app | ends in `dist:win:msix` — release-only toolchain |

A lane runs twice: **patched** (branch tree) and **base** (same commit, no patch).
Only the two counts together mean anything.

## Budget (measured on a 16 GB Windows laptop)

- `ui`, default pool: the worker **dies** — `Error: Worker exited unexpectedly` (host OOM), and a crashed run leaves no `Test Files / Tests` summary to quote.
- `ui`, `--no-file-parallelism`: it does not die, it does not finish — 2 of 1,239 files completed in 22 min.
- `electron` project: completes in ~10 min/tree, with host-specific failures on both trees (no display, no `zsh`, no ssh mux, no `dbus-native`).

So: never start the full `ui` lane as a parity run. Run the changed test files on
both trees, run the cheap lanes, and let CI carry the full `ui` lane — the body
says which rows came from CI and why.

## Detached runner — one lane at a time

Two suites at once OOM this box, and a subagent-owned run dies with its child.

```bash
#!/usr/bin/env bash
set -u
ROOT="<cache>/upstream-pr-<slug>"
PATCHED="$ROOT/clone-main/apps/desktop"   # branch tree
BASE="$ROOT/clone/apps/desktop"           # unpatched tree
OUT="$ROOT/parity"
mkdir -p "$OUT"; rm -f "$OUT/parity.done"
export NODE_OPTIONS="--max-old-space-size=8192"

run() {
  local name="$1" tree="$2"; shift 2
  echo "=== $name: $* ===" >> "$OUT/runner.log"
  ( cd "$tree" && "$@" > "$OUT/$name.log" 2>&1 )
  local code=$?
  echo "$name exit=$code" >> "$OUT/runner.log"
  grep -aE "Test Files|Tests  " "$OUT/$name.log" | tail -2 >> "$OUT/runner.log"
  echo "$code" > "$OUT/$name.code"
  return 0
}

run targeted-patched  "$PATCHED" npx vitest run --project ui <changed test files>
run targeted-base     "$BASE"    npx vitest run --project ui <changed test files>
run platforms-patched "$PATCHED" npm run test:desktop:platforms
run platforms-base    "$BASE"    npm run test:desktop:platforms

touch "$OUT/parity.done"
```

Launch it detached, handing the shell the script path as ONE argument:

```bash
powershell.exe -NoProfile -Command 'Start-Process -FilePath "C:\Program Files\Git\bin\bash.exe" -ArgumentList @("/c/path/run-parity.sh") -WindowStyle Hidden'
```

`-ArgumentList @('-lc','a command with spaces')` mis-parses — Start-Process joins
the array without quoting — and the job silently never starts. Verify the first
lane's log exists and is growing before telling the user anything.

## Watcher — so a long run cannot go quiet

Poll every 60 s; print on **lane transitions** (a new `exit=` line in
`runner.log`); heartbeat every ~10 min with the current lane and each log's size
and last-write; exit when `parity.done` appears.

- **`$(grep -c X f || echo 0)` yields `0\n0`**: `grep -c` prints `0` *and* exits
  `1` on no match, so the fallback appends a second `0` and every
  `[ "$n" -gt "$m" ]` test silently fails — transitions never announce while the
  heartbeat still works, which looks like a healthy watcher. Take the stdout:
  `n=$(grep -ac X f | head -1); n=${n:-0}`.
- Judge progress by log **size + last-write time**, not by the log existing: a
  lane that is alive but glacial is otherwise indistinguishable from a hung one.
- The user will ask "what is running now" — answer with the current lane, the
  log path, the last write, and how many lanes remain, not "still running".

## Stopping a run that went glacial

- **The runner's wrapper bash is not the lane.** Killing it leaves the vite/vitest
  `node` child grinding against the tree, so the next lane competes with it.
  Match by name AND a path unique to the job, then verify the set is empty:
  `Get-CimInstance Win32_Process | Where-Object { $_.Name -eq 'node.exe' -and $_.CommandLine -like '*<cache>/upstream-pr-*' }`.
- **Never match on a bare word you also typed in the kill command.** The agent's
  own shell appears in a `CommandLine -like '*run-parity*'` filter — its wrapper
  embeds the command text — so the filter kills the caller's own shell: the call
  comes back with exit `-1` and every check after the kill silently never runs.
  Keep the tool's `-c "source …"` wrapper shells out of the match set.
- A lane killed mid-run writes no `Test Files / Tests` summary, so its log proves
  nothing: record it as *not run*, never as failed, and re-run it in the smaller
  shape instead.

## Reading and comparing a run

- Per-lane summary: `grep -aE 'Test Files|Tests  ' "$OUT/$name.log" | tail -2`.
- Failing-file SETS, then the delta — this is the whole comparison:
  ```bash
  grep -aE '^ (❯|✗) \|ui\|' "$OUT/ui-patched.log" | sed 's/([0-9].*//' | sort -u > patched.txt
  grep -aE '^ (❯|✗) \|ui\|' "$OUT/ui-base.log"    | sed 's/([0-9].*//' | sort -u > base.txt
  comm -23 patched.txt base.txt   # only in patched = the only thing that is yours
  ```
  One file is the typical delta, and it is usually a real gap in your own change
  (a generated catalog that was not regenerated, a new key the completeness test
  cannot see). Fix it; do not explain it away in the body.
- Quote **both** counts plus the delta file. Never one count alone, and never the
  exit code alone — a lane exits non-zero on pre-existing host failures.
- **Name the base tree's SHA, and say how it relates to the branch base.** The
  reuse of a dead session's workspace usually leaves `clone` at whatever main the
  MANAGED checkout sat on — often thousands of commits behind the branch's own
  base — so the two trees differ by upstream drift as well as by your patch. State
  both SHAs in the body and read the delta direction: a failure that exists ONLY
  on the older base tree is drift you inherited fixed, while a failure only on the
  patched tree is the only one worth chasing. When the failing-file delta is empty
  against an older base, the claim is *stronger* than parity, not weaker.
- **Recognise the packaging lane's environmental failure.** `check:test:desktop:all`
  routinely dies before any assertion with `Installed app-builder-lib is not the
  source's lock-pinned package; prepare Node dependencies first` (or the
  `assert-root-install` preflight): the clone JUNCTIONS the managed install's
  `node_modules`, whose versions predate the branch's lockfile. Run it on both
  trees, quote the identical error, and name the cause — only a standalone
  `npm ci` clone can actually measure that lane locally.
