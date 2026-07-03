# Migrating from a git-clone install to PyPI / `uvx`

This guide moves an existing **git-clone + venv** install of `wiki-js-mcp-server`
to the published PyPI package, run on demand with [`uvx`](https://docs.astral.sh/uv/).

No more `git pull`, venv, or local `.env`: each client runs
`uvx wiki-js-mcp-server@latest`, which fetches the package from PyPI and runs it.
Because every machine runs its own process, each user keeps **their own** Wiki.js
API key.

> **Windows note:** it works exactly like macOS/Linux. Only three things differ:
> (1) how you install `uv`, (2) where the client config lives, (3) **absolute
> paths** for `WIKIJS_MCP_DB` / `LOG_FILE` in Windows syntax. Windows-specific
> steps are marked 🪟.

---

## Step 0 — Save your current settings ⚠️ (don't skip)

A git-clone install kept its credentials in a `.env` at the repo root. With `uvx`
there is **no clone and no `.env`** — those values must move into the **client
config**. Open the old `.env` (or the old Claude config) and note:

```
WIKIJS_URL=...
WIKIJS_API_KEY=...
WIKIJS_MCP_DB=...   # old path
LOG_FILE=...        # old path
```

---

## Step 1 — Install `uv`

**macOS / Linux**
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

🪟 **Windows (PowerShell)**
```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Close and reopen the terminal, then check:
```bash
uv --version
```

---

## Step 2 — Pre-flight: confirm the package resolves

```bash
uvx --from wiki-js-mcp-server@latest python -c "import server; print(server.__version__)"
```

Should print the version (e.g. `1.1.0`). This downloads the package into an
ephemeral environment, prints the version, and exits — it does **not** start the
server.

---

## Step 3 — Prepare a data folder (absolute paths)

The server process does not start from your project directory, so
`WIKIJS_MCP_DB` and `LOG_FILE` **must be absolute paths**, and their parent
folder must exist.

**macOS / Linux**
```bash
mkdir -p ~/.local/share/wiki-js-mcp
# WIKIJS_MCP_DB=/Users/<you>/.local/share/wiki-js-mcp/wikijs_mappings.db
# LOG_FILE=/Users/<you>/.local/share/wiki-js-mcp/wikijs_mcp.log
```

🪟 **Windows (PowerShell)**
```powershell
mkdir "$env:LOCALAPPDATA\wiki-js-mcp" -Force
# WIKIJS_MCP_DB = C:\Users\<you>\AppData\Local\wiki-js-mcp\wikijs_mappings.db
# LOG_FILE      = C:\Users\<you>\AppData\Local\wiki-js-mcp\wikijs_mcp.log
```

---

## Step 4 — Reconfigure the client

### Option A — Claude Code (CLI, identical on every OS)

```bash
claude mcp remove wikijs
claude mcp add wikijs \
  --env WIKIJS_URL=https://your-wiki.example.com \
  --env WIKIJS_API_KEY=<your-key> \
  --env WIKIJS_MCP_DB=<absolute-db-path> \
  --env LOG_FILE=<absolute-log-path> \
  -- uvx wiki-js-mcp-server@latest --stdio
```

🪟 On Windows PowerShell, use backticks for line continuation and quote the paths:
```powershell
claude mcp remove wikijs
claude mcp add wikijs `
  --env WIKIJS_URL=https://your-wiki.example.com `
  --env WIKIJS_API_KEY=<your-key> `
  --env WIKIJS_MCP_DB="C:\Users\<you>\AppData\Local\wiki-js-mcp\wikijs_mappings.db" `
  --env LOG_FILE="C:\Users\<you>\AppData\Local\wiki-js-mcp\wikijs_mcp.log" `
  -- uvx wiki-js-mcp-server@latest --stdio
```

### Option B — Claude Desktop (JSON config)

Config file location:
- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- 🪟 Windows: `%APPDATA%\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "wikijs": {
      "command": "uvx",
      "args": ["wiki-js-mcp-server@latest", "--stdio"],
      "env": {
        "WIKIJS_URL": "https://your-wiki.example.com",
        "WIKIJS_API_KEY": "<your-key>",
        "WIKIJS_MCP_DB": "C:\\Users\\<you>\\AppData\\Local\\wiki-js-mcp\\wikijs_mappings.db",
        "LOG_FILE": "C:\\Users\\<you>\\AppData\\Local\\wiki-js-mcp\\wikijs_mcp.log"
      }
    }
  }
}
```

🪟 In JSON, Windows paths need **double backslashes** (`\\`).

---

## Step 5 — Restart and verify

Restart the client, then ask: *"Check my Wiki.js connection status"*.
The reply must include **`server_version`** (e.g. `1.1.0`) — that confirms you
are running the PyPI build, not the old clone.

---

## Step 6 — Clean up the old clone (only after Step 5 passes)

If you want to keep the file↔page mappings, copy the old SQLite DB into the new
data folder first, then delete the clone and its venv.

**macOS / Linux**
```bash
cp /path/to/clone/wikijs_mappings.db ~/.local/share/wiki-js-mcp/
rm -rf /path/to/clone
```

🪟 **Windows (PowerShell)**
```powershell
copy "C:\path\to\clone\wikijs_mappings.db" "$env:LOCALAPPDATA\wiki-js-mcp\"
Remove-Item -Recurse -Force "C:\path\to\clone"
```

---

## Update policy

| Config arg | Behaviour |
|---|---|
| `uvx wiki-js-mcp-server@latest` | Always newest. Each client restart re-resolves PyPI and picks up new releases. |
| `uvx wiki-js-mcp-server@1.1.0` | Pinned. Never changes until you edit the version. |
| `uvx wiki-js-mcp-server` | Uses whatever `uvx` cached; `uv cache clean wiki-js-mcp-server` forces a refresh. |

---

## 🪟 Windows-specific gotchas

| Problem | Fix |
|---|---|
| **Claude Desktop can't find `uvx`** (it doesn't always inherit your PATH) | Put the full path in `command`. Find it with `(Get-Command uvx).Source` — typically `C:\Users\<you>\.local\bin\uvx.exe` (escape as `\\` in JSON). |
| Relative DB/log paths | Always **absolute** — otherwise the server fails to start (read-only cwd). |
| Backslashes in JSON | Double them: `C:\\Users\\...` |
| `tzdata` on Windows | Already handled — the package declares it as a `sys_platform == 'win32'` dependency. |

## Rollback

The old clone keeps working until you delete it. If anything goes wrong, restore
the previous client config pointing at the clone's `venv` Python. Only run Step 6
after Step 5 succeeds.
