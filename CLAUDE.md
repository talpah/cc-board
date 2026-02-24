# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`cc-board` is a Claude Code plugin that adds a per-project kanban board to any project. It wraps [`dynamic-kanban-mcp`](https://github.com/talpah/dynamic-kanban-mcp) (forked at `~/Projects/dynamic-kanban-mcp`) with a `/enable-kanban` slash command that handles setup automatically.

## Architecture

```
cc-board/                          ← this repo (the plugin)
├── .claude-plugin/plugin.json     ← plugin manifest
├── commands/enable-kanban.md      ← slash command definition
└── skills/enable-kanban/SKILL.md  ← setup instructions executed by CC

~/Projects/dynamic-kanban-mcp/     ← the MCP server (separate repo, our fork)
├── mcp-kanban-server.py           ← MCP stdio server entry point
├── kanban_controller.py           ← board logic and WebSocket broadcast
├── config.py                      ← patched: supports KANBAN_DATA_DIR env var
├── mcp_protocol.py                ← JSON-RPC 2.0 over stdio
├── models.py                      ← Pydantic data models
└── kanban-board.html              ← browser UI (connects via WebSocket on :8765)
```

Per-project data lands in `<project>/.kanban/kanban-progress.json`.

## How it works

The `/enable-kanban` command (see `skills/enable-kanban/SKILL.md`) does:

1. Detects git root as the project root
2. Verifies `~/Projects/dynamic-kanban-mcp` is present and patched
3. Creates `.kanban/` in the target project
4. Writes/merges a `kanban` entry into the project's `.mcp.json`
5. Appends a `## Kanban Board` section to the project's `CLAUDE.md`

Since the MCP transport is **stdio**, Claude Code manages the server process lifecycle — no manual start/stop needed.

## Development

When modifying the plugin:
- `SKILL.md` changes take effect immediately (no reinstall needed)
- `plugin.json` or `commands/` changes require plugin reinstall: run `/plugin` in CC

When modifying the server (`~/Projects/dynamic-kanban-mcp`):
- Changes take effect on the next Claude Code session restart (server is a subprocess)
- `config.py` must retain the `KANBAN_DATA_DIR` patch — do not revert it

## Key constraint

`config.py` in the server repo has a one-line patch to `get_progress_file_path` that redirects data files via `KANBAN_DATA_DIR`. This is what enables per-project isolation. It must survive any upstream merges.
