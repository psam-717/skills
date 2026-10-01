---
name: wps-ai-hermes-bridge
description: Use when wiring Hermes to WPS Office live docs via MCP.
version: 1.0.0
---

# WPS-AI ↔ Hermes Bridge (MCP)

Hermes can read/edit LIVE open documents in WPS Office (Writer/ET/Presentation/PDF) through the Lingxi AI plugin's MCP server.

## Architecture
```
Hermes (mcp_servers.wps_ai) → node tools/mcp-server.js (stdio JSON-RPC)
  → HTTP proxy-server.js :3890 → plugin TaskPane in WPS → WPS JSAPI → live doc
```

## Setup (completed on psam's machine, 2026-09-29)
- Repo clone: D:/Projects/py-projects/WPS-AI (bundled Node: plugin/runtime/node-win-x64/node.exe — system has NO node)
- Host variants built: `node tools/build-variants.js --out ~/.lingxi-ai` (creates plugin-wps/-et/-wpp/-pdf)
- CRITICAL: WPS is the Microsoft STORE edition → AppData is virtualized. publish.xml must go to:
  `C:/Users/psam/AppData/Local/Packages/ZhuhaiKingsoftOfficeSoftw.WPSOffice2019_924xes6e8q1tw/LocalCache/Roaming/Kingsoft/wps/jsaddons/publish.xml`
  (normal %APPDATA%/kingsoft path is ignored by Store WPS)
- Serve plugin: `node plugin/tools/serve-permanent.js --root C:/Users/psam/.lingxi-ai` (ports 3889 static + 3890 proxy; run from repo dir so node_modules resolve — do NOT copy serve-permanent.js out alone, it requires pick-node.js etc.)
- Plugin MCP toggle must be ON inside WPS Lingxi AI panel (Settings → MCP 服务) — tools/list returns nothing until WPS is open with the plugin loaded
- Hermes config: mcp_servers.wps_ai → command = bundled node.exe, args = [D:/Projects/py-projects/WPS-AI/plugin/tools/mcp-server.js]. Set via `hermes config set`, then RESTART GATEWAY (MCP discovery is startup-only)
- Verify: get_host_info returns host + active document name

## Usage
- ~161 tools load as mcp__wps_ai__* (wps_read_document, wps_find_replace, wps_insert_text, et_read_range, ...)
- Read before claiming edits landed: wps_read_document(mode=text, offset/maxChars for paging)
- wps_find_replace(find, replace) for surgical fixes; wps_replace_document nukes all content — avoid
- reveal_location scrolls/highlights in the UI so the user can watch
- Editing open docs goes ONLY through MCP; closed files use the docx skill (python-docx)

## Phase 2: panel → Hermes API (completed 2026-10-01)
- Gateway runs MULTIPLEXED (default profile serves default+preflight+prometheus): API server lives on the DEFAULT profile's .env (C:/Users/psam/AppData/Local/hermes/.env: API_SERVER_ENABLED/_KEY/_CORS_ORIGINS). Vars in a secondary profile's .env are IGNORED and can trip MultiplexConfigError.
- API server: http://127.0.0.1:8642/v1, model name echo only — ALL requests run on the DEFAULT profile's agent core; model:"prometheus" does NOT route to the prometheus profile in this build (873 commits behind upstream; per-profile routing arrived later).
- WPS panel provider config: Lingxi AI settings → add Custom OpenAI-compatible provider, baseURL http://127.0.0.1:8642/v1, any key = API_SERVER_KEY, model hermes-agent. Panel chat then drives the live doc via its own MCP tools.
- Default profile (Atlas) was switched to z-ai/glm-5.3 @ openrouter (was deepseek-v4-flash, unresponsive) and given mcp_servers.wps_ai too — panel needs the tools on the profile that serves api_server.
- Verify runs in %LOCALAPPDATA%/hermes/logs/agent.log: session api-*, look for mcp__wps_ai__* tool completions.

## Gotchas
- tools/list silent until WPS open + plugin loaded + MCP enabled
- Store WPS = virtualized paths (see setup)
- Unsaved docs: plugin fail-fasts mutations on unsaved documents
- Autostart of serve-permanent.js deliberately NOT registered (user choice, 2026-09-29)
- Tool names arrive underscored: mcp__wps_ai__wps_read_document
