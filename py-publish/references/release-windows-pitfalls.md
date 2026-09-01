# Windows release pitfalls (build → publish on Windows)

Windows + git-bash (MSYS) behaves differently from POSIX shells at several
points in the publish pipeline. These are the traps that actually bite.

## 1. Setuptools/_distutils (SRE) mismatch breaks builds

Symptom: `python -m build` (or `pip install -e .`) dies with
`ModuleNotFoundError: No module named 'setuptools._distutils'` or
`module 'setuptools' has no attribute 'Distribution'` — a version skew
between the `setuptools` and `setuptools._distutils` (SRE) layers in the
active environment.

Fix (in the project venv):

```bash
uv pip install --upgrade setuptools wheel
python -m build
```

Or bypass the environment entirely with isolation: `uv build` (uv builds in
an isolated env by default, immune to the host setuptools skew).

## 2. `uv tool install` replaces pipx on Windows

pipx works, but uv is the project's package manager — use it consistently:

```bash
uv tool install psamvault          # installs a CLI globally
uv tool upgrade psamvault
```

## 3. `taskkill` in MSYS/git-bash needs doubled slashes

`taskkill /PID 1234 /F` in git-bash: MSYS eats the leading slash, so use
`taskkill //PID 1234 //F`. Or skip the translation entirely:

```bash
powershell -NoProfile -Command "Stop-Process -Id 1234 -Force"
```

**Never** `taskkill //IM python.exe //F` — that kills every python process
on the machine (including the Hermes desktop backend and gateway). Target
by PID only, or filter by command line first:

```bash
powershell -NoProfile -Command "Get-CimInstance Win32_Process -Filter \"Name='python.exe'\" | Where-Object { \$_.CommandLine -match 'twine upload' } | ForEach-Object { Stop-Process -Id \$_.ProcessId -Force }"
```

## 4. Editable installs for dev

```bash
uv pip install -e .        # preferred — modern, path-stable
```

Plain `pip install -e .` on Windows can fall back to legacy setuptools
editable installs that break when the project path moves or contains
spaces. If a stale editable install shadows the real package, reinstall
with `--force-reinstall`.

## 5. Interpreter naming

- Windows cmd/PowerShell: `python` (not `python3`).
- git-bash: `python` resolves too, but it may be a *different* interpreter
  than the one on PATH in cmd. Always invoke build/publish through the
  project venv explicitly: `.venv/Scripts/python.exe -m build`,
  `.venv/Scripts/python.exe -m twine upload dist/*`.

## 6. Misc

- `rm -rf dist/*` works fine in git-bash; in cmd use `del /q dist\*` — or
  just let `python -m build` overwrite.
- Antivirus/Defender can hold freshly built `dist/` files open briefly —
  if an upload races a locked file, wait a second and retry; do not
  "clean" by deleting the whole dist tree and rebuilding from scratch
  unless the build itself was dirty.
- Paths with spaces: quote them; MSYS does NOT auto-translate path
  arguments for native tools (`uv run --env-file` is the one exception
  that handles it via its own arg parser).
