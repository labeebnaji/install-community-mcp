---
name: install-community-mcp
description: Install and register a community (non-official) MCP server, typically a Python/uv GitHub repo, on Windows in Qoder. Use when the user asks to install, add, or connect an MCP server found on GitHub, or when the official extension market returns no match for a needed tool. Covers repo auditing, uv/uvx pitfalls, network fallback ladders, registration in the Qoder settings file, and honest verification.
---

# Install Community MCP Server (Windows)

## Overview

End-to-end playbook for vetting, testing, installing, and registering a third-party MCP server (usually Python + uv) so its tools appear in Qoder. Derived from a full install cycle of openags/paper-search-mcp, including the failures that shaped the fallback ladder.

**Hard rules:**
- Never execute community code before the audit and explicit user approval (step 2).
- Never declare success until the server's tools appear in an MCP catalog. A registration in the settings file is not proof it loaded.

## Prerequisites

- Windows, Qoder CLI.
- Python 3.10+ on PATH, or `uv` available anywhere on disk (see step 3 for locating it).
- Network access to GitHub and to a Python package index.

## Step 1 — Discover

1. Search the official extension market first (`mcp__extension-market__search_extensions`). An official entry is always preferable — install it and skip the rest of this playbook. Note: the keyword filter is unreliable and may return an unfiltered general list; scan the results yourself rather than trusting an empty-looking match.
2. For community candidates, find the repo via web search with the exact project name plus "github mcp". Never guess `owner/repo` — a guessed owner returns a 404 and wasted retries.
3. Collect 2-4 candidates and compare: stars, last activity, scope, runtime.

## Step 2 — Audit the repo (mandatory, before any execution)

Fetch `README.md` and `pyproject.toml` from `https://raw.githubusercontent.com/OWNER/REPO/BRANCH/...` (more reliable than rendering the GitHub page). Extract:

- Entry points from `[project.scripts]` in pyproject.toml — the console-script name is what uv/pip will launch.
- Dependencies — flag anything obscure; standard PyPI packages (requests, fastmcp, mcp[cli], lxml, pydantic) are fine.
- Required environment variables (API keys, contact emails) — ask the user before proceeding if any are mandatory.
- Security signals: does the README state network transports bind to `127.0.0.1`? Any telemetry? License? Star count as a maturity proxy.
- The MCP config shape the README advertises (usually `command: uvx` plus args) — note it, then adapt per step 5.

Present a short audit report (tools exposed, dependencies, keys, security) and get explicit user approval to install.

## Step 3 — Probe the runtime

Do not install uv blindly first. Check what already exists:

```bash
where.exe uv 2>/dev/null; where.exe uvx 2>/dev/null
ls "$USERPROFILE/uv/" 2>/dev/null
python --version 2>/dev/null || py -3 --version
```

Facts observed on real Windows setups:
- Installer scripts can leave `uv.exe` present (for example under `%USERPROFILE%\uv\`) while `uvx.exe` is missing. `uvx PKG` is equivalent to `uv tool run PKG`; `uv tool run --from git+REPO ENTRY` replaces any README recipe that uses uvx.
- A system Python (3.10+) can serve as the pip fallback runtime from step 4.

Smoke-test the server before registering anything:

```bash
UV_HTTP_TIMEOUT=300 "$UV_PATH/uv.exe" tool run --from git+https://github.com/OWNER/REPO.git ENTRY --help
```

Run anything that downloads packages in the background with output redirected to a log file (`... > install.log 2>&1`). Piping long-lived uv output through `tail`/`grep` has blocked stdout for hours on slow networks. Poll the log and the process list (`tasklist | grep -i uv`) instead of waiting synchronously.

## Step 4 — Network failure ladder

Some Windows networks break uv's HTTP/2 multiplexed TLS connections while HTTP/1.1 works normally. Recognize the signature:

```
Caused by: client error (Connect)
Caused by: unexpected eof while tunneling
```

A successful `curl` (HTTP/1.1) to the same index while uv fails is itself the diagnosis. Work the ladder in order, cheapest rung first:

1. Slow-but-working PyPI: retry with `UV_HTTP_TIMEOUT=300`. The default 30s is often too short and produces misleading per-package timeout errors.
2. Serialize downloads: add `UV_CONCURRENT_DOWNLOADS=1`. Parallel downloads can trigger the connection resets.
3. Alternate index via `%APPDATA%\uv\uv.toml`:
   ```toml
   [[index]]
   url = "https://mirrors.aliyun.com/pypi/simple/"
   default = true
   ```
   Warning: on the network where this ladder was mapped, Tsinghua and Aliyun mirrors both failed through uv with the same tunneling error — the problem was uv's connection layer, not the index. Do not loop through mirrors more than once.
4. Final fallback: pip uses HTTP/1.1 and completed installs where every uv route failed.
   ```bash
   pip install --user ENTRY_PACKAGE
   # or: pip install --user git+https://github.com/OWNER/REPO.git
   ```
   Then register `command` as the console script under `%APPDATA%\Python\PYTHON_VERSION\Scripts\` or as `python -m MODULE` — not uvx.
5. Remove any `uv.toml` mirror that did not help, so future uv runs are not pinned to a dead index.

## Step 5 — Register in the settings file

The file Qoder actually reads is `%USERPROFILE%\.qoder\settings.json` under the `mcpServers` key. There is no `mcp.json` in `.qoder` on current Qoder builds — do not create one.

1. Read the file first and take a timestamped backup. Broken JSON disables every existing MCP server.
2. Add an entry matching the style of the existing ones:

```json
"example-mcp": {
  "command": "C:\\Users\\USERNAME\\uv\\uv.exe",
  "args": ["tool", "run", "--from", "git+https://github.com/OWNER/REPO.git", "ENTRY"],
  "env": { "UV_HTTP_TIMEOUT": "300" }
}
```

Prefer `uv tool install --from git+REPO PACKAGE` plus the direct executable path over `tool run` once the package is cached: faster startup, no re-resolve per launch. If installed via pip, use the absolute Scripts-path executable instead.

3. Re-validate the JSON (`python -m json.tool settings.json`) after editing.

## Step 6 — Verify

- `mcp_list` will not show the new server in the current session; the catalog is loaded at session start. An absent result right after registering is expected, not a failure.
- Ask the user to restart Qoder or open a new session, then confirm the server's tools appear in the session's MCP listing.
- Until then, report status explicitly as "registered, not yet loaded". Do not claim "installed and working" based on the settings file alone.
- After reload, call one lightweight tool from the server to prove it responds.

## Common pitfalls

| Pitfall | Reality |
|---|---|
| `command: uvx` from README | uvx.exe may not exist; substitute full-path `uv.exe tool run` |
| Config in `.qoder/mcp.json` | Wrong file — it does not exist; use `settings.json` under `mcpServers` |
| Guessing repo owner | 404; always confirm from search results or market listings |
| Synchronous install with pipes | stdout can block for hours; use a background task and a log file |
| Mirror fixes everything | No — the tunneling error is in uv's HTTP/2 layer; switch to pip |
| mcp_list shows the new server immediately | It will not; new servers load only in a fresh session |
| Keyword filter in the extension market | May ignore the keyword and return a general list; read the results |
