# Install Kadvia and connect Claude

## 1. Install the app
Download Kadvia for macOS (.dmg) or Windows (installer) from the Kadvia website and
open it once. Claude can only work with Kadvia while the app is running.

## 2. Connect Claude
Kadvia ships a small MCP server, `kadvia-mcp`, inside the app. In Kadvia open
**AI → Claude** to see its exact path and setup snippet.

### Claude Desktop
Add to `claude_desktop_config.json` (Claude Desktop → Settings → Developer → Edit Config), then restart Claude Desktop:

```json
{
  "mcpServers": {
    "kadvia": { "command": "/Applications/Kadvia.app/Contents/MacOS/kadvia-mcp" }
  }
}
```

On Windows the command is the `kadvia-mcp.exe` next to `Kadvia.exe` in the install folder.

### Claude Code
```bash
claude mcp add --scope user kadvia -- "/Applications/Kadvia.app/Contents/MacOS/kadvia-mcp"
```

## 3. Check it works
Ask Claude: *"Is Kadvia running?"* Claude calls `kadvia_status` and reports the app version.
Then try: *"Make a 50 mm cube with a 20 mm hole through it in Kadvia."* The part appears in the
Kadvia window as Claude builds it.
In Kadvia, **AI → Claude** shows a green dot while Claude is connected.

## Privacy
The connection is local to your computer (no network port). Each app launch creates a new
access key readable only by your user account.
