# Assemblies (`.kasm`)

An assembly places **components** (Kadvia parts or STEP files) in space and constrains them
with **mates**. Kadvia solves the mates, reports what is still free to move, lists a bill of
materials and checks for interference. The live reference is
`kadvia:modeling_reference {"topic": "assemblies"}`.

## Contents
1. [Tools](#1-tools)
2. [Workflow](#2-workflow)
3. [Components](#3-components)
4. [Mates](#4-mates)
5. [Reading the solve report](#5-reading-the-solve-report)
6. [BOM, interference, exploded view, export](#6-bom-interference-exploded-view-export)
7. [Worked example: bracket bolted to a plate](#7-worked-example-bracket-bolted-to-a-plate)
8. [Limits](#8-limits)

## 1. Tools
| Tool | Use it to |
|---|---|
| `kadvia:new_assembly` | Create an empty assembly (kind `assembly`) in the user's window |
| `kadvia:get_assembly` | Read components, mates and the solve status (call it before editing an assembly you did not just build) |
| `kadvia:apply_assembly_operations` | Add/update/remove components and mates as **one transaction** (one solve, one undo step); `dry_run: true` checks without committing |
| `kadvia:assembly_bom` | Bill of materials: components grouped by source with quantities (and masses with `density_g_cm3`) |
| `kadvia:check_interference` | Overlapping solids between components (approximate volume and depth) |
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
  - `step`: a STEP file (mesh only, see [Limits](#8-limits)).
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
- `kadvia:assembly_bom {"model_id": "m4", "density_g_cm3": 2.70}` → items with `item`, `name`, `source`, `quantity`, component ids, volume per unit (mm³) and, with a density, mass per unit and `totalMass` (g). Suppressed or failed components are listed as excluded. Different materials: call it once per density, or compute per item.
- `kadvia:check_interference {"model_id": "m4"}` (optional `components: ["bolt"]`) → overlapping pairs with approximate shared `volume` (mm³), deepest overlap `depth` (mm) and the overlap box. Touching faces are not interference. A pin exactly the size of its hole can show a sliver about the tessellation tolerance deep (~0.05 mm); ignore those.
- `{"op": "set_exploded_view", "scale": 1.5}` spreads the components apart in the user's view (display only: positions and mates don't change). `{"scale": 0}` or `null` turns it off.
- `kadvia:export_model` with `.step`: one STEP file with every part solid placed. STEP components are skipped with a warning. With `.stl`: every instance (including STEP components) in one mesh.
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

## 8. Limits
- No sub-assemblies (a `.kasm` cannot be a component) in this version.
- STEP components are mesh only: mate references on them support only `near` and `ids`, and STEP export skips them (STL export includes them).
- `kadvia:check_design` does not run on assemblies: check the component parts, and use `kadvia:check_interference` for the assembly.
- Interference is computed on the display meshes, so volumes and depths are approximate.
- 2D drawings of assemblies are not supported yet.
