# Kadvia — AI instructions & skill

Design and inspect CAD parts by chatting with **any AI assistant** that supports MCP — Claude,
ChatGPT/Codex, Gemini, Cursor, VS Code Copilot, Windsurf, LM Studio with local models, and more.
Kadvia is a desktop CAD app with a built-in MCP server; this repository teaches your AI how to
use it well:
- build **parametric parts** from a description or a photo or drawing, with named dimensions
  you can change later;
- edit existing parts through their parameters;
- verify parts with exact mass properties and screenshots;
- save `.kadvia` or export STEP/STL;
- open and measure STEP files.

> Status: preview (v0.2). Parametric modeling (sketch, extrude, revolve, holes, fillets,
> chamfers, patterns, mirror), viewing and inspection.

The Kadvia MCP server already sends core instructions to every client, so this repository is
optional — it makes results better (design rules, workflows, worked examples).

## What's here

| Path | For |
|---|---|
| `instructions/AGENTS.md` | **Any AI client**: use as Cursor rules, Codex `AGENTS.md`, Gemini `GEMINI.md`, VS Code Copilot instructions, or a system prompt |
| `skills/kadvia/` | The same guidance in the open **Agent Skills** format (`SKILL.md` + references + examples), e.g. for Claude |
| `skills/kadvia/references/` | Modeling operations, image-to-CAD, design rules (DFM), views & conventions, troubleshooting |
| `skills/kadvia/examples/` | Worked examples: L-bracket, flange, enclosure, inspecting a STEP file |
| `install/README.md` | Install Kadvia and connect your AI client |

## Quick start
1. Install and open Kadvia (macOS or Windows) — see [install/README.md](install/README.md).
2. In Kadvia choose **AI → Connect AI assistant** and click Connect next to your client
   (or add the `kadvia` MCP server to your client's config by hand).
3. Optional — add the instructions:
   - **Cursor:** copy `instructions/AGENTS.md` to `.cursor/rules/kadvia.mdc` in your project.
   - **OpenAI Codex CLI:** put it in your project's `AGENTS.md`.
   - **Gemini CLI:** put it in `GEMINI.md`.
   - **VS Code Copilot:** `.github/copilot-instructions.md`.
   - **Claude:** copy `skills/kadvia` to `~/.claude/skills/kadvia` (Claude Code) or upload the zipped folder as a skill (Claude apps).
   - **Other clients:** paste `instructions/AGENTS.md` into the system prompt.
4. Ask your AI: *"In Kadvia, make a 100 × 60 × 6 mm aluminium plate with four M5 holes 10 mm from the corners."*

Works best with models that accept images (they can look at Kadvia's renders); text-only models
still get every number (bounding box, volume, mass).

## Example prompts

**Text-to-CAD**
- "Make an L-bracket from 5 mm aluminium: 60 mm wide, legs 50 and 40 mm, two M5 holes per leg."
- "Design a flange: OD 140, 14 mm thick, Ø70 × 20 hub, 50 mm bore, 6 × M10 on a 100 mm PCD."
- "I need a 3D-printed box for an 80 × 50 mm PCB, 2 mm walls, M3 lid screws and a USB-C cut-out."
- "Model a shaft: Ø20 for 40 mm, then Ø15 for 25 mm, with a 1 mm chamfer at both ends."

**Image-to-CAD**
- "Here's a photo of a bracket next to a ruler. Rebuild it in Kadvia as a parametric part."
- "Recreate this drawing (attached) in Kadvia. Ask me if any dimension is unclear."

**Parametric edits**
- "Make it 20 mm wider and switch the holes to M6."
- "Add a 2 mm fillet to the top edges, then show me the iso view."
- "Change the bolt pattern to 8 holes and tell me the new weight in steel."
- "Undo that last change."

**Checking and exporting**
- "What does this part weigh in PETG? Is any wall thinner than 1.2 mm?"
- "Save it as ~/parts/bracket.kadvia and export a STEP for the machinist."
- "Open ~/Downloads/part.step and tell me its size and volume, in inches too."
- "I clicked a face in Kadvia. What is its area?"
