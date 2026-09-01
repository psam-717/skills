# Publishing with psamvault-stored PyPI tokens (zero-knowledge)

psamvault (https://github.com/psam-717/psamvault-cli) stores PyPI tokens as
encrypted API-key entries. Tokens are decrypted locally at publish time and
fed to twine — **no `~/.pypirc` file ever sits on disk**, and the raw token
never enters the agent's context if you keep it in an env file.

## Prerequisites

```bash
psamvault login        # session + VEK live in the OS keychain
                       # (~/.psamvault/session.json is only a presence marker)
psamvault ak-list      # confirm 'pypi' and 'testpypi' entries exist
```

If an entry is missing:

```bash
psamvault ak-add pypi --service PyPI        # prompts for the token
psamvault ak-add testpypi --service TestPyPI
```

## Fetch the token at publish time

```bash
psamvault ak-get pypi            # prints the decrypted token
psamvault ak-get pypi --copy     # or copy straight to clipboard
```

## Inject the token into twine — env-file method (recommended)

Never `echo`/`export` the token in a shell: PyPI tokens frequently contain
`***` sequences that bash expands as globs, silently truncating the token
(the classic `403 Forbidden` / `Invalid or non-existent authentication
information` twine failure). Write an env file instead — use the agent's
`write_file` tool, NOT a shell heredoc:

```
# File: <temp>/twine_pypi.env
TWINE_USERNAME=__token__
TWINE_PASSWORD=<full token from psamvault ak-get>
```

Then upload with `--env-file` so nothing leaks into the shell environment:

```bash
# TestPyPI first (note --extra-index-url: TestPyPI does NOT mirror PyPI's
# dependency index, so transitive deps must fall back to pypi.org)
python -m twine upload --repository testpypi dist/*   # with testpypi env file

# Real PyPI, once the user has approved:
python -m twine upload dist/*                         # with pypi env file
```

Clean up immediately after:

```bash
rm -f <temp>/twine_pypi.env <temp>/twine_testpypi.env
```

## Verification / failure modes

- **`403 Forbidden` or "Invalid or non-existent authentication information"**
  → the token was truncated. Confirm what actually reached twine:
  ```bash
  set -a; source <temp>/twine.env 2>/dev/null; echo "length: ${#TWINE_PASSWORD}"
  ```
  If the length is ~3 or looks clipped, re-fetch with `psamvault ak-get` and
  rewrite the env file. Double-check the token matches
  `https://pypi.org/manage/account/token/`.
- **TestPyPI upload works, PyPI fails** → likely the wrong env file was used
  (testpypi token vs pypi token). psamvault stores them as separate entries.
- **`ak-get` empty or errors** → re-run `psamvault login` (session expired).

## psamvault-cli specific note

psamvault-cli ships `scripts/release.py` (reads the version from
pyproject.toml + changelog.py, creates the GitHub release with built
artifacts attached). The publish sequence for that repo is:
`python -m build` → twine upload (env-file above) → `python scripts/release.py`.
