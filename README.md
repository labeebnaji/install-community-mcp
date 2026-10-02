# install-community-mcp

A Qoder CLI skill that installs and registers community (non-official) MCP servers from GitHub on Windows: repo audit, runtime smoke test, network fallback ladder, registration in the Qoder settings file, and honest verification.

## Why

Installing a third-party MCP server looks like two lines in a README, but on Windows behind a restrictive network it hides a sequence of traps: `uvx.exe` missing while `uv.exe` exists, HTTP/2 connection resets that break uv but not pip, PyPI timeouts reported as package failures, a settings file that is not where most guides say it is, and a verification step that silently lies because new servers do not load into the running session. This skill encodes that whole path, walked end to end on a real install, so an agent following it does not rediscover each dead end.

## Requirements

- Windows with Qoder CLI.
- Python 3.10+ on PATH, or a `uv` binary somewhere on disk (the skill locates it).
- Network access to GitHub and to a Python package index.
- No API keys are needed by the skill itself. Individual MCP servers may require their own keys or environment variables; the skill's audit step surfaces these before any install.

## Install

Copy the `install-community-mcp` folder into your skills directory:

- User-level (all projects): `%USERPROFILE%\.qoder\skills\install-community-mcp\`
- Project-level (single repo): `.qoder\skills\install-community-mcp\`

Restart the session or run `/skills reload`. Verify with `/skills list`.

## Use

Invoke directly:

```
/install-community-mcp
```

Or ask for something like "install the MCP server from this GitHub repo" — the skill triggers when the request is about adding an MCP server from a public repository.

## What it does

1. Searches the official extension market first, then locates community candidates by verified repository names.
2. Audits the candidate repo before executing anything: entry points, dependencies, required environment variables, license, activity, network bind behavior. Reports to the user and waits for approval.
3. Probes the local runtime (uv, uvx, Python) and runs an isolated `--help` smoke test in the background with a log file.
4. Works a network fallback ladder ordered cheapest-first: raised timeout, serialized downloads, alternate package index, then pip over HTTP/1.1 when uv's HTTP/2 connections fail.
5. Registers the server in `%USERPROFILE%\.qoder\settings.json` under `mcpServers`, with a backup and JSON re-validation.
6. Verifies honestly: a registered server only counts as working once its tools appear in a fresh session and one tool responds to a call.

## Repository layout

```
install-community-mcp/
├── README.md
├── LICENSE
└── install-community-mcp/
    └── SKILL.md
```

## Author

Labeeb Derhem — labeebderhem@gmail.com — [GitHub](https://github.com/labeebnaji)

## Notes

- The skill targets Qoder's configuration layout. For other MCP clients, the settings-file location and session-reload behavior will differ; everything else (audit, runtime probe, network ladder, registration shape) applies unchanged.
- Network observations (uv HTTP/2 tunneling failures, mirror behavior) come from documented installs on restrictive networks and are ordered as fallbacks, not universal claims.

## License

MIT — see [LICENSE](LICENSE).
