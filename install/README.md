# Install Kadvia and connect your AI client

## 1. Install the app
Download Kadvia for macOS (.dmg) or Windows (installer) from the Kadvia website and open it
once. Your AI can only work with Kadvia while the app is running.

## 2. Connect your AI client
Kadvia ships a small MCP server, `kadvia-mcp`, inside the app. In Kadvia open
**AI → Connect AI assistant**: it lists the AI clients installed on your computer and connects
them with one click (it shows what it will write and backs up the client's config first).

### By hand
Add a stdio MCP server named `kadvia` whose command is the absolute path to `kadvia-mcp`:
- macOS: `/Applications/Kadvia.app/Contents/MacOS/kadvia-mcp`
- Windows: `kadvia-mcp.exe` next to `Kadvia.exe` in the install folder

Most clients use this JSON shape in their MCP config:

```json
{ "mcpServers": { "kadvia": { "command": "/Applications/Kadvia.app/Contents/MacOS/kadvia-mcp" } } }
```

| Client | Config |
|---|---|
| Claude Desktop | `claude_desktop_config.json` (Settings → Developer → Edit Config); restart Claude |
| Claude Code | `claude mcp add --scope user kadvia -- "/Applications/Kadvia.app/Contents/MacOS/kadvia-mcp"` |
| Cursor | `~/.cursor/mcp.json` |
| Windsurf | `~/.codeium/windsurf/mcp_config.json` |
| Gemini CLI | `~/.gemini/settings.json` |
| LM Studio (local models) | `~/.lmstudio/mcp.json` |
| VS Code (Copilot agent mode) | user `mcp.json`: `{"servers": {"kadvia": {"type": "stdio", "command": "<path>"}}}` |
| OpenAI Codex CLI | `~/.codex/config.toml`: `[mcp_servers.kadvia]` + `command = "<path>"` |

Config locations can change between client versions — check your client's MCP documentation.
Clients that only support **remote** MCP servers over HTTPS (for example the ChatGPT apps) are
not supported yet; a remote connector is planned.

## 3. Check it works
Ask your AI: *"Is Kadvia running?"* — it calls `kadvia_status` and reports the app version.
Then try: *"Make a 50 mm cube with a 20 mm hole through it in Kadvia."* The part appears in the
Kadvia window as the AI builds it, and Kadvia shows which assistant made the change.

## Privacy
The connection is local to your computer (no network port). Each app launch creates a new
access key readable only by your user account. Your model files stay on your computer; what the
AI sees (numbers and renders) goes to whichever AI service your client uses.
