# Kadvia — AI instructions & skill

Design, edit and check CAD parts by chatting with **any AI assistant** that supports MCP —
Claude, ChatGPT/Codex, Gemini, Cursor, VS Code Copilot, Windsurf, LM Studio with local models,
and more. Kadvia is a desktop CAD app with a built-in MCP server; this repository teaches your
AI assistant how to use it well:
- build **parametric parts** from a description, or from a photo or drawing traced as a
  calibrated **reference image**, with named dimensions you can change later;
- edit existing parts through their parameters, with a **rollback bar** to insert features
  anywhere in the history;
- add **standard holes** with the hole wizard — tapped, clearance, counterbored, countersunk and
  counterdrilled, ISO metric and inch sizes — and **cosmetic or modeled threads**;
- sketch with **splines, ellipses, slots, text and conics**, edit sketches with trim, extend,
  offset, fillet, mirror and patterns, project model edges, **import DXF** outlines and
  **export any sketch as DXF** for laser or waterjet cutting;
- assign **materials** from a library of 22 engineering materials for real mass and BOM weights,
  set **colours and finishes** per part, body or face, and make **product renders** (studio,
  outdoor or warehouse lighting, transparent background);
- **edit imported STEP files**: resize holes, remove fillets and chamfers, push/pull and move
  faces, add new features, export STEP again;
- put parts together in **assemblies** with mates (including gear, rack-and-pinion, cam, limit,
  symmetric and width mates), mate suggestions, rigid **sub-assemblies**, DOF diagnostics,
  **motion previews** (drag a component and see what moves), a top-level, indented or flat bill
  of materials and an interference check;
- make dimensioned **2D drawings** of parts, assemblies and STEP models: standard, detail, section
  and exploded views, ordinate/baseline/chain dimensions, **tolerances and ISO fits**, **GD&T**
  (datums, feature control frames) and surface finish, standard **hole callouts** and thread
  lines, **BOM tables and balloons**, multiple sheets and a title block, exported as PDF, DXF or
  SVG;
- run a **design check** for CNC, 3D printing (FDM/SLA), injection molding or sheet metal and
  fix the issues it finds;
- verify everything with exact mass properties, measurements and screenshots from any angle
  (custom directions, perspective, sections) of **one model at a time**, or of chosen bodies
  only (hide the lid to look inside), without touching your other open models, and **point at
  faces** by highlighting them in your Kadvia window;
- save `.kadvia` / `.kasm` / `.kdraw`, or export STEP, STL, 3MF, OBJ or a sketch as DXF.

> Status: preview (v0.7). Parametric modeling (sketch, extrude, revolve, holes, fillets and
> chamfers including mixed convex/concave corners, patterns, mirror, loft, sweep, shell, draft),
> hole wizard and threads, sketch tools and DXF import/export, materials, appearances and
> product renders, reference images, imported-STEP editing (mesh-only bodies kept), assemblies
> with sub-assemblies, motion mates, mate suggestions and motion previews, 2D drawings of parts
> and assemblies with tolerances, GD&T, hole callouts, BOM and collision-free balloons, design
> check with feature-aware hole sizes, viewing (per-model renders, isolated bodies, custom
> angles, perspective, sections), selection highlighting, rebuild and inspection.

The Kadvia MCP server already sends core instructions to every client, so this repository is
optional — it makes results better (design rules, workflows, worked examples).

## What's here

| Path | For |
|---|---|
| `instructions/AGENTS.md` | **Any AI client**: use as Cursor rules, Codex `AGENTS.md`, Gemini `GEMINI.md`, VS Code Copilot instructions, or a system prompt |
| `skills/kadvia/` | The same guidance in the open **Agent Skills** format (`SKILL.md` + references + examples), e.g. for Claude |
| `skills/kadvia/references/` | Modeling operations, holes and threads, sketch tools and DXF import/export, materials and appearance, image-to-CAD and reference images, editing STEP files, assemblies, drawings, design check, design rules (DFM), views & conventions, troubleshooting |
| `skills/kadvia/examples/` | Worked examples: L-bracket, flange, enclosure, inspecting a STEP file (more in each reference: M5 tapped and M6 counterbored holes, engraved text, DXF outline to part and back to DXF, anodised aluminium product render, bracket-on-plate assembly, gear pair, L-bracket drawing, toleranced flange drawing with GD&T, assembly drawing with BOM and balloons, STEP hole resize, CNC design-check fix, photo tracing) |
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

**Holes and threads**
- "Add four M5 tapped holes, 10 deep, 8 mm in from the corners."
- "Put counterbored holes for M6 socket head screws in each corner of the plate."
- "Make those clearance holes a close fit, and switch them to M8."
- "Thread the end of the shaft M10 for 20 mm — a real thread, I'm going to 3D print it."

**Sketches and DXF**
- "Engrave 'LOT 42' on the top face, 5 mm high and 0.5 mm deep."
- "Import ~/Downloads/gasket.dxf — only the OUTLINE layer — and extrude it 3 mm."
- "Make an elliptical plate 80 × 50 with a 30 mm slot in the middle."
- "Offset the outline 2 mm outward and round its corners R3."
- "Export that sketch as a DXF for the laser cutter."

**Materials and renders**
- "Make it anodised blue aluminium and render a product shot on a transparent background."
- "What does it weigh in stainless 316? And in PEEK?"
- "Make the feet black rubber and the top face polished."

**Image-to-CAD**
- "Here's a photo of a bracket next to a ruler. Rebuild it in Kadvia as a parametric part."
- "Put ~/Pictures/bracket-front.jpg on the front plane, the base is 100 mm wide, and trace it."
- "Recreate this drawing (attached) in Kadvia. Ask me if any dimension is unclear."

**Editing STEP files**
- "Open ~/Downloads/mount.step, make all the Ø6 holes Ø6.6 and remove the fillets, then export a new STEP."
- "Make the top plate of this STEP part 2 mm thicker and add two M5 tapped holes on it."
- "What holes does this STEP file have? List them by size."

**Assemblies**
- "Bolt the bracket to the plate with an M6 bolt and check nothing collides."
- "Make an assembly of the enclosure and its lid, and give me a BOM with masses in PETG."
- "Why can the shaft still move? Fix the mates so only rotation is free."
- "Mesh a 20-tooth pinion with a 40-tooth gear, module 2, and turn the pinion 30°."
- "Limit the slider's travel to 10–60 mm."
- "Which mate should I use between the shaft and this bore?"
- "Turn the crank a quarter turn and tell me how far the slider moves — don't change anything."
- "Use the saved bracket-on-plate assembly twice on this base and give me an indented BOM and a flat parts list."

**2D drawings**
- "Make an A4 drawing of the bracket with material AL 6061-T6 and tolerance ISO 2768-m, then export a PDF and a DXF."
- "Add a section view through the holes and a detail view of the bend."
- "Switch the drawing to third-angle projection on A3."
- "Make the bore an H7 fit, the thickness ±0.1, and add a position tolerance for the bolt holes to datums A and B."
- "Dimension the hole positions as ordinates from the left edge."
- "Make an assembly drawing with a parts list and balloons, and an exploded view on a second sheet."
- "Put the balloons on the right of the exploded view, 4 mm apart."

**Design check**
- "Can this be machined on a 3-axis mill with a 6 mm cutter? Fix what can't."
- "Is it ready to print in PETG? Check overhangs with Z up and upside down."
- "Check the housing for injection molding and add the draft it needs."

**Parametric edits**
- "Make it 20 mm wider and switch the holes to M6."
- "Add a 2 mm fillet to the top edges, then show me the iso view."
- "Change the bolt pattern to 8 holes and tell me the new weight in steel."
- "Roll back to the base plate and add a rib before the fillets."
- "Undo that last change."

**Checking and exporting**
- "What does this part weigh in PETG? Is any wall thinner than 1.2 mm?"
- "Save it as ~/parts/bracket.kadvia and export a STEP for the machinist."
- "Export a 3MF for my slicer."
- "Open ~/Downloads/part.step and tell me its size and volume, in inches too."
- "I clicked a face in Kadvia. What is its area?"
- "Which face is too thin? Highlight it for me."
- "Show me the underside in perspective, and turn on a section through the bore."
- "Render the housing without its lid so I can see the bosses — leave my other parts open."
- "I re-exported the linked STEP file — rebuild the part."
