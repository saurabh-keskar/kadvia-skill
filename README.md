# Skio CAD — Claude Skill

Work with CAD models by chatting with Claude. This skill teaches Claude how to drive the
**Skio CAD** desktop app: open STEP files, check dimensions and volume, change views, read
what you selected, and look at the model through screenshots before answering.

> Status: early preview (v0.1). Viewing and inspection today; parametric modeling next.

## What's here

| Path | Contents |
|---|---|
| `skills/skio-cad/SKILL.md` | The skill Claude reads |
| `skills/skio-cad/references/` | Views & conventions, troubleshooting |
| `skills/skio-cad/examples/` | Worked examples |
| `install/README.md` | Install Skio CAD and connect it to Claude |
| `docs/images/` | Screenshots |

## Quick start
1. Install and open Skio CAD (macOS or Windows) — see [install/README.md](install/README.md).
2. Connect Claude Desktop or Claude Code to Skio CAD (one entry, shown in the app under **AI → Claude**).
3. Add this skill:
   - **Claude Code:** copy `skills/skio-cad` to `~/.claude/skills/skio-cad`.
   - **Claude Desktop / claude.ai:** zip the `skills/skio-cad` folder and upload it in Settings → Capabilities → Skills.
4. Ask Claude: *"Open ~/Downloads/part.step in Skio and tell me its size and volume."*

## Example prompts
- "Open this STEP file in Skio and show me the four standard views."
- "What are the overall dimensions? Give them in inches too."
- "I clicked a face in Skio — what is its area?"
- "Switch to wireframe and look from the top. How many holes are there?"
