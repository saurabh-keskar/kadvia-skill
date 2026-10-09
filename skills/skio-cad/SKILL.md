---
name: skio-cad
description: Works with mechanical CAD models in the Skio CAD desktop app through the Skio MCP server - opens STEP files, reports dimensions, bounding boxes, volumes and face/edge counts, changes views and display modes, reads what the user selected, and takes multi-view screenshots to check results visually. Use when the user mentions Skio or Skio CAD, asks to open, view, inspect, measure or compare a STEP/STP part, or refers to something they clicked in the Skio window.
compatibility: Requires the Skio CAD desktop app (macOS or Windows) running, with its MCP server ("skio") connected to Claude Desktop or Claude Code.
metadata:
  version: "0.1.0"
---

# Skio CAD

Skio CAD is a desktop CAD application. Claude controls the **running app** through the
`skio` MCP server; everything Claude does appears live in the user's Skio CAD window.

Current capabilities: open and inspect STEP files (view, measure, screenshots, selection).
Parametric modeling tools are not available yet — say so if the user asks to create or
edit geometry, and offer what is possible (inspect, measure, compare, describe).

## Conventions
- Units are **millimetres** (areas mm², volumes mm³). Convert only when the user asks.
- **Z is up.** `front` looks along +Y (camera on −Y), `top` looks down −Z, `right` looks along −X.
- Model ids look like `m1`, body ids like `b0`. Get them from `skio:list_models` — never guess.
- Paths for `skio:open_step_file` must be absolute (`~/` is allowed).

Details: [references/views-and-conventions.md](references/views-and-conventions.md).

## Core loop (copy this checklist)
- [ ] `skio:skio_status` — is the app running and the window ready? What is already open?
- [ ] Do the task (open / inspect / change view).
- [ ] **Look before you answer:** `skio:render_views` (default iso, front, top, right) whenever
      the answer depends on shape, orientation or whether something loaded correctly.
- [ ] Report numbers from tool results, with units. Mention import warnings if any.

## Tools
| Tool | Use it to |
|---|---|
| `skio:skio_status` | Check the app is running; list open models |
| `skio:open_step_file` | Open a `.step`/`.stp` file in the user's window |
| `skio:list_models` / `skio:get_model_info` | Bodies, faces, edges, triangles, bounding box, volume, warnings |
| `skio:render_views` | See the model: 1–8 standard views as images (up to 2048 px) |
| `skio:set_view` / `skio:fit_view` | Move the user's camera to a standard view / zoom to fit |
| `skio:set_display_mode` | `shaded-edges` (default), `shaded`, `wireframe`, `hidden-lines` |
| `skio:get_selection` | What the user clicked (faces, edges, bodies) with approximate area/length |
| `skio:close_model` | Remove a model from the window (confirm with the user first) |

## Working with the user's selection
When the user says "this face", "the hole I clicked", "that edge":
1. Call `skio:get_selection`.
2. If nothing is selected, ask them to click it in Skio CAD (Cmd/Ctrl-click for several), then call it again.
3. Areas and lengths from selection come from the display mesh: say "about" for curved faces.

## Answering dimension questions
- Overall size: bounding box `size` from `skio:get_model_info` (X × Y × Z in mm).
- Volume: per body and in `totals`; it is missing when a body is not a closed solid — say so.
- Mass: volume × density (e.g. steel 7.85 g/cm³ = 0.00785 g/mm³, aluminium 6061 2.70 g/cm³). Show the formula.

## When something fails
Tool errors include a code and a hint. Common cases:
- *Skio CAD is not running* → ask the user to open the app, then retry.
- `not_found` → check the path (absolute? typo?) or call `skio:list_models` for valid ids.
- Import warnings (e.g. "faces could not be tessellated") → the model opened but parts may be missing; render it and tell the user what looks incomplete.

More: [references/troubleshooting.md](references/troubleshooting.md).

## Example
[examples/inspect-a-step-file.md](examples/inspect-a-step-file.md) — open a part, verify it visually, report size and volume.
