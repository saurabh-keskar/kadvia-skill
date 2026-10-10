# Assemblies (`.kasm`)

An assembly places **components** (Kadvia parts, STEP files or saved assemblies used as
sub-assemblies) in space and constrains them with **mates**: positioning mates (coincident,
concentric, distance, …), limit mates, and motion mates (gear, rack and pinion, cam). Kadvia
solves the mates, reports what is still free to move, lists a bill of materials (top-level,
indented or flat) and checks for interference. The live reference is
`kadvia:modeling_reference {"topic": "assemblies"}`.

## Contents
1. [Tools](#1-tools)
2. [Workflow](#2-workflow)
3. [Components](#3-components)
4. [Mates](#4-mates)
5. [Reading the solve report](#5-reading-the-solve-report)
6. [BOM, interference, exploded view, export](#6-bom-interference-exploded-view-export)
7. [Worked example: bracket bolted to a plate](#7-worked-example-bracket-bolted-to-a-plate)
8. [Motion mates: gear, rack and pinion, cam](#8-motion-mates-gear-rack-and-pinion-cam)
9. [Limit, symmetric and width mates](#9-limit-symmetric-and-width-mates)
10. [Worked example: gear pair](#10-worked-example-gear-pair)
11. [Sub-assemblies](#11-sub-assemblies)
12. [Drawings of assemblies](#12-drawings-of-assemblies)
13. [Limits](#13-limits)

## 1. Tools
| Tool | Use it to |
|---|---|
| `kadvia:new_assembly` | Create an empty assembly (kind `assembly`) in the user's window |
| `kadvia:get_assembly` | Read components, mates and the solve status (call it before editing an assembly you did not just build) |
| `kadvia:apply_assembly_operations` | Add/update/remove components and mates as **one transaction** (one solve, one undo step); `dry_run: true` checks without committing |
| `kadvia:assembly_bom` | Bill of materials: components grouped by source with quantities, material and mass (from the parts' materials, or `density_g_cm3`); `structure`: `top` (default), `indented` or `flat` |
| `kadvia:set_material` with `component_id` | Override the material of one component (for example the same part in steel and in brass); `null` removes the override |
| `kadvia:check_interference` | Overlapping solids between components, part by part inside sub-assemblies (approximate volume and depth) |
| `kadvia:new_drawing` with the assembly as `source` | An assembly drawing with a BOM table and balloons (see [drawings.md](drawings.md)) |
| `kadvia:undo` / `kadvia:redo`, `kadvia:save_part` (writes `.kasm`), `kadvia:export_model`, `kadvia:mass_properties`, `kadvia:measure`, `kadvia:render_views` | Work on assemblies too |

`kadvia:open_step_file` opens a saved `.kasm` file (kind `assembly`).

## 2. Workflow
- [ ] Build or open the parts first (`kadvia:new_part` + `kadvia:apply_operations`, or `kadvia:open_step_file` for `.kadvia`/`.step`). Note their model ids, or save them and use file paths.
- [ ] Read each part's geometry (`kadvia:get_part` bbox, or `kadvia:recognize_features` for hole centres): mate references use the **part's own coordinates**.
- [ ] `kadvia:new_assembly {"name": "..."}`.
- [ ] Batch 1: `add_component` for every part. The first one is grounded (`fixed: true`). Give the others a rough starting `transform.position` near where they go: the solver moves components as little as possible from where they start.
- [ ] Batches 2…n: mates in small groups (one component at a time). After each batch read `solve.status`, `solve.dof`, `underConstrained` and every mate's `status`.
- [ ] `kadvia:render_views` (`iso` plus a view that shows the contact faces).
- [ ] `kadvia:check_interference`, then `kadvia:assembly_bom`.
- [ ] Offer `kadvia:save_part` (`.kasm`; save unsaved parts first so the assembly can reference their files) and `kadvia:export_model`.

## 3. Components
```json
[
  {"op": "add_component", "component": {"id": "plate", "source": {"path": "/Users/ana/cad/plate.kadvia"}, "fixed": true}},
  {"op": "add_component", "component": {"id": "bracket", "name": "Bracket", "source": {"model": "m3"},
    "transform": {"position": [0, 0, 30], "rotation": [0, 0, 0]}}},
  {"op": "add_component", "component": {"id": "motor", "source": {"step": "/Users/ana/cad/motor.step"},
    "transform": {"position": [150, 0, 0]}}}
]
```
- `source` is exactly one of:
  - `path`: a `.kadvia` file (relative paths are relative to the saved `.kasm`);
  - `model`: an open part's model id (stored as its file path if it is saved and unmodified, otherwise embedded);
  - `part`: an embedded part document;
  - `step`: a STEP file (mesh only, see [Limits](#13-limits));
  - a saved `.kasm` as `path` (or `model` with the id of an open, saved and unmodified assembly): a rigid **sub-assembly**, see [Sub-assemblies](#11-sub-assemblies).
- `transform`: `position` [x, y, z] mm and `rotation` [rx, ry, rz] degrees about the fixed world axes X, then Y, then Z (or `quaternion` [x, y, z, w]). World point = R·p + position. The solver rewrites the transforms of mated components.
- Optional: `id`, `name`, `fixed`, `suppressed`, `color` (`"#rrggbb"`).
- `update_component {id, patch}` (transform keys merge), `remove_component {id}` (removes its mates too).

## 4. Mates
A mate reference names a `component` (and optionally a `body`) plus **exactly one** piece of
geometry, in the component's **own** coordinates (the part as modeled, before placement):

| Reference | Resolves to |
|---|---|
| `face`: a face selector (`normal`, `plane`, `near`, `surface`, `radius`, `ids`, …) | planar face → plane (outward normal); cylinder → axis (+ radius); sphere → centre |
| `edge`: an edge selector (`parallel`, `type`, `plane`, `radius`, `near`) | line → axis; circle/arc → circle (centre + axis) |
| `vertex: [x, y, z]` | the nearest vertex |
| `point: [x, y, z]` or `"origin"` | a fixed point of the component |
| `axis: "X" \| "Y" \| "Z"` | datum axis through the component origin |
| `plane: "XY" \| "XZ" \| "YZ"` | datum plane through the component origin (normals +Z, −Y, +X) |

| `type` | Use | `value` / `flip` |
|---|---|---|
| `coincident` | faces touch (plane–plane, normals opposed); point/line/circle on a plane; same point, line or circle | `flip: true`: normals aligned instead of opposed |
| `concentric` | holes, pins, shafts: two axes/cylinders/circles on one line | |
| `distance` | plane–plane gap (along A's normal), point–plane, point–point, axis–axis | `value` in mm (may be an expression); `flip`: aligned normals |
| `angle` | between plane normals or axes | `value` in degrees, 0–180; `flip` reverses B's direction |
| `parallel` / `perpendicular` | plane normals or axes | |
| `tangent` | cylinder or sphere on a plane, cylinder–cylinder, sphere–sphere | `flip`: the other side / inside |
| `fixed` | `a` only: ground a component | |
| `lock` | `a`, `b`: keep the current relative placement (rigid group) | |
| `distance` / `angle` with `min` / `max` | **limit mate**: the distance or angle stays in the range (no `value`) | bounds in mm / degrees; one may be omitted |
| `gear` | `a`, `b` axes (cylinder faces, circular edges, datum axes): B turns `ratio` × A, opposite sense | `ratio` (> 0, required); `flip`: same sense (internal gear, belt) |
| `rack_pinion` | `a` = pinion axis, `b` = rack direction (straight edge or datum axis): the rack moves `value` mm per pinion turn | `value` = π × pitch diameter (required, ≠ 0); `flip` reverses |
| `cam` | `a` = cam face (any face, free-form too), `b` = follower: point, vertex, sphere face or circular edge (its radius rolls on the cam) | `flip`: follower on the inner side |
| `symmetric` | `a`, `b` (two points, planes or axes) are mirror images about plane `c` | `c` required |
| `width` | `a`, `b` = slot faces, `c`, `d` = tab faces: the tab is centred in the slot | `c`, `d` required |

Details and examples for the last six rows: [sections 8–10](#8-motion-mates-gear-rack-and-pinion-cam).

Other operations: `update_mate {id, patch}`, `remove_mate {id}`, `set_parameter` /
`delete_parameter` (assembly parameters usable in mate values), `rename_assembly {name}`,
`set_exploded_view {scale}`, `rebuild` (reload part files that changed on disk).

Patterns that work:
- **Bolt or pin in a hole:** `concentric` (shaft ↔ hole wall) + `coincident` (head underside ↔ seating face). The bolt keeps 1 DOF (spin about its axis), which is fine.
- **Box-like part on a face, fully located:** `coincident` (faces) + `concentric` (a hole pair) + `parallel` (side faces), or `coincident` + two `distance` mates.
- **Hinge:** `concentric` (pin axes) + `coincident` or `distance` (axial position) leaves 1 rotational DOF; add an `angle` mate to pose it.
- If a mate points the wrong way (the part lands upside down or on the far side), set `flip: true` on it with `update_mate`.

Rules:
- Selector coordinates are the **part's** coordinates. Read them from the part (bbox, hole centres from `kadvia:recognize_features`), never from world positions in the assembly.
- Prefer `normal`, `plane`, `surface` + `radius` filters; use `near` with a point **on** the face (for a hole wall: centre + radius along X).
- The batch is strict: if a component cannot load, or a mate cannot be resolved or conflicts with the others, **nothing changes**; the error names `opIndex` and the mate or component (`featureId`).

## 5. Reading the solve report
- `solve.status`: `fully_constrained`, `under_constrained`, `over_constrained` or `failed`; `solve.dof` = remaining degrees of freedom; `underConstrained` lists components that can still move.
- `components[]`: `status`, `fixed`, `dof` (0 = fully positioned), solved `transform`, world `bbox`, `volume`.
- `mates[]`: `status` `ok` / `redundant` / `conflicting` / `error` (with `message`), the resolved `geometry` kinds (for example `["plane", "plane"]`, `["axis", "circle"]`) and `residual`.

| You see | Meaning | Do |
|---|---|---|
| `under_constrained`, dof > 0 | Something can still move | Fine for bolts (spin) and mechanisms; otherwise add the mate that removes the listed motion |
| A mate `redundant` | Already implied by the others | Usually harmless; remove it if it duplicates another |
| A mate `conflicting` / batch refused | It can't be satisfied together with the others | Check the reference faces (wrong face? `flip` needed?) and the `value` |
| A mate `error` | A reference did not resolve, or that geometry pair is not supported | Fix the selector (it must match exactly one face/edge in part coordinates) |
| Geometry kinds unexpected (`point` instead of `axis`) | The selector picked a different face than intended | Use a filter (`surface: "cylinder"`, `normal`) or a better `near` point |

## 6. BOM, interference, exploded view, export
- `kadvia:assembly_bom {"model_id": "m4"}` → items with `item`, `number`, `name`, `source` (`path`, `part`, `step` or `assembly`), `quantity`, component ids, volume per unit (mm³) and mass per unit and `material` from the parts' materials (or from `density_g_cm3`, which overrides them), with `totalMass` (g) when every row has a mass. Suppressed or failed components are listed as excluded. Components with different material overrides form separate rows. Set materials on the parts (`kadvia:set_material`) so the BOM needs no density argument.
- `structure` (assemblies with sub-assemblies): `"top"` (default; a sub-assembly is one row), `"indented"` (its contents listed under its row as 3.1, 3.2, with `level`; nested quantities are per parent unit) or `"flat"` (parts only, quantities summed over every level: what to order).
- `kadvia:check_interference {"model_id": "m4"}` (optional `components: ["bolt"]`) → overlapping pairs with approximate shared `volume` (mm³), deepest overlap `depth` (mm) and the overlap box. Touching faces are not interference. A pin exactly the size of its hole can show a sliver about the tessellation tolerance deep (~0.05 mm); ignore those.
- `{"op": "set_exploded_view", "scale": 1.5}` spreads the components apart in the user's view (display only: positions and mates don't change). `{"scale": 0}` or `null` turns it off.
- `kadvia:export_model` with `.step`: one STEP file with every part solid placed, through every sub-assembly level (the nesting is flattened). STEP components are skipped with a warning. With `.stl`: every instance (including STEP components) in one mesh.
- `kadvia:mass_properties` on the assembly combines all component bodies (world centre of mass).

## 7. Worked example: bracket bolted to a plate
**User:** "Bolt the bracket to the plate with an M6 bolt through the first hole, and check
nothing collides."

Parts (all built with their corner at the origin, Z up):
- Plate `m1`: 100 × 50 × 10, Ø6.6 through holes at (40, 25) and (60, 25).
- Bracket `m2`: foot 40 × 30 × 6 with a Ø6.6 hole at (25, 15), upright 6 × 30 × 40 at X = 0..6.
- Bolt `m3`: shaft radius 3 from Z 0 to 21, head radius 5 from Z 20 to 24.

For example, the bolt:
```json
{"model_id": "m3", "operations": [
  {"op": "add_feature", "feature": {"id": "shaft", "type": "cylinder", "radius": 3, "height": 21}},
  {"op": "add_feature", "feature": {"id": "head", "type": "cylinder", "radius": 5, "height": 4,
    "base": [0, 0, 20], "operation": "add"}}
]}
```

1. `kadvia:new_assembly {"name": "Bracket on plate"}` → model `m4`.
2. Components and mates (one batch is fine here because every reference is simple):

```json
{"model_id": "m4", "operations": [
  {"op": "add_component", "component": {"id": "plate", "source": {"model": "m1"}, "fixed": true}},
  {"op": "add_component", "component": {"id": "bracket", "source": {"model": "m2"}, "transform": {"position": [0, 0, 30]}}},
  {"op": "add_component", "component": {"id": "bolt", "source": {"model": "m3"}, "transform": {"position": [0, 0, 70]}}},
  {"op": "add_mate", "mate": {"id": "on_plate", "type": "coincident",
    "a": {"component": "plate", "face": {"plane": "max_z"}},
    "b": {"component": "bracket", "face": {"normal": "-Z"}}}},
  {"op": "add_mate", "mate": {"id": "hole_align", "type": "concentric",
    "a": {"component": "plate", "face": {"near": [[43.3, 25, 3]]}},
    "b": {"component": "bracket", "face": {"near": [[28.3, 15, 3]]}}}},
  {"op": "add_mate", "mate": {"id": "square", "type": "parallel",
    "a": {"component": "plate", "face": {"normal": "-X"}},
    "b": {"component": "bracket", "face": {"normal": "-X"}}}},
  {"op": "add_mate", "mate": {"id": "bolt_axis", "type": "concentric",
    "a": {"component": "bracket", "face": {"near": [[28.3, 15, 3]]}},
    "b": {"component": "bolt", "face": {"near": [[3, 0, 10]]}}}},
  {"op": "add_mate", "mate": {"id": "bolt_seat", "type": "coincident",
    "a": {"component": "bracket", "face": {"normal": "+Z", "near": [[30, 15, 6]]}},
    "b": {"component": "bolt", "face": {"normal": "-Z", "near": [[4, 0, 20]]}}}}
]}
```

Why these references: `[43.3, 25, 3]` lies on the wall of the plate hole at (40, 25)
(centre + radius 3.3 along X); `[28.3, 15, 3]` on the bracket hole wall; `[3, 0, 10]` on the
bolt shaft; `[4, 0, 20]` on the underside of the head (an annulus between radius 3 and 5).

3. Check the result: the bracket is at position **[15, 10, 10]** (foot on the plate top, its
   hole over the plate hole at x = 40), the bolt at **[40, 25, −4]** (head seated on the foot
   at Z = 16). `solve.dof` = **1**: the bolt can spin, which is expected. Every mate is `ok`.
4. `kadvia:render_views {"views": ["iso", "front"]}`: the foot sits flat on the plate, the
   upright is at the plate's −X side, and the bolt head is on the foot.
5. `kadvia:check_interference {"model_id": "m4"}` → no pairs (the Ø6 shaft sits in Ø6.6
   clearance holes).
6. `kadvia:assembly_bom {"model_id": "m4", "density_g_cm3": 7.85}` → Plate ×1, Bracket ×1,
   Bolt ×1 with masses.
7. Optional: `{"op": "set_exploded_view", "scale": 1}` to show the stack-up, then
   `kadvia:save_part {"model_id": "m4", "path": "~/cad/bracket-on-plate.kasm"}` after saving
   the three parts.

Report (excerpt):
> Assembled the bracket on the plate with one M6 bolt (assembly m4). The bracket is fully
> located (face contact, hole alignment, square to the plate edge); the bolt keeps only its
> spin. No interference. BOM: plate, bracket, bolt, 1 each.

## 8. Motion mates: gear, rack and pinion, cam
These couple **motion** rather than fix a position.
- **Gear** `{"type": "gear", "ratio": r, "a": <axis of A>, "b": <axis of B>}`: B turns `r` × A's rotation, in the opposite sense (external gears). `ratio` = teeth of A / teeth of B (= pitch diameter of A / pitch diameter of B). `flip: true` = same sense (internal gear, belt or chain).
- **Rack and pinion** `{"type": "rack_pinion", "value": v, "a": <pinion axis>, "b": <rack direction>}`: the rack slides `v` mm per pinion revolution. `v` = π × pitch diameter (module m, z teeth: π·m·z). `flip` reverses the direction.
- **Cam** `{"type": "cam", "a": <cam face>, "b": <follower>}`: the follower (a point, a vertex, a sphere face or a circular edge whose radius rolls on the cam) stays on the cam face. `flip`: the follower runs on the inner side.

How they behave:
- They measure each component's rotation or travel **from where it is before the solve**, so they keep the current phase. Place the gears in mesh first, then add the mate.
- They **drive each other**: when one component moves (the user drags it, an `angle` or `distance` mate's `value` changes), the other follows. That is how to pose a mechanism: add an `angle` mate on the driver and change its `value`.
- They don't hold an axis in place. Hold each gear or pinion with other mates (a hinge: `concentric` or a point-on-axis `coincident`, plus a `coincident` face) and the rack with a slide (face contact plus a `distance` to a side plane).
- Rotations are measured in world coordinates, so gears riding on a moving carrier (planetary sets) need the carrier held still while they turn. Change angles in steps below 180° per solve.
- They model motion, not teeth: Kadvia has no gear-tooth feature, so a gear is typically modeled as its pitch-diameter blank. Choose the ratio from the real tooth counts.
- A cam follower is checked against the cam face's fine triangulation, so a curved cam is followed within that tolerance.

```json
[
  {"op": "add_mate", "mate": {"id": "mesh", "type": "gear", "ratio": 0.5,
    "a": {"component": "pinion", "axis": "Z"}, "b": {"component": "wheel", "axis": "Z"}}},
  {"op": "add_mate", "mate": {"id": "drive", "type": "rack_pinion", "value": 62.832,
    "a": {"component": "pinion", "face": {"surface": "cylinder"}}, "b": {"component": "rack", "axis": "X"}}},
  {"op": "add_mate", "mate": {"id": "lift", "type": "cam",
    "a": {"component": "cam", "face": {"near": [[20, 0, 4]]}}, "b": {"component": "follower", "point": "origin"}}}
]
```
(62.832 = π × 20: a 20 mm pitch-diameter pinion.)

## 9. Limit, symmetric and width mates
- **Limit mate:** a `distance` or `angle` mate with `min` and/or `max` instead of `value`. Inside the range it adds no constraint (DOF unchanged, never `redundant`); at a bound it holds like a normal mate, so motion and dragging stop there. Use it for a slider's stroke or a hinge's opening angle. Angle limits stay within 0–180°.
- **Symmetric:** `a` and `b` (two points, planes or axes) are mirror images about the plane `c`. Use it to keep two jaws or two sliders centred.
- **Width:** `a`, `b` are the two inner faces of a slot, `c`, `d` the two faces of a tab: the tab is held parallel and centred in the slot (a clevis and its tongue).

```json
[
  {"op": "add_mate", "mate": {"id": "stroke", "type": "distance", "min": 10, "max": 60,
    "a": {"component": "frame", "plane": "YZ"}, "b": {"component": "slider", "plane": "YZ"}}},
  {"op": "add_mate", "mate": {"id": "mirror", "type": "symmetric",
    "a": {"component": "left", "point": "origin"}, "b": {"component": "right", "point": "origin"},
    "c": {"component": "frame", "plane": "YZ"}}},
  {"op": "add_mate", "mate": {"id": "centred", "type": "width",
    "a": {"component": "clevis", "face": {"normal": "+X", "near": [[10, 0, 5]]}},
    "b": {"component": "clevis", "face": {"normal": "-X", "near": [[30, 0, 5]]}},
    "c": {"component": "tab", "face": {"normal": "-X"}}, "d": {"component": "tab", "face": {"normal": "+X"}}}}
]
```
`value`, `min` and `max` may be expressions over assembly parameters (`set_parameter`).

## 10. Worked example: gear pair
**User:** "Put a 20-tooth pinion and a 40-tooth gear (module 2) on a base plate so they mesh,
and show the gear turned by 30°."

Pitch diameters: 2 × 20 = 40 mm and 2 × 40 = 80 mm, so the centre distance is
(40 + 80)/2 = **60 mm** and the ratio (pinion → gear) is 20/40 = **0.5**.

Parts (Z up), each a blank at its pitch diameter, 8 mm thick, centred on its own Z axis:
- Base `m1`: box 200 × 100 × 10 with its corner at the origin (top face at Z = 10).
- Pinion `m2`: `{"id": "c", "type": "cylinder", "radius": 20, "height": 8}`.
- Gear `m3`: `{"id": "c", "type": "cylinder", "radius": 40, "height": 8}`.

1. `kadvia:new_assembly {"name": "Gear pair"}` → `m4`.
2. Components near their places, hinges and the gear mate:

```json
{"model_id": "m4", "operations": [
  {"op": "add_component", "component": {"id": "base", "source": {"model": "m1"}, "fixed": true}},
  {"op": "add_component", "component": {"id": "pinion", "source": {"model": "m2"}, "transform": {"position": [50, 50, 10]}}},
  {"op": "add_component", "component": {"id": "gear", "source": {"model": "m3"}, "transform": {"position": [110, 50, 10]}}},
  {"op": "add_mate", "mate": {"id": "pinion_axis", "type": "coincident",
    "a": {"component": "base", "point": [50, 50, 10]}, "b": {"component": "pinion", "axis": "Z"}}},
  {"op": "add_mate", "mate": {"id": "pinion_seat", "type": "coincident",
    "a": {"component": "base", "face": {"plane": "max_z"}}, "b": {"component": "pinion", "face": {"normal": "-Z"}}}},
  {"op": "add_mate", "mate": {"id": "gear_axis", "type": "coincident",
    "a": {"component": "base", "point": [110, 50, 10]}, "b": {"component": "gear", "axis": "Z"}}},
  {"op": "add_mate", "mate": {"id": "gear_seat", "type": "coincident",
    "a": {"component": "base", "face": {"plane": "max_z"}}, "b": {"component": "gear", "face": {"normal": "-Z"}}}},
  {"op": "add_mate", "mate": {"id": "mesh", "type": "gear", "ratio": 0.5,
    "a": {"component": "pinion", "axis": "Z"}, "b": {"component": "gear", "axis": "Z"}}}
]}
```
   Each `coincident` point–axis mate puts the part's Z axis through a point on the base: with
   the seat mate that is a hinge. Check: every mate `ok`, `solve.dof` = **1** (the pair turns
   together; turning one turns the other).
3. Pose it with an angle mate on the pinion (driver):

```json
{"model_id": "m4", "operations": [
  {"op": "add_mate", "mate": {"id": "turn", "type": "angle", "value": 30,
    "a": {"component": "base", "plane": "XZ"}, "b": {"component": "pinion", "plane": "XZ"}}}
]}
```
   Check: the pinion turned 30° about Z, the gear **15° the other way**, `solve.dof` = **0**.
   To turn the gear from 30° to 90°, `update_mate {"id": "turn", "patch": {"value": 90}}`
   (steps below 180° per change).
4. `kadvia:render_views {"views": ["top", "iso"]}`. `kadvia:check_interference` may report a
   sliver where the two pitch circles touch; that is tangency on the mesh, not a real overlap.
5. Report: ratio 0.5 (20:40), centre distance 60 mm, one DOF before posing. Remove the `turn`
   mate to leave the pair free for the user to drag.

Variants: a rack instead of the gear → `rack_pinion` with `value` = π × 40 = 125.664 mm per
pinion turn and the rack held on a slide; an internal ring gear → `gear` with `flip: true`.

## 11. Sub-assemblies
A saved `.kasm` used as a component is a **rigid** sub-assembly: its own mates are solved
inside it, and it moves in the parent as one body.

```json
[
  {"op": "add_component", "component": {"id": "s1", "source": {"path": "/Users/ana/cad/bracket-on-plate.kasm"},
    "transform": {"position": [10, 10, 10]}}},
  {"op": "add_mate", "mate": {"id": "seat", "type": "coincident",
    "a": {"component": "base", "face": {"plane": "max_z"}},
    "b": {"component": "s1", "face": {"normal": "-Z", "near": [[50, 25, 0]]}}}}
]
```
- Source: `{"path": "<.kasm>"}`, or `{"model": "<open assembly id>"}` once that assembly is saved and has no unsaved changes.
- Mate references into a sub-assembly use **the sub-assembly's own coordinates**. Its bodies are named `<child>.<body>` (`plate.b0`; nested: `<child>.<grandchild>.<body>`). Without `body`, a `near` or `vertex` pick chooses the nearest body; `"body": "plate.b0"` names one explicitly.
- To change a sub-assembly, open its `.kasm`, edit, save; the parent reloads it (`{"op": "rebuild"}` forces a reload).
- `kadvia:assembly_bom` with `structure: "indented"` lists its contents under its row (2.1, 2.2, …); `"flat"` adds its parts to the totals.
- `kadvia:check_interference` checks its parts one by one (pairs named like `"s1/bolt"`); `components: ["s1"]` checks everything inside `s1`.
- STEP export places every part through all levels.
- An assembly cannot contain itself (directly or through its sub-assemblies); at most 8 nesting levels. Component errors inside a sub-assembly show in that component's `message`.

Example: two bracket-on-plate sub-assemblies on a base. `structure: "top"` → 1 Base ×1,
2 Bracket on plate ×2. `"indented"` → the same plus 2.1 Plate, 2.2 Bracket, 2.3 Bolt (×1 per
sub-assembly). `"flat"` → Base ×1, Plate ×2, Bracket ×2, Bolt ×2.

## 12. Drawings of assemblies
`kadvia:new_drawing {"source": "<assembly id>"}` makes an assembly drawing: hidden lines
across components, overall dimensions, a BOM table and balloons (item numbers from the BOM),
optional exploded views and extra sheets. Save the assembly (and its parts) before saving the
drawing. See [drawings.md](drawings.md#15-worked-example-assembly-drawing-with-bom-and-balloons).

## 13. Limits
- Sub-assemblies are rigid: their internal mates can't move inside the parent (flexible sub-assemblies are not supported yet). Edit and save the sub-assembly's file to change it.
- STEP components are mesh only: mate references on them support only `near` and `ids`, and STEP export skips them (STL export includes them).
- Gear and rack-and-pinion mates relate motion, not tooth geometry; their axes must be held by other mates; planetary sets need the carrier held while the gears turn; change angles in steps below 180°.
- Cam followers follow the cam face's triangulation, within its tolerance.
- `kadvia:check_design` does not run on assemblies: check the component parts, and use `kadvia:check_interference` for the assembly.
- Interference is computed on the display meshes, so volumes and depths are approximate.
- Material overrides apply to a whole component; overrides on components inside a rigid sub-assembly are not applied when it is placed. STEP components have no material (no mass in the BOM unless `density_g_cm3` is given).
