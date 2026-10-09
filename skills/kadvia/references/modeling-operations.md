# Modeling operations reference (Kadvia part model v1)

Everything `kadvia:apply_operations` accepts. The same content is available live from
`kadvia:modeling_reference`. Snippets use illustrative parameter names (`width`, `thickness`,
...); define your own with `set_parameter` first.

## Contents
1. [Document model](#1-document-model)
2. [Operations](#2-operations)
3. [Expressions](#3-expressions)
4. [Sketch planes and frames](#4-sketch-planes-and-frames)
5. [Profiles](#5-profiles)
6. [Features](#6-features): extrude, revolve, box, cylinder, sphere, hole, fillet, chamfer, linear_pattern, circular_pattern, mirror
7. [Edge selectors](#7-edge-selectors)
8. [Bodies and booleans](#8-bodies-and-booleans)
9. [Gotchas](#9-gotchas)

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
| `update_feature` | `id`, `patch` | **Shallow merge**: a patched `plane`, `profiles`, `points` or `edges` replaces the whole value |
| `delete_feature` | `id` | |
| `move_feature` | `id`, `after?` | Omit `after` to move it to the front |
| `suppress_feature` | `id`, `suppressed` | |
| `set_parameter` | `name`, `value?`, `unit?`, `min?`, `max?`, `description?` | Creates the parameter if it is missing |
| `delete_parameter` | `name` | Fails while the parameter is still used |
| `rename_part` | `name` | |

- The batch is **one transaction**: everything applies, the part regenerates once, and the change is one undo step.
- Any invalid op or failing feature means **nothing changes**. The error names `operations[i]` and the feature id.
- Ops run in order, so define parameters before the features that use them, and a sketch before its extrude.
- `dry_run: true` validates and regenerates without committing.
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
A plane is `{"base": "XY" | "XZ" | "YZ", "offset"?: Expr}`. 2D points in sketches and holes are
`[u, v]` in the plane's frame. `offset` moves the plane along its **normal**.

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

### fillet / chamfer
```json
{"id": "corner_fillet", "type": "fillet", "edges": {"parallel": "Z"}, "radius": "corner_r"}
```
```json
{"id": "top_chamfer", "type": "chamfer", "edges": {"plane": "max_z", "type": "line"}, "distance": 0.5}
```

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
- You can pattern `extrude` (add or cut), `hole` and primitive features.
- `count` **includes the original**.
- `circular_pattern` with the default `angle` (360) spaces the copies evenly.

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
