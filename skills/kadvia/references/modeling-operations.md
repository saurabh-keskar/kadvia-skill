# Modeling operations reference (Kadvia part model)

Everything `kadvia:apply_operations` accepts. The same content is available live from
`kadvia:modeling_reference`. Snippets use illustrative parameter names (`width`, `thickness`,
...); define your own with `set_parameter` first.

## Contents
1. [Document model](#1-document-model)
2. [Operations](#2-operations)
3. [Expressions](#3-expressions)
4. [Sketch planes and frames](#4-sketch-planes-and-frames)
5. [Profiles](#5-profiles)
6. [Features](#6-features): extrude, revolve, box, cylinder, sphere, hole, thread, fillet, chamfer, linear_pattern, circular_pattern, mirror, plane, loft, sweep, shell, draft
7. [Edge selectors](#7-edge-selectors)
8. [Bodies and booleans](#8-bodies-and-booleans)
9. [Gotchas](#9-gotchas)
10. [Face selectors](#10-face-selectors)
11. [Rollback bar](#11-rollback-bar)
12. [Constrained sketches](#12-constrained-sketches)
13. [Known limits](#13-known-limits)

Related: reference images (`add_reference` …) in [image-to-cad.md](image-to-cad.md#8-reference-images-in-kadvia);
direct edits on imported geometry (`delete_face`, `offset_face`, `move_face`, `resize_hole`,
`import`) in [editing-step.md](editing-step.md); standard holes and the `thread` feature in
[holes-threads.md](holes-threads.md); splines, ellipses, slots, text, `edit_sketch` and DXF
import in [sketch-tools.md](sketch-tools.md); `set_material` / `set_appearance` in
[materials-appearance.md](materials-appearance.md).

## 1. Document model
A part is `{schema: "kadvia.part/1", name, units: "mm", parameters: [...], features: [...]}`.
Kadvia replays the features in order (regeneration) every time something changes.

- **Parameter:** `{name, value, unit?, min?, max?, description?}`.
  - `name` matches `[A-Za-z_][A-Za-z0-9_]*` and must be unique.
  - `value` is a number or an expression.
  - `unit` is `"mm"` (default), `"deg"`, or `""` for counts and ratios.
  - `min`/`max` only set the slider range in the UI.
- **Feature:** `{id, type, name?, suppressed?, ...type fields}`.
  - `id` is your own unique string. Always set it.
  - `suppressed: true` skips the feature during regeneration.
- **Regeneration report** (returned by every tool):
  - `revision`
  - per-feature status: `ok`, `ERROR: message`, `suppressed` or `skipped`
  - bodies: id, bbox `{min, max, size}`, volume, face/edge counts
  - `changes` compared with the previous state

## 2. Operations
`kadvia:apply_operations {"model_id": "...", "operations": [...], "dry_run"?: bool}`

| `op` | Fields | Notes |
|---|---|---|
| `add_feature` | `feature`, `after?` | Appends by default; `after` = id of the feature to insert after |
| `update_feature` | `id`, `patch` | **Shallow merge**: a patched `plane`, `profiles`, `entities`, `constraints`, `dimensions`, `points`, `edges`, `faces`, `openFaces`, `path` or `sections` replaces the whole value |
| `delete_feature` | `id` | |
| `move_feature` | `id`, `after?` | Omit `after` to move it to the front |
| `suppress_feature` | `id`, `suppressed` | |
| `set_parameter` | `name`, `value?`, `unit?`, `min?`, `max?`, `description?` | Creates the parameter if it is missing |
| `delete_parameter` | `name` | Fails while the parameter is still used |
| `rename_part` | `name` | |
| `set_rollback` | `to?` | Rollback bar: only features up to `to` regenerate (`"@start"` = none). Omit `to` to roll forward to the end. See [section 11](#11-rollback-bar) |
| `add_reference` / `update_reference` / `remove_reference` | `reference` / `id`, `patch` / `id` | Reference images for tracing (never affect geometry) |
| `edit_sketch` | `id`, `edits` | Sketch edits (trim, offset, fillet, mirror, patterns, …) on a stored sketch; see [sketch-tools.md](sketch-tools.md#5-editing-sketches-edit_sketch) |
| `set_material` / `set_appearance` | `material`, `body?` / `appearance`, `body?`, `faces?`, `replace?`, `clearFaces?` | Material (mass) and look (display only); see [materials-appearance.md](materials-appearance.md#8-operations-inside-apply_operations) |

- The batch is **one transaction**: everything applies, the part regenerates once, and the change is one undo step.
- Any invalid op or failing feature means **nothing changes**. The error names `operations[i]` and the feature id.
- Ops run in order, so define parameters before the features that use them, and a sketch before its extrude.
- `dry_run: true` validates and regenerates without committing.
- At most 200 operations per call; keep batches much smaller (one logical step).
- `kadvia:set_parameters {"model_id": "...", "values": {"width": 120}}` is shorthand for `set_parameter` ops.

```json
[
  {"op": "set_parameter", "name": "slot_w", "value": 6, "min": 3, "max": 12, "description": "Slot width"},
  {"op": "update_feature", "id": "base", "patch": {"distance": "thickness + 2"}},
  {"op": "suppress_feature", "id": "edge_fillet", "suppressed": true},
  {"op": "move_feature", "id": "pocket", "after": "base"},
  {"op": "delete_feature", "id": "old_hole"},
  {"op": "rename_part", "name": "Sensor bracket"}
]
```

## 3. Expressions
- **Allowed:** `+ - * / ^ ( )`, unary minus, numbers (`2.5`, `1e3`), parameter names, `pi`, and the functions `sqrt abs min max sin cos tan asin acos atan atan2 round floor ceil`. Trig uses **degrees**.
- **Not allowed:** units inside strings (`"10mm"`), `%`, conditionals.
- **Errors:** undefined names and circular references.

Examples: `"width/2 - hole_inset"`, `"2*wall + pcb_w"`, `"pcd/2 * cos(30)"`,
`"max(1, wall/2)"`, `"floor(length / pitch) + 1"`.

## 4. Sketch planes and frames
A plane is one of these (all take `"offset"?: Expr`, which moves the plane along its **normal**):
- `{"base": "XY" | "XZ" | "YZ"}`: the standard planes (table below).
- `{"face": FaceSelector, "body"?: "b0"}`: a **planar face** of a body (default the last body). The selector must leave exactly one planar face. Normal = the face's outward normal; origin = the world origin projected onto the face; u = world +X projected onto the plane (world +Y if the normal is close to ±X). So the top face of a block has the same `[u, v]` as `XY`. A face plane follows the face when parameters change.
- `{"origin": [x, y, z], "normal": [x, y, z], "xDir"?: [x, y, z]}`: explicit.
- `{"ref": "datum1"}`: a reference `plane` feature.

2D points in sketches and holes are `[u, v]` in the plane's frame. `hole`, `mirror` and
reference images accept the same plane forms.

| base | u | v | normal | plane at offset `o` | "front" of the plane faces |
|---|---|---|---|---|---|
| `XY` | +X | +Y | +Z | Z = o | up (top view) |
| `XZ` | +X | +Z | −Y | **Y = −o** | the viewer of the front view |
| `YZ` | +Y | +Z | +X | X = o | the viewer of the right view |

```
            Z (up)
            |
            |   XZ plane = front wall        YZ plane = side wall
            |   u=+X, v=+Z, normal -Y        u=+Y, v=+Z, normal +X
            |
            +------------ Y   (front view looks along +Y)
           /
          /   XY plane = floor: u=+X, v=+Y, normal +Z
         X
```

- `XY` offset `h` is the horizontal plane at height `h`, for example the top face of a plate of thickness `h`.
- `XZ` offset `d` is the vertical plane at **Y = −d**. The front face of a part centred on Y = 0 with depth `D` is `XZ` offset `D/2`.
- `YZ` offset `x` is the vertical plane at X = x.

## 5. Profiles
Profiles are closed shapes. A profile completely inside another profile of the same sketch becomes a hole.

| kind | Fields |
|---|---|
| `rect` | `width` (along u), `height` (along v), plus `center` **or** `corner` (smallest u, v). Optional `cornerRadius` |
| `circle` | `center`, plus `radius` **or** `diameter` |
| `polygon` | `center`, `sides`, `radius` (centre to vertex), optional `rotation` in degrees |
| `slot` | `center`, `length` (between the end-arc centres), `width`, optional `angle` (degrees from +u) |
| `path` | `start`, `segments`: `{"line": [u, v]}` or `{"arc": {"through": [u, v], "to": [u, v]}}`. Closes back to `start` automatically |

```json
{"id": "plate_sk", "type": "sketch", "plane": {"base": "XY"},
 "profiles": [
   {"kind": "rect", "center": [0, 0], "width": "width", "height": "depth", "cornerRadius": "corner_r"},
   {"kind": "circle", "center": [0, 0], "diameter": "bore_d"},
   {"kind": "slot", "center": [0, "depth/2 - 10"], "length": 20, "width": 6},
   {"kind": "polygon", "center": ["width/2 - 12", 0], "sides": 6, "radius": 5}
 ]}
```

```json
{"id": "l_sk", "type": "sketch", "plane": {"base": "XZ"},
 "profiles": [{"kind": "path", "start": [0, 0], "segments": [
   {"line": ["leg_x", 0]}, {"line": ["leg_x", "t"]}, {"line": ["t", "t"]},
   {"line": ["t", "leg_z"]}, {"line": [0, "leg_z"]}
 ]}]}
```

```json
{"id": "tab_sk", "type": "sketch", "plane": {"base": "XY"},
 "profiles": [{"kind": "path", "start": [-10, 0], "segments": [
   {"line": [10, 0]}, {"line": [10, 15]},
   {"arc": {"through": [0, 25], "to": [-10, 15]}}
 ]}]}
```

## 6. Features

### extrude
```json
{"id": "base", "type": "extrude", "sketch": "base_sk", "distance": "thickness"}
```
- `distance` > 0.
- `direction`: `"normal"` (default, along the plane normal), `"reverse"` or `"symmetric"` (half to each side).
- `through: true` cuts through everything (ignores `distance`); use it with `"operation": "cut"`.
- `operation`: `"new"` for the first body, otherwise `"add"`. `target` is a body id.
- `draftAngle` (degrees, |angle| < 89): tapers the extrude. Positive leans every wall inward moving away from the sketch plane (the outer profile shrinks, holes grow); negative leans outward. A side of length `L` extruded `d` at a positive angle `a` ends at `L − 2·d·tan(a)`. Not with `through`.

```json
{"id": "pocket", "type": "extrude", "sketch": "pocket_sk", "distance": "pocket_depth",
 "direction": "reverse", "operation": "cut"}
```

### revolve
```json
{"id": "shaft", "type": "revolve", "sketch": "half_profile_sk",
 "axis": {"origin": [0, 0], "direction": [0, 1]}, "angle": 360}
```
- The axis is given in the sketch's `[u, v]` coordinates. On an `XZ` sketch, `direction: [0, 1]` is the Z axis.
- Sketch only half of the profile, on one side of the axis.

### box / cylinder / sphere
```json
{"id": "block", "type": "box", "size": ["length", "width", "height"], "center": [0, 0, "height/2"]}
```
```json
{"id": "boss", "type": "cylinder", "radius": "boss_d/2", "height": "boss_h",
 "base": [0, 0, "thickness"], "axis": "Z", "operation": "add"}
```
```json
{"id": "ball", "type": "sphere", "radius": 10, "center": [0, 0, 30], "operation": "add"}
```
- `box`: `center` or `corner` (minimum corner); the default is `corner [0, 0, 0]`.
- `cylinder`: `base` is the centre of the start face (default origin); it grows along +`axis` (default `"Z"`).
- `sphere`: `radius`, `center`. Spheres work with `add`/`cut`/`intersect` against boxes, cylinders and other spheres: pockets, domes (a sphere of the cylinder's radius on its top face), capsules, holes drilled through a ball, caps cut off. Avoid a sphere that touches another body in a single point, and two coincident spheres.

### hole
```json
{"id": "mount_holes", "type": "hole", "plane": {"base": "XY", "offset": "thickness"},
 "points": [["-hole_x", "-hole_y"], ["hole_x", "-hole_y"], ["hole_x", "hole_y"], ["-hole_x", "hole_y"]],
 "diameter": "hole_d", "through": true,
 "counterbore": {"diameter": "cb_d", "depth": "cb_depth"}}
```
- Drills from the plane along the **negative normal**:
  - `XY` at the top face drills down.
  - `XZ` drills toward +Y.
  - `YZ` drills toward −X.
- Put the plane **on the entry face**.
- Give `depth` or `through: true`. `counterbore` is optional. A hole always cuts its `target`.
- **For screws use the hole wizard** instead of typed diameters: `"kind": "tapped"`, `"clearance"`, `"counterbore"`, `"countersink"` or `"counterdrill"` with `"size": "M6"` (or `"1/4-20 UNC"`); `diameter` then comes from the tables. Fields, tables and examples: [holes-threads.md](holes-threads.md).

```json
{"id": "m5_holes", "type": "hole", "plane": {"face": {"normal": "+Z", "plane": "max_z"}},
 "points": [[-20, -10], [20, -10]], "kind": "tapped", "size": "M5", "threadDepth": 10}
```

### thread
```json
{"id": "shaft_thread", "type": "thread", "faces": {"surface": "cylinder", "concave": false, "diameter": 10}, "length": 20}
```
- A cosmetic (default) or modeled (`"modeled": true`) thread on the cylindrical faces of one hole or shaft; see [holes-threads.md](holes-threads.md#5-thread-feature).

### fillet / chamfer
```json
{"id": "corner_fillet", "type": "fillet", "edges": {"parallel": "Z"}, "radius": "corner_r"}
```
```json
{"id": "top_chamfer", "type": "chamfer", "edges": {"plane": "max_z", "type": "line"}, "distance": 0.5}
```
- Constant radius / equal distance.
- **Corners work:** `{"all": true}` on a block or prism (three convex edges meet in a spherical corner patch; chamfers meet in a point), pocket corners where three concave edges meet, the edges around one face, tangent chains (rounded-rectangle outlines), rims of revolved parts, and concave edges such as boss bases and the inside of a bracket.
- **Refused** (`geometry` error): corners where three or more selected **curved** edges meet; corners where selected convex and concave edges meet (for example `{"all": true}` on an L-bracket: fillet the convex and the concave edges in separate features); more than three selected edges at one vertex; sizes too large for the neighbouring faces.

### linear_pattern / circular_pattern / mirror
```json
{"id": "hole_row", "type": "linear_pattern", "features": ["hole_1"], "direction": [1, 0, 0],
 "count": "hole_count", "spacing": "pitch"}
```
```json
{"id": "bolt_ring", "type": "circular_pattern", "features": ["bolt_hole"],
 "axis": {"origin": [0, 0, 0], "direction": [0, 0, 1]}, "count": "bolt_count"}
```
```json
{"id": "rib_mirror", "type": "mirror", "features": ["rib"], "plane": {"base": "YZ", "offset": 0}}
```
- You can pattern `extrude` (add or cut), `hole`, primitive, `loft` and `sweep` features. Patterns can't repeat other patterns or mirrors; `shell`, `draft`, `import` and the direct edits can't be patterned or mirrored.
- `count` **includes the original**.
- `circular_pattern` with the default `angle` (360) spaces the copies evenly.

### plane (reference plane)
```json
{"id": "datum1", "type": "plane", "face": {"plane": "max_z"}, "offset": 15}
```
- The plane fields written flat (`base` / `face` + `body` / `origin` + `normal` + `xDir` / `ref`, plus `offset`). It creates no geometry; sketches, holes and mirrors use it with `{"ref": "datum1"}`.

### loft
```json
{"id": "transition", "type": "loft", "sections": ["sec_bottom", "sec_mid", "sec_top"], "ruled": false}
```
- `sections`: 2 or more sketch ids in loft order (for example `XY` sketches at increasing `offset`). Sections must not intersect each other.
- Each section has exactly **one region** (holes are allowed if every section has the same number).
- `ruled: false` (default) is smooth through all sections; `true` is straight between consecutive sections.
- For predictable results use the same profile kind or segment count in every section (circles, rects and rounded rects map well).

### sweep
```json
{"id": "pipe", "type": "sweep", "sketch": "pipe_sk",
 "path": {"plane": {"base": "XY"}, "start": [0, 0],
          "segments": [{"line": [0, 50]}, {"arc": {"through": [5.858, 64.142], "to": [20, 70]}}]}}
```
```json
{"id": "coil", "type": "sweep", "sketch": "wire_sk",
 "path": {"helix": {"radius": "coil_r", "pitch": "pitch", "turns": "turns", "axis": "Z"}}}
```
- `sketch`: the profile, one region (a ring sweeps a tube). Draw it **at the start of the path, crossing it** (normally perpendicular to the first segment).
- `path`: either `segments` (an **open** chain of lines/arcs in a plane, like a sketch `path` profile; junctions must be tangent except line→line corners, which get a mitre) or `helix` (`radius`, `pitch`, `turns`, optional `axis`, `origin`, `leftHanded`). A Z-axis helix starts at [radius, 0, 0] heading about +Y, so its profile goes on `XZ` centred at `[radius, 0]`.
- `orientation`: `"frenet"` (default, the profile turns with the path) or `"fixed"`.
- `operation: "cut"` sweeps a groove.

### shell
```json
{"id": "hollow", "type": "shell", "thickness": "wall", "openFaces": {"normal": "+Z"}}
```
- Offsets every face of the target body **inward** by `thickness`. `openFaces` (a face selector) removes faces, for example the top of an enclosure; omit it for a closed hollow body.
- Fillet outer edges **before** the shell (radius > thickness); cut holes and add bosses after it.

### draft
```json
{"id": "side_draft", "type": "draft", "faces": {"normal": "+X"}, "angle": 3, "pull": "+Z"}
```
- Tilts existing **planar** faces (molding draft). Positive `angle` narrows the part along `pull` (default `"+Z"`). `neutral` = the coordinate of the hinge plane on the pull axis (default the body's minimum along `pull`).
- Draft walls **before** filleting their edges. For a sketched solid, `draftAngle` on the extrude is simpler.

## 7. Edge selectors
Used by `fillet.edges` and `chamfer.edges`. Criteria combine with **AND**. `near` and `ids`
select a union of the listed edges. Zero matches is an error.

| Field | Meaning |
|---|---|
| `all: true` | Every edge of the target body |
| `parallel: "X" \| "Y" \| "Z"` | Straight edges parallel to that axis |
| `type: "line" \| "circle" \| "curve"` | Edge geometry |
| `plane: "max_x" … "min_z"` | Edges lying on that face of the body's bounding box |
| `radius: Expr` | Circular edges of that radius (±0.01 mm) |
| `near: [[x, y, z], ...]` | The edge closest to each point (expressions allowed) |
| `ids: [n, ...]` | Edge ids of the current regeneration; fragile, avoid |

| Intent | Selector |
|---|---|
| Vertical outer corners of a block | `{"parallel": "Z"}` (before adding holes/bosses) |
| Every edge of the top face | `{"plane": "max_z"}` |
| Straight top edges only | `{"plane": "max_z", "type": "line"}` |
| Vertical edges on the +X side | `{"parallel": "Z", "plane": "max_x"}` |
| Both rims of Ø8 holes | `{"type": "circle", "radius": 4}` |
| Rims of Ø8 holes on the top face | `{"type": "circle", "radius": 4, "plane": "max_z"}` |
| Inside bend of an L profile | `{"near": [["t", 0, "t"]]}` |

## 8. Bodies and booleans
- Features with `operation` (extrude, revolve, primitives) create (`"new"`) or modify (`"add"`, `"cut"`, `"intersect"`) a body. The default `target` is the last body.
- The first solid defaults to `"new"`; later ones default to `"add"`. Write `"operation": "cut"` explicitly for removals.
- `"add"`/`"cut"`/`"intersect"` before any body exists is an error.
- Body ids are `b0`, `b1`, ... Read them from the result or `kadvia:get_part`.
- Prefer one body. Use extra `"new"` bodies only for multi-body designs the user asked for.

## 9. Gotchas
- **XZ plane:** the normal is −Y, so a positive offset moves the plane to −Y, and a default extrude grows toward −Y. Use `"direction": "reverse"` or `"symmetric"` when needed.
- **Holes drill against the normal.** An XY hole plane at Z = 0 drills downward, into nothing. Put the plane on the top face (`offset` = top height).
- **Selectors see the body as built so far.**
  - Fillet the outer vertical corners before adding holes and bosses, because cylindrical faces may add straight seam edges along their axis.
  - Do small edge breaks last.
- **Fillet too large:** the radius must be less than the adjacent face widths (for example less than half a plate's thickness for a full round).
- **Coplanar faces:** cuts that end exactly on a face, or bosses that only touch a wall, can make slivers or fail. Overlap by 0.5–1 mm, or make cuts go through.
- **update_feature is shallow.** Patching `{"plane": {"offset": 5}}` drops `base`; send the whole object (`{"plane": {"base": "XY", "offset": 5}}`).
- **Give ids.** Without them you can't reference a sketch from an extrude in the same batch.
- **Fillet corners in separate features** when convex and concave edges meet (see the fillet notes above).

## 10. Face selectors
Used by `shell.openFaces`, `draft.faces`, sketch/hole `plane: {"face": ...}`, assembly mates
and the direct edits. Filters combine with **AND**; `near` and `ids` pick among the filtered
faces. Zero matches is an error.

| Field | Meaning |
|---|---|
| `normal: "+X" … "-Z"` | Planar faces whose outward normal points that way (±0.5°) |
| `plane: "max_x" … "min_z"` | Faces lying on that face of the body's bounding box |
| `surface` | `"plane"`, `"cylinder"`, `"cone"`, `"sphere"`, `"torus"`, `"freeform"` |
| `radius` / `diameter` | Cylinders/spheres of that size (tori: the tube radius), ±0.01 mm |
| `concave` | Curved faces: `true` = hole walls, inside fillets; `false` = bosses, rounds |
| `axis: "X" \| "Y" \| "Z"` | Cylinders, cones, tori whose axis is parallel |
| `near: [[x, y, z], ...]` | The face closest to each point (expressions allowed) |
| `ids: [n, ...]` | Face ids of the current regeneration (fragile, avoid) |

| Intent | Selector |
|---|---|
| Top face of a box | `{"normal": "+Z", "plane": "max_z"}` (`normal` alone also matches upward faces inside the part) |
| Bottom face | `{"normal": "-Z"}` |
| Only the outer right wall | `{"normal": "+X", "plane": "max_x"}` |
| All Ø6 hole walls | `{"surface": "cylinder", "concave": true, "diameter": 6}` |

## 11. Rollback bar
`{"op": "set_rollback", "to": "<feature id>"}` moves the rollback bar below that feature: later
features stay in the part but are not regenerated (status `skipped`, message "rolled back").
`"to": "@start"` rolls back everything; `{"op": "set_rollback"}` (no `to`) rolls forward to the
end. The bar is saved with the part and is part of undo/redo.

- While rolled back, `add_feature` **without** `after` inserts at the bar (not at the end) and moves the bar below the new feature, so it regenerates.
- Deleting the feature at the bar moves the bar to the previous feature.
- Uses: insert a feature before the fillets without computing `after`; look at (render, measure) an earlier state; find which feature broke something.
- **Always roll forward when you're done**, and tell the user if you leave the part rolled back (it looks unfinished).

```json
[
  {"op": "set_rollback", "to": "base"},
  {"op": "add_feature", "feature": {"id": "rib", "type": "box", "size": [4, 40, 20], "center": [0, 0, 10], "operation": "add"}},
  {"op": "set_rollback"}
]
```
(Adds `rib` right after `base`, before the holes and fillets that follow it, then rolls
forward so everything regenerates.)

## 12. Constrained sketches
Sketches drawn by the user in the sketcher are stored as `entities` (point, line, arc,
circle), `constraints` (coincident, horizontal, vertical, parallel, perpendicular, tangent,
equal, midpoint, concentric, fix, symmetric, point_on_line, point_on_circle) and driving
`dimensions` (distance, horizontal_distance, vertical_distance, length, radius, diameter,
angle) whose `value` can be a parameter expression. You can write them too, but `profiles`
remain the simpler path for standard shapes. Sketches also take `spline`, `ellipse`,
`elliptical_arc`, `slot`, `arc_slot`, `text` and `conic` entities, construction geometry, more
relations (`curvature`, …) and dimensions (`major_radius`, `minor_radius`, `width`), and can be
edited with `edit_sketch`: see [sketch-tools.md](sketch-tools.md).
- The result lists each sketch under `sketches` with `status` (`fully_constrained`, `under_constrained`, `over_constrained`, `failed`), `dof`, redundant and conflicting constraint ids.
- `kadvia:solve_sketch {"model_id": "...", "sketch": {...}}` solves a sketch against the part's parameters **without** changing the part: use it to debug before `add_feature`.
- Read `kadvia:modeling_reference {"topic": "constraints"}` for the exact fields before editing one.

## 13. Known limits
What the current version does not do (say so instead of improvising):
- **Fillets/chamfers:** constant size only; refused at corners of three or more curved edges and at mixed convex/concave corners (split into separate features).
- **Spheres:** no single-point (tangent) contact with another body; no two coincident spheres.
- **Patterns/mirrors:** can't repeat patterns or mirrors; `shell`, `draft`, `import` and direct edits can't be patterned or mirrored.
- **Shell:** thickness must be smaller than the smallest convex radius and less than half the thinnest section; open faces must not be tangent to a face that stays closed; faces with a pole (spheres, cone tips) can't be shelled yet.
- **Draft:** planar faces only, not perpendicular to the pull direction.
- **Loft:** one region per section. **Sweep:** tangent junctions (or line→line mitres); a helix profile must fit within one pitch and not reach the axis.
- **Sheet metal:** no bend/flange/unfold features; model a uniform-thickness solid (see [design-rules.md](design-rules.md)).
- **Threads:** cosmetic by default; modeled threads are slow and may fail where they cross other features. No pipe, tapered or multi-start threads (see [holes-threads.md](holes-threads.md#10-limits)).
- **Sketch curves:** splines, ellipses and conics become fine arcs in solids and STEP (see [sketch-tools.md](sketch-tools.md#12-limits)).
- **Imported STEP:** see [editing-step.md](editing-step.md#8-honest-limits); assemblies, drawings and the design check have their own limits in [assemblies.md](assemblies.md), [drawings.md](drawings.md) and [design-check.md](design-check.md).
