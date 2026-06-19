---
name: psamvault-mcp
description: "Use psamvault MCP server: credential vault, browser login, API key injection, .env secret protection, and CLI command injection with credentials. MCP tools keep secrets out of the agent's context window."
version: 1.1.0
author: psam-717
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [mcp, credentials, security, vault, browser-login, pypi, cli]
    related_skills: [native-mcp, mcp-tool-integration-skill, mcp-server-diagnostics]
---

# psamvault-mcp

## Overview

[psamvault-mcp](https://pypi.org/project/psamvault-mcp/) is an MCP server for [psamvault](https://pypi.org/project/psamvault/) — a zero-knowledge credential vault for AI agents. It lets agents use stored credentials (site passwords, API keys, tokens) without ever seeing their plaintext values.

The server provides four complementary flows:

- **`browser_login`** — Opens a real Chromium browser, navigates to the site, fills credentials directly in the browser process. The agent only sees whether login succeeded.
- **`use_credential`** — Makes authenticated HTTP requests using stored API keys. Credential is decrypted locally, the HTTP request is made, and only the response is returned.
- **`run_with_credential`** — Runs a shell command with a credential injected as an environment variable or stdin. Output is auto-redacted of the credential value.
- **`scan_and_protect`** — Scans a project directory for `.env` files, detects secrets using pattern matching, encrypts them into the psamvault vault, and replaces plaintext with `psamvault:KEY_NAME` placeholders.

This is the **key differentiator**: credentials are never in the agent's context window, never in training data, and never accidentally leaked in a prompt.

## When to Use

- User asks you to interact with psamvault-mcp tools
- User wants to log into a website via agent (`browser_login`)
- User needs authenticated API calls via agent (`use_credential`)
- User needs to run CLI commands with credentials (twine upload, docker login, git push — `run_with_credential`)
- User wants to secure their project's `.env` files (`scan_and_protect`)
- Any task involving credential lookup or use — prefer MCP tools over terminal-based CLI
- **Do not use for**: reading credentials directly (always use `use_credential` instead)

## Prerequisites

The MCP server must be running and connected. Hermes connects via **stdio transport** (configured in `config.yaml`):

### Setup (for Hermes)

The Hermes config already has this entry — it's active:

```yaml
mcp_servers:
  psamvault:
    command: psamvault-mcp
    enabled: true
    env:
      PYTHONPATH: ""
```

No manual start needed — Hermes manages the process lifecycle.

### Installation

```bash
pipx install psamvault-mcp
playwright install chromium
```

The user must also have the psamvault CLI installed and logged in:

```bash
pipx install psamvault
psamvault configure
psamvault login
```

### HTTP/SSE alternative (for non-Hermes clients)

For clients that don't support stdio MCP transport:

```bash
psamvault-mcp --http --port 8433
```

## Available Tools

Once connected, these tools become available (prefixed `mcp_psamvault_*` in Hermes):

| Tool | Parameters | Description | Security |
|------|-----------|-------------|----------|
| `get_version` | None | Return installed version | No secrets involved |
| `search_vault_tools` | `query: str` | Discover which tool to use (call this first) | No secrets involved |
| `list_vault_sites` | None | List stored credential sites | Returns names only, no passwords |
| `list_api_keys` | `project_name: str (optional)` | List stored API key names with service hints | Returns names only, no key values |
| `check_credential_exists` | `site_name: str` | Check if credential exists | Returns boolean + username hint |
| `get_username_for_site` | `site_name: str` | Get stored username | Returns username, never password |
| `use_credential` | `site_name, target_url, method, inject_as, fields, header_name, body, extra_headers` | Make authenticated HTTP request | ❗ Credential stays server-side |
| `browser_login` | `site_name, login_url, selectors, timeout_ms` | Log into website via browser | ❗ Fills directly in browser |
| `run_with_credential` | `site_name, command, inject_as, env_var_name, extra_env, workdir, timeout` | Run CLI command with credential injected | ❗ Output redacted of credential |
| `scan_and_protect` | `project_dir, patterns, project_name` | Scan & encrypt .env secrets | ❗ Encrypts into vault |
| `capture_stripe_credentials` | `provider, project_dir, dry_run` | Capture Stripe provisioned creds | ❗ Encrypts into vault |

## Workflows

### 1. Browser Login

When the user asks "Log me into X":

1. **Check** the credential exists:
   ```
   check_credential_exists(site_name="github.com")
   ```
2. **Call** browser_login:
   ```
   browser_login(site_name="github.com")
   ```
3. **Relay** the result — always include the `message` field from the response verbatim to the user
4. **Handle CAPTCHA**: If `captcha_detected` is `true`, tell the user the browser is paused and they need to solve the CAPTCHA manually

### 2. API Credential Usage

When the user needs an authenticated API call:

1. **Check** the credential:
   ```
   check_credential_exists(site_name="github.com")
   ```
2. **Call** use_credential:
   ```
   use_credential(
       site_name="github.com",
       target_url="https://api.github.com/user",
       inject_as="bearer_token"
   )
   ```
3. **Use the `fields` parameter** to return only needed data (reduces tokens):
   ```
   use_credential(
       site_name="github.com",
       target_url="https://api.github.com/user/repos",
       inject_as="bearer_token",
       fields=["name", "html_url", "language"]
   )
   ```

Supported injection modes:
- `bearer_token` — `Authorization: Bearer ***`
- `api_key_header` — Custom header name with API key value
- `basic_auth` — `Authorization: Basic base64(user:pass)`

### 3. CLI Command with Credential (run_with_credential)

When the user needs to run a CLI tool that requires credentials:

1. **Check** what credential to use:
   ```
   list_api_keys()  # or list_vault_sites()
   ```
2. **Call** run_with_credential:
   ```
   # Publish to PyPI
   run_with_credential(
       site_name="pypi",
       command="twine upload dist/*",
       inject_as="env",
       env_var_name="TWINE_PASSWORD"
   )

   # Docker login
   run_with_credential(
       site_name="dockerhub",
       command="docker login -u myuser --password-stdin",
       inject_as="stdin"
   )

   # Push to private repo
   run_with_credential(
       site_name="github-api",
       command="git push origin main",
       inject_as="env",
       env_var_name="GITHUB_TOKEN"
   )

   # npm publish
   run_with_credential(
       site_name="npm",
       command="npm publish",
       inject_as="env",
       env_var_name="NPM_TOKEN"
   )
   ```

3. **The credential value NEVER enters your context** — all output is scanned and redacted before being returned.

Supported injection modes:
- `env` — Set credential as an environment variable. Requires `env_var_name`.
- `stdin` — Pipe credential as stdin to the process.

Auto-convenience: When `env_var_name="TWINE_PASSWORD"`, the tool automatically sets `TWINE_USERNAME=__token__`.

### 4. Protecting .env Files

When the user wants to secure project secrets:

1. **Call** scan_and_protect:
   ```
   scan_and_protect(project_dir="/path/to/project")
   ```
2. **For cleaner per-project organisation**, pass a `project_name`:
   ```
   scan_and_protect(project_dir="/path/to/project", project_name="myapp")
   ```
   Keys are stored as `myapp/.env/KEY_NAME` — filterable later with `list_api_keys(project_name="myapp")`.
3. **Inform** the user what was found and encrypted
4. **Recommend** [pv-dotenv](https://pypi.org/project/pv-dotenv/) for runtime resolution:
   ```
   pip install pv-dotenv
   # Then replace:
   from dotenv import load_dotenv  # → from pv_dotenv import load_dotenv
   ```
5. **Nothing else needed** — the `.env` file now has `psamvault:KEY` placeholders and pv-dotenv resolves them at runtime

### 5. Stripe Projects Integration

When the user runs `stripe projects add <provider>`:

1. **Call** capture_stripe_credentials:
   ```
   capture_stripe_credentials(provider="neon", project_dir="/path")
   ```
2. **Optionally dry-run first**:
   ```
   capture_stripe_credentials(provider="neon", dry_run=True)
   ```

### 6. Vault Discovery

When the user asks "What credentials do I have?":

1. **Call** `list_vault_sites()` to see all stored credential sites
2. **Call** `list_api_keys()` to see all stored API keys
3. **For each site**, optionally `get_username_for_site(site_name)` to show usernames
4. Present the information as a clean list to the user

## Common Agent Prompts

Tell users they can say things like:

- "What credentials do I have stored?"
- "Log me into github.com"
- "Check my GitHub profile info"
- "Get my top 10 starred repositories"
- "What API keys do I have stored?"
- "Upload my package to PyPI"
- "Docker login"
- "Push to my private repo"
- "Protect the secrets in my project directory"
- "Check if I have a credential for z.ai"
- "What's my username for Kaggle?"

## Architecture Notes

- **Single-process Playwright**: The browser lives in-process with the MCP server. No subprocess daemon chain. If the browser crashes, the next call auto-restarts it.
- **Stdio transport (default)**: Hermes manages the process lifecycle. No manual server start needed.
- **OS keychain auth**: The VEK (Vault Encryption Key) is stored in the OS keychain by the psamvault CLI at login time.
- **No consent dialog**: The v0.4.0+ architecture removed the consent dialog requirement. Credentials are used on demand.
- **Output redaction**: `run_with_credential` scans all output for the credential value and redacts it before returning to the agent.

## Security Rules

| Rule | Rationale |
|------|-----------|
| **Always use MCP tools** for credential ops | Never read `~/.psamvault/` files or env vars directly |
| **Never print raw credentials** | Keeps secrets out of context window and transcript |
| **Never ask user to paste credentials** | Use `browser_login`, `use_credential`, or `run_with_credential` instead |
| **Use `fields` parameter** | Reduces token usage and avoids returning unnecessary data |
| **Relay `message` field verbatim** | Browser login messages contain user-facing instructions |
| **Never run `psamvault` CLI commands** | Always use MCP tools for credential access |

## Common Pitfalls

1. **Forgetting to call `search_vault_tools("")` first** — always discover available tools before assuming what's available
2. **Not checking `captcha_detected`** — if a CAPTCHA appears, the browser pauses. You must inform the user to solve it and click Sign in.
3. **Calling wrong injection mode** — `api_key_header` needs a `header_name` parameter; `basic_auth` uses username:password; `bearer_token` uses the password as the token
4. **Not using `fields` on large API responses** — a full GitHub user response is ~40 fields. Use `fields=["login","id","public_repos"]` to return only what's needed.
5. **Mixing up site name vs API key name** — `use_credential` checks API key entries first, then vault password entries. If you need an API key but stored it as a site password, it still works as fallback.
6. **Server not running** — MCP tools return connection errors if the server is not started. If using stdio, Hermes starts it automatically on connect. If using HTTP/SSE, start with `psamvault-mcp --http --port 8433` in a terminal.
7. **Token expiry** — The access token has ~1 hour lifetime. psamvault-mcp auto-refreshes via `api_client.py`, but if the server has been running for hours without use, the initial call might need a refresh cycle. The `api_client` handles 401 → refresh → retry automatically.

## Related Projects

| Package | What It Does |
|---------|-------------|
| [`psamvault`](https://pypi.org/project/psamvault/) | CLI — store, list, manage credentials |
| [`psamvault-mcp`](https://pypi.org/project/psamvault-mcp/) | MCP server for AI agents |
| [`pv-dotenv`](https://pypi.org/project/pv-dotenv/) | Drop-in python-dotenv replacement — resolves `psamvault:` placeholders |
| [`psamvault-cli`](https://github.com/psam-717/psamvault-cli) | CLI + vault management |

## Verification Checklist

- [ ] MCP server is connected (stdio, managed by Hermes automatically)
- [ ] `search_vault_tools("")` returns the list of all 11 tools
- [ ] `get_version()` returns a version string
- [ ] `list_vault_sites()` returns stored sites (or empty list)
- [ ] `list_api_keys()` returns stored API keys (or empty list)
- [ ] `check_credential_exists("github.com")` returns expected result
- [ ] `use_credential` makes authenticated HTTP calls successfully
- [ ] `run_with_credential` runs CLI commands with credential injection
