# Design check (manufacturability)

`kadvia:check_design` analyses whether a part (or an imported STEP model) can be made with a
given process and lists every problem with its exact location. The user can see the same check
with a heat map in Kadvia (**Inspect → Design check**). The live reference is
`kadvia:modeling_reference {"topic": "design_check"}`.

## Contents
1. [When to run it](#1-when-to-run-it)
2. [Arguments](#2-arguments)
3. [Rules and default limits](#3-rules-and-default-limits)
4. [Reading the result](#4-reading-the-result)
5. [Fixing issues](#5-fixing-issues)
6. [Worked example: a CNC plate with a pocket](#6-worked-example-a-cnc-plate-with-a-pocket)
7. [Limits](#7-limits)

## 1. When to run it
- When a design is finished, before saving or exporting it for production.
- When the user asks "can this be machined / printed / molded?", "is it ready to print?" or similar.
- After every fix, to confirm the issue is gone and nothing new appeared.

Ask how the part will be made if it isn't clear. Don't guess between CNC and printing: the
rules differ.

## 2. Arguments
```json
{"model_id": "m1", "process": "cnc", "build_direction": "+Z",
 "params": {"toolRadius": 3, "minWall": 1.0}}
```
| Field | Meaning |
|---|---|
| `model_id` | An open part or imported model (not an assembly) |
| `process` | `cnc`, `print_fdm`, `print_sla`, `injection`, `sheet_metal` or `general` (default) |
| `build_direction` | `"+Z"` (default), `"-Z"`, `"+X"`, … or a vector `[x, y, z]`: the print build direction, the CNC tool axis (setups along ±) or the molding pull direction |
| `params` | Overrides of the limits (mm, degrees or ratios); `null` disables a rule. Keys: `minWall`, `maxWall`, `minHoleDiameter`, `maxHoleDepthRatio`, `minEdgeDistanceRatio`, `minHoleSpacingRatio`, `toolRadius`, `maxPocketDepthRatio`, `overhangAngle`, `maxBridge`, `minDraft`, `sheetTolerance`, `minBendRadiusRatio`, `buildVolume`, `minFeature`, `undercuts`, `standardHoles`, `setups` (CNC tool directions, e.g. `["+Z", "-Z", "+X"]`) |
| `body_id` | Check only one body (default all) |
| `max_issues` | Issues listed (default 40, most severe first; the rest are counted) |

Pass `params` when the user names a machine, nozzle, cutter or material, for example
`{"buildVolume": [220, 220, 250]}` for a smaller printer or `{"toolRadius": 1.5}` for a Ø3 cutter.

## 3. Rules and default limits
| Rule (`rule`) | cnc | print_fdm | print_sla | injection | sheet_metal | general |
|---|---|---|---|---|---|---|
| `min_wall` (`minWall`, mm) | 0.8 error | 0.8 error | 0.6 error | 1.2 error | – | 0.5 warning |
| `max_wall` (`maxWall`) | – | – | – | 3.5 | – | – |
| `small_hole` (`minHoleDiameter`) | 1.0 | 2.0 | 0.5 | – | ≥ t | – |
| `hole_depth_ratio` (`maxHoleDepthRatio`, × d) | 4 (error > 10) | – | – | – | – | 10 |
| `hole_edge_distance` (`minEdgeDistanceRatio`, centre to edge × d) | 1.5 | 1.5 | 1.5 | 1.5 | 1.5, web ≥ 2t | 1.5 |
| `hole_spacing` (`minHoleSpacingRatio`, centres × d) | 2 | – | – | 2 | – | 2 |
| `hole_standard_size` (`standardHoles`, ISO metric M2–M12 clearance / tap drill / counterbore) | info | info | info | – | info | info |
| `sharp_internal_corner` (concave edge along the tool axis) | error | – | – | info | – | – |
| `internal_corner_radius` (`toolRadius`, mm) | 1.0 | – | – | – | – | – |
| `deep_pocket` / `narrow_pocket` (`maxPocketDepthRatio`, × width) | 4 | – | – | – | – | – |
| `undercut` (`undercuts`, `setups`) | ± build direction | – | – | ± pull | – | – |
| `hole_setup` (hole axis off the setups) | info | – | – | – | – | – |
| `overhang` (`overhangAngle`, ° from vertical) / `bridge` (`maxBridge`) | – | 45° / 10 mm | 45° (info) | – | – | – |
| `draft` (`minDraft`, °) | – | – | – | 1.0 | – | – |
| `sheet_thickness` (`sheetTolerance`) / `bend_radius` (`minBendRadiusRatio`, × t) | – | – | – | – | 5 % / 1 | – |
| `build_volume` (`buildVolume`, mm) | 500×400×300 | 250×210×210 | 145×145×175 | – | 3000×1500×500 | – |
| `invalid_solid`, `tiny_edge`, `tiny_face` (`minFeature`, 0.1 mm) | all | all | all | all | all | all |

The defaults follow common shop rules (see [design-rules.md](design-rules.md) for the
background). The supplier's own limits always win.

## 4. Reading the result
- `summary`: `{errors, warnings, infos}`.
- `issues[]`: `{id, severity, rule, message, suggestion, location: {bodyId, point, faceId?, edgeId?, faceIds?}, value, limit}`, errors first, then warnings, then infos.
  - `value` vs `limit` is the measured number against the rule (mm, degrees or a ratio of the hole diameter / pocket width).
  - `location.point` is a 3D point on the problem; it works directly in `near` selectors.
- `holes[]`: every detected hole (diameter, depth, through, axis, centre, matching standard size).
- `stats`: minimum wall (and where), overhang area, overall size; `limits`: the effective limits used.
- `exact`: true for parts (exact geometry); false for imported models (fitted from the mesh, so treat values as approximate).

How to report it to the user:
1. Errors first, then warnings, in plain words with the numbers: "the wall at the pocket's
   right side is 0.6 mm; CNC needs at least 0.8 mm".
2. Infos are advice, not defects (for example a non-standard hole size). Mention them briefly.
3. Propose concrete fixes (next section) and **ask before changing the design**.
4. After the fix, run `kadvia:check_design` again and say what changed.

## 5. Fixing issues
Change the **parameter or feature that made the face**, not the symptom.

| Issue | Typical fix |
|---|---|
| `sharp_internal_corner` (CNC) | `cornerRadius` on the pocket's `rect` profile, or a fillet `{"edges": {"near": [location.point]}, "radius": 3}`; use `{"parallel": "Z"}` + `plane` filters if several corners need it |
| `internal_corner_radius` | Raise the corner radius to at least the tool radius (or pass the real `toolRadius` if the shop uses a smaller cutter) |
| `min_wall` | Move the face: change the sketch/extrude dimension or parameter that sets the wall (or `offset_face` on an imported part) |
| `max_wall` (molding) | Core out thick sections (`shell`, pockets from below) and keep walls uniform |
| `small_hole` / `hole_standard_size` | Use the size from `suggestion` (standard clearance or tap drill); `resize_hole` on imported parts |
| `hole_edge_distance` / `hole_spacing` | Move the holes (hole position parameters) or reduce the diameter |
| `hole_depth_ratio` / `deep_pocket` / `narrow_pocket` | Make it shallower or wider, or split it into a through feature machined from both sides |
| `undercut` / `hole_setup` (CNC) | Add a setup in `params.setups` if the shop can machine from that side, or redesign the feature to open toward a setup |
| `overhang` (FDM) | A 45° chamfer under the ledge, a different `build_direction`, or accept supports |
| `bridge` | Shorten the span, or add a chamfer or rib under it |
| `draft` (molding) | A `draft` feature on the walls or `draftAngle` on the extrude (≥ 1°) |
| `sheet_thickness` / `bend_radius` | Make the thickness uniform; inside bend radius ≥ t |
| `build_volume` | Ask: split the part, rotate it (`build_direction`), or use a bigger machine (`buildVolume`) |
| `invalid_solid` / `tiny_edge` / `tiny_face` | Find the feature that made the sliver (coplanar faces, a cut that ends exactly on a face) and overlap or extend it |

## 6. Worked example: a CNC plate with a pocket
**User:** "Can this be machined?" Model `m1` is a 100 × 60 × 8 plate (parameters `width`,
`depth`, `thickness`) with a 40 × 20 × 5 pocket cut from the top by sketch `pocket_sk`
(a `rect` without corner radius) and extrude `pocket`.

1. `kadvia:check_design {"model_id": "m1", "process": "cnc"}`. Example result (abridged; the exact wording varies):

```json
{"summary": {"errors": 4, "warnings": 0, "infos": 0},
 "issues": [
   {"id": "i1", "severity": "error", "rule": "sharp_internal_corner",
    "message": "Sharp inside corner along the tool axis (a milling cutter leaves a radius)",
    "suggestion": "Add a corner radius of at least 1 mm (3 mm is typical)",
    "location": {"bodyId": "b0", "point": [20, 10, 5.5], "edgeId": 31}}
 ]}
```
   The four pocket corners each produce an issue (only the first is shown).

2. Explain: "The pocket's four inside corners are sharp. A milling cutter is round, so it
   can't cut them; the machinist would need EDM or would round them anyway. I suggest a
   3 mm corner radius (a Ø6 cutter). Shall I add it?"
3. On yes, change the feature that made the corners (a parameter, so the user can tune it):

```json
{"model_id": "m1", "operations": [
  {"op": "set_parameter", "name": "pocket_r", "value": 3, "description": "Pocket corner radius (≥ cutter radius)"},
  {"op": "update_feature", "id": "pocket_sk", "patch": {"profiles": [
    {"kind": "rect", "center": [0, 0], "width": 40, "height": 20, "cornerRadius": "pocket_r"}]}}
]}
```
   (`update_feature` is a shallow merge, so the whole `profiles` list is sent.)
4. `kadvia:check_design {"model_id": "m1", "process": "cnc"}` again → `summary.errors` = 0.
   Report: "Fixed: the pocket corners now have R3 (parameter `pocket_r`). The check passes
   for CNC with a Ø6 cutter."

The same loop works for printing: `{"process": "print_fdm"}`, then fix overhangs with
chamfers or try `"build_direction": "-Z"` (upside down) and compare the overhang area in `stats`.

## 7. Limits
- Assemblies are not checked; check each part, and use `kadvia:check_interference` for the assembly.
- Imported models are checked on their mesh (`exact: false`): values are approximate. Convert them with `kadvia:convert_to_part` for exact results.
- Wall thickness is sampled over the surface: confirm critical walls with `kadvia:measure` (two faces give their exact distance).
- The check knows geometry, not material or tolerances: say what you assumed (cutter size, nozzle, machine).
