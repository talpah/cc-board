---
name: Enable Kanban
description: This skill should be used when the user runs /cc-board:enable-kanban or asks to "set up kanban", "add kanban board", "enable kanban for this project", or "set up task board". Sets up dynamic-kanban-mcp for the current project by configuring .mcp.json and updating CLAUDE.md.
version: 1.0.0
---

# enable-kanban

Set up a project-specific kanban board by configuring the `dynamic-kanban-mcp` server for the current project.

## Steps

### 1. Detect project root

Run `git rev-parse --show-toplevel`. If that fails (not a git repo), use `pwd`. Store as `PROJECT_ROOT`. `PROJECT_NAME` = basename of `PROJECT_ROOT`.

### 2. Locate the kanban server

Check for the server in this order: `~/Projects/dynamic-kanban-mcp` (developer install, primary), then `~/.local/share/cc-board/server` (standard install).

If neither exists, run: `mkdir -p ~/.local/share/cc-board && git clone https://github.com/talpah/dynamic-kanban-mcp ~/.local/share/cc-board/server`

Store the resolved path as `SERVER_DIR`.

### 3. Verify the KANBAN_DATA_DIR patch

Run `grep -q "KANBAN_DATA_DIR" $SERVER_DIR/config.py` to check if already patched.

If not found, use the Edit tool to update `get_progress_file_path` in `$SERVER_DIR/config.py`. Find the method that returns `str(Path(__file__).parent / "kanban-progress.json")` and replace with a version that first checks `os.getenv("KANBAN_DATA_DIR")` and returns `str(Path(data_dir) / "kanban-progress.json")` when set.

### 4. Install Python dependencies

Run `cd $SERVER_DIR && uv sync` to install from `pyproject.toml` into `$SERVER_DIR/.venv`.

### 5. Create project data directory

Run `mkdir -p $PROJECT_ROOT/.kanban`.

### 6. Write or update `.mcp.json`

Read `$PROJECT_ROOT/.mcp.json` if it exists. Merge in the `kanban` entry under `mcpServers`. Write back. The entry:

    {
      "mcpServers": {
        "kanban": {
          "command": "uv",
          "args": ["run", "--project", "<SERVER_DIR>", "python", "<SERVER_DIR>/mcp-kanban-server.py"],
          "env": {
            "KANBAN_DATA_DIR": "<PROJECT_ROOT>/.kanban",
            "KANBAN_WEBSOCKET_HOST": "127.0.0.1"
          }
        }
      }
    }

Replace `<SERVER_DIR>` and `<PROJECT_ROOT>` with actual absolute paths.

### 7. Update `CLAUDE.md`

If `$PROJECT_ROOT/CLAUDE.md` does not contain `## Kanban Board`, append this section (replace `<SERVER_DIR>` with actual path):

    ## Kanban Board

    This project has a kanban board enabled. The `kanban` MCP server starts automatically with each Claude Code session.

    **At session start:** call `kanban_status` to see the current board.
    **After completing a task:** call `kanban_get_next_task` for the next item.

    Key tools:
    - `kanban_status` — board overview with task counts per column
    - `add_feature` — add a task (`id`, `title`, `description`, `priority`, `effort`)
    - `kanban_move_card` — advance: `backlog` → `ready` → `progress` → `testing` → `done`
    - `kanban_get_next_task` — next highest-priority ready task
    - `kanban_start_session` / `kanban_end_session` — track work sessions

    Board data: `.kanban/kanban-progress.json`
    Board UI: open `<SERVER_DIR>/kanban-board.html` in your browser (WebSocket on port 8765).

### 8. Report to the user

Summarise what was done: project name and root, server path, board data path, board UI path. Remind user to restart Claude Code (or reload MCP servers) for the kanban MCP to activate.
