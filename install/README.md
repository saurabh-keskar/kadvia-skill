# Install Skio CAD and connect Claude

## 1. Install the app
Download Skio CAD for macOS (.dmg) or Windows (installer) from the Skio CAD website and
open it once. Claude can only work with Skio while the app is running.

## 2. Connect Claude
Skio CAD ships a small MCP server, `skio-mcp`, inside the app. In Skio CAD open
**AI → Claude** to see its exact path and setup snippet.

### Claude Desktop
Add to `claude_desktop_config.json` (Claude Desktop → Settings → Developer → Edit Config), then restart Claude Desktop:

```json
{
  "mcpServers": {
    "skio": { "command": "/Applications/Skio CAD.app/Contents/MacOS/skio-mcp" }
  }
}
```

On Windows the command is the `skio-mcp.exe` next to `Skio CAD.exe` in the install folder.

### Claude Code
```bash
claude mcp add --scope user skio -- "/Applications/Skio CAD.app/Contents/MacOS/skio-mcp"
```

## 3. Check it works
Ask Claude: *"Is Skio CAD running?"* Claude calls `skio_status` and reports the app version.
Then try: *"Make a 50 mm cube with a 20 mm hole through it in Skio."* The part appears in the
Skio window as Claude builds it.
In Skio CAD, **AI → Claude** shows a green dot while Claude is connected.

## Privacy
The connection is local to your computer (no network port). Each app launch creates a new
access key readable only by your user account.
