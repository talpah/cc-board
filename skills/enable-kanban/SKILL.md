# enable-kanban

Set up a project-specific kanban board by configuring the `dynamic-kanban-mcp` server for the current project.

## Steps

### 1. Detect project root

```bash
git rev-parse --show-toplevel
```

If that fails (not a git repo), use `pwd`. Store as `PROJECT_ROOT`.
`PROJECT_NAME` = basename of `PROJECT_ROOT`.

---

### 2. Locate the kanban server

Check for the server in this order:

1. `~/Projects/dynamic-kanban-mcp` — developer install (primary)
2. `~/.local/share/cc-board/server` — standard install

If neither exists, clone and install:
```bash
mkdir -p ~/.local/share/cc-board
git clone https://github.com/talpah/dynamic-kanban-mcp ~/.local/share/cc-board/server
uv pip install websockets pydantic
```

Store the resolved path as `SERVER_DIR`.

---

### 3. Verify the KANBAN_DATA_DIR patch

Check if `config.py` in `SERVER_DIR` already has the patch:
```bash
grep -q "KANBAN_DATA_DIR" $SERVER_DIR/config.py
```

If **not found**, apply the patch — use the Edit tool to update `get_progress_file_path` in `$SERVER_DIR/config.py`:

**Find:**
```python
    @classmethod
    def get_progress_file_path(cls):
        from pathlib import Path
        return str(Path(__file__).parent / "kanban-progress.json")
```

**Replace with:**
```python
    @classmethod
    def get_progress_file_path(cls):
        from pathlib import Path
        data_dir = os.getenv("KANBAN_DATA_DIR")
        if data_dir:
            return str(Path(data_dir) / "kanban-progress.json")
        return str(Path(__file__).parent / "kanban-progress.json")
```

---

### 4. Install Python dependencies

```bash
cd $SERVER_DIR && uv pip install -r requirements.txt
```

Only if deps are not already installed (check with `python3 -c "import websockets, pydantic"`).

---

### 5. Create project data directory

```bash
mkdir -p $PROJECT_ROOT/.kanban
```

---

### 6. Write or update `.mcp.json`

Read `$PROJECT_ROOT/.mcp.json` if it exists. Merge in the `kanban` server entry under `mcpServers`. Write the result back.

The entry to add/replace:
```json
{
  "mcpServers": {
    "kanban": {
      "command": "python3",
      "args": ["<SERVER_DIR>/mcp-kanban-server.py"],
      "env": {
        "KANBAN_DATA_DIR": "<PROJECT_ROOT>/.kanban",
        "KANBAN_WEBSOCKET_HOST": "127.0.0.1"
      }
    }
  }
}
```

Replace `<SERVER_DIR>` and `<PROJECT_ROOT>` with their actual absolute paths (expand `~`).

---

### 7. Update `CLAUDE.md`

If `$PROJECT_ROOT/CLAUDE.md` does **not** contain the string `## Kanban Board`, append this section:

```markdown

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
```

Replace `<SERVER_DIR>` with the actual absolute path.

---

### 8. Report to the user

Summarise what was done:
- Project: `<PROJECT_NAME>` (`<PROJECT_ROOT>`)
- Server: `<SERVER_DIR>/mcp-kanban-server.py`
- Board data: `<PROJECT_ROOT>/.kanban/`
- Board UI: `<SERVER_DIR>/kanban-board.html`
- **Restart Claude Code** (or reload MCP servers) for the kanban MCP to activate.
