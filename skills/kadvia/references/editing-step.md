# Editing imported STEP files

A STEP file opens as kind `imported`: you can view, measure, section and export it, but not
change it. To edit it, convert it to a part. The part's first feature holds the file's solids,
and every later change is a normal, undoable feature, so you can edit, suppress, reorder or
roll back each edit. The live reference is `kadvia:modeling_reference {"topic": "imported"}`.

## Contents
1. [Workflow](#1-workflow)
2. [recognize_features](#2-recognize_features)
3. [Direct-edit features](#3-direct-edit-features)
4. [Face selectors for imported geometry](#4-face-selectors-for-imported-geometry)
5. [Regular features on imported faces](#5-regular-features-on-imported-faces)
6. [Worked example: make all Ø6 holes Ø6.6 and remove the fillets](#6-worked-example-make-all-ø6-holes-ø66-and-remove-the-fillets)
7. [Inserting a STEP file into a part](#7-inserting-a-step-file-into-a-part)
8. [Honest limits](#8-honest-limits)

## 1. Workflow
- [ ] `kadvia:open_step_file {"path": "..."}` → an imported model (for example `m1`). `kadvia:render_views` to see it.
- [ ] `kadvia:convert_to_part {"model_id": "m1"}` → a **new** part (for example `m2`) with one `import` feature that holds the file's solids. The STEP data is stored inside the part; the imported model stays open (close it if it gets in the way of renders, with the user's OK).
- [ ] `kadvia:recognize_features {"model_id": "m2"}` → holes, bosses, fillets, chamfers (and optionally planar faces) with face ids and `near` points.
- [ ] Plan the edit and tell the user what you'll change (which holes, which fillets).
- [ ] `kadvia:apply_operations` on the part with direct edits and/or regular features, in small batches. Put new sizes in parameters (`set_parameter`) so they're easy to change.
- [ ] Verify: feature status `ok`, `kadvia:recognize_features` again (hole sizes, fillet count), `kadvia:mass_properties` (volume moved the right way), `kadvia:render_views`.
- [ ] With the user's OK: `kadvia:export_model` to `.step` (for other CAD tools or the shop) and/or `kadvia:save_part` to `.kadvia` (keeps the edits editable).

If `kadvia:convert_to_part` fails, nothing is created: the file has no closed solid Kadvia can
edit (see [Honest limits](#8-honest-limits)). Offer to rebuild the part from its measurements
as a new parametric part instead.

## 2. recognize_features
`kadvia:recognize_features {"model_id": "m2", "include_planes"?: true, "body_id"?: "b0", "max_items"?: 60}`
works on parts and imported models. Per body it returns:
- `counts` and `holeSizes`, for example `{"Ø6 through": 4, "Ø4.2 blind": 2}`;
- `holes[]`: diameter, depth, `through`, axis, centre, `faceIds`, `wallFaceIds`, `near`;
- `bosses[]`: diameter, height, face ids, `near`;
- `fillets[]`: radius, `convex` (a round on an outside edge) or concave (a fillet in an inside corner), face ids, `near`;
- `chamfers[]`: width, distance, face ids, `near`;
- with `include_planes: true`, the largest planar faces (normal, area, a point).

Use face ids for an edit in the same session (`{"ids": [...]}`), and `near` points or
geometric filters in features you'll keep: ids change after `delete_face` and on
regeneration. On an imported model the ids are those of the part `kadvia:convert_to_part`
will create.

## 3. Direct-edit features
Added with `add_feature` like any other feature:

| `type` | Fields | Effect |
|---|---|---|
| `resize_hole` | `faces`, `diameter` | Sets the diameter of the selected cylinders: hole walls or bosses. One feature can resize many holes |
| `delete_face` | `faces` | Removes faces and heals the solid: holes, bosses and pockets are closed by the surrounding face; fillets and chamfers are removed by extending the neighbouring faces to a sharp edge |
| `offset_face` | `faces`, `distance`, `tangent?` | Moves faces along their own outward normals: positive adds material on outer faces and **shrinks** holes; negative removes material and grows holes. Planes move, cylinders/cones/spheres/tori change radius |
| `move_face` | `faces`, `distance`, `direction?`, `tangent?` | Translates faces (default direction: the common normal of the selected planar faces). Select every face of a feature to move the feature (a boss: wall and top; a hole: wall and floor) |

- **Whole surfaces:** a selection grows to every connected face that continues the same surface (a hole wall split in halves edits as one).
- **Tangent faces follow:** with `tangent` (default true) fillets touching an offset or moved face move with it; `tangent: false` keeps them.
- **Same topology:** the neighbours of edited faces are re-intersected and the set of faces and edges stays the same. An edit that would change which faces meet (a face pushed past another, a hole moved off its face, an offset larger than a radius) fails with a `geometry` error and the batch rolls back. Nothing is ever half-applied.
- `delete_face` of a blind hole: select the wall; the floor goes with it. A boss: select its wall; the top goes with it.

## 4. Face selectors for imported geometry
Imported geometry has no feature history, so select faces by what they are:

| Field | Meaning |
|---|---|
| `surface` | `"plane"`, `"cylinder"`, `"cone"`, `"sphere"`, `"torus"`, `"freeform"` |
| `diameter` / `radius` | Cylinders and spheres of that size (tori: the fillet tube radius), ±0.01 mm |
| `concave` | Curved faces only: `true` = hole walls and inside fillets, `false` = bosses and outside rounds |
| `axis` | `"X"`, `"Y"` or `"Z"`: cylinders, cones, tori whose axis is parallel |
| `normal`, `plane` | Planar faces facing a direction / lying on a bounding-box face |
| `near` | The face closest to each point (from `recognize_features`) |
| `ids` | Face ids of the current state (fragile; this session only) |

Filters combine with AND; `near` and `ids` pick among the filtered faces.

| Intent | Selector |
|---|---|
| All Ø6 holes | `{"surface": "cylinder", "concave": true, "diameter": 6}` |
| All vertical Ø6 holes only | `{"surface": "cylinder", "concave": true, "diameter": 6, "axis": "Z"}` |
| All R3 outside rounds | `{"surface": "cylinder", "concave": false, "radius": 3}` (careful: a Ø6 boss matches too; check `recognize_features` first) |
| All R2 inside fillets | `{"surface": "cylinder", "concave": true, "radius": 2}` (careful: Ø4 holes match too, so add `near` points or use the fillet face ids) |
| The top face only | `{"normal": "+Z", "plane": "max_z"}` (`normal` alone also matches upward faces inside the part) |
| One specific fillet | `{"near": [[x, y, z]]}` with the point from `recognize_features` |

## 5. Regular features on imported faces
After conversion every modeling feature works on the imported solid:
- **New holes** on a face, preferably wizard holes: `{"type": "hole", "plane": {"face": {"normal": "+Z", "plane": "max_z"}}, "points": [[10, 10]], "kind": "tapped", "size": "M5", "threadDepth": 8}` (see [holes-threads.md](holes-threads.md)). A face plane's origin is the world origin projected onto the face, with u = world +X (world +Y if the face normal is close to ±X), so `points` are close to world X/Y on a top face.
- **Pockets and bosses**: a sketch with `plane: {"face": ...}`, then an extrude with `"operation": "cut"` or `"add"`.
- **Fillets and chamfers** with edge selectors, `shell`, patterns of your new features.
- **Threads** on existing holes or shafts: a `thread` feature on their cylindrical face (for example after resizing a hole to a tap drill).
- **Material and look:** imported geometry has no material; `kadvia:set_material` gives it mass, `kadvia:set_appearance` a colour.

## 6. Worked example: make all Ø6 holes Ø6.6 and remove the fillets
**User:** "Here's ~/Downloads/mount.step. Make all the Ø6 holes Ø6.6 for M6 clearance and
remove the fillets, then give me a new STEP."

1. `kadvia:open_step_file {"path": "~/Downloads/mount.step"}` → `m1` (imported).
2. `kadvia:convert_to_part {"model_id": "m1"}` → part `m2`.
3. `kadvia:recognize_features {"model_id": "m2"}` → for example `holeSizes`
   `{"Ø6 through": 4}`, 4 convex fillets of radius 3 with `faceIds` [10, 11, 12, 13].
   Note the volume from the result or `kadvia:mass_properties` as a baseline.
4. Tell the user: "4 × Ø6 through holes → Ø6.6; 4 × R3 outside rounds removed (sharp
   edges). OK?" Then:

```json
{"model_id": "m2", "operations": [
  {"op": "set_parameter", "name": "clearance_d", "value": 6.6, "description": "M6 clearance (medium)"},
  {"op": "add_feature", "feature": {"id": "holes_66", "type": "resize_hole",
    "faces": {"surface": "cylinder", "concave": true, "diameter": 6}, "diameter": "clearance_d"}},
  {"op": "add_feature", "feature": {"id": "no_fillets", "type": "delete_face",
    "faces": {"surface": "cylinder", "concave": false, "radius": 3}}}
]}
```
5. Check:
   - both features `ok`;
   - `kadvia:recognize_features` again shows `{"Ø6.6 through": 4}` and no fillets;
   - the volume went **down** by about 4·π·(3.3² − 3²)·depth for the holes and **up** slightly where the rounds were filled to sharp edges;
   - `kadvia:render_views {"views": ["iso", "top"]}`.
6. With the user's OK: `kadvia:export_model {"model_id": "m2", "path": "~/Downloads/mount-v2.step"}`.
   Offer `kadvia:save_part` too, so the edits stay editable (for example `clearance_d` can be
   changed later).

More edits on an imported part:
```json
[
  {"op": "add_feature", "feature": {"id": "thicker", "type": "offset_face",
    "faces": {"normal": "+Z", "plane": "max_z"}, "distance": 2}},
  {"op": "add_feature", "feature": {"id": "boss_move", "type": "move_face",
    "faces": {"near": [[20, 15, 18], [20, 10, 14]]}, "direction": [1, 0, 0], "distance": 8}},
  {"op": "add_feature", "feature": {"id": "no_chamfer", "type": "delete_face",
    "faces": {"near": [[30, 0.5, 9.5]]}}}
]
```
(Make the top plate 2 mm thicker; move a boss, both its wall and top, 8 mm along +X; remove
one chamfer. The `near` points come from `kadvia:recognize_features`.)

## 7. Inserting a STEP file into a part
An `import` feature can also bring a STEP file into any part, for example to combine it with
modeled features. The file is stored in the part unless `"link": true` (then it is re-read
from disk on every regeneration; after the file changed, `kadvia:rebuild_part {"model_id": ..., "force": true}` picks up the new version):
```json
[
  {"op": "add_feature", "feature": {"id": "bracket", "type": "import",
    "source": {"path": "/Users/me/parts/bracket.step"},
    "transform": {"translate": [0, 0, 20]}, "operation": "add"}}
]
```
`transform` takes `rotate: {axis: {origin, direction}, angle}` and `translate` (rotate first);
`bodies` picks some of the file's solids by index (default all).

## 8. Honest limits
- **Not every STEP file converts.** Files whose shells lose entities in conversion, surface-only models (no closed solid) or faces the kernel can't handle stay view-only: `kadvia:convert_to_part` fails and nothing is created. They can still be opened, measured, rendered and exported. Offer to rebuild the part parametrically from its measurements.
- **Edits keep the topology.** An edit that would make faces appear, disappear or meet differently (pushing a face past another, an offset larger than a fillet radius, moving a hole off its face) is refused with a `geometry` error; nothing changes.
- **`delete_face` removes only whole features or blend strips:** holes, bosses, pockets, fillets and chamfers. Any other face is refused ("not a blend strip or a feature face"). Removing a fillet needs neighbours that are planes, cylinders or cones; fillets next to free-form or toroidal faces can't be removed by extension. Corners where three or more blends meet are removed only when the corner is one patch face bordered by removed strips.
- **`move_face`** translates; rotating faces is not available yet.
- **`resize_hole`** only accepts cylindrical faces (not cones or countersinks).
- Direct edits and `import` features cannot be patterned or mirrored.
- Face ids shift after `delete_face`: use `near` points or geometric filters in features you keep.
- Feature recognition on a model whose exact geometry can't be read falls back to the mesh (`exact: false`: holes and planes only, approximate values).
