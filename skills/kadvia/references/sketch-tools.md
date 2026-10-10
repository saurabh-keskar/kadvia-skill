# Sketch tools, DXF import and mesh exports

Constrained sketches can hold splines, ellipses, elliptical arcs, slots, arc slots, text,
conics and construction geometry besides points, lines, arcs and circles. Sketches can be edited
with trim, extend, split, offset, fillet, chamfer, mirror, move/rotate/scale, patterns, explode
and projected model edges. DXF files import as sketches. The live reference is
`kadvia:modeling_reference {"topic": "sketch_tools"}` (basic entities, constraints and dimensions:
topic `constraints`).

## Contents
1. [When to use what](#1-when-to-use-what)
2. [New sketch entities](#2-new-sketch-entities)
3. [Constraints and dimensions (additions)](#3-constraints-and-dimensions-additions)
4. [Profiles from curves](#4-profiles-from-curves)
5. [Editing sketches (edit_sketch)](#5-editing-sketches-edit_sketch)
6. [Projected model edges](#6-projected-model-edges)
7. [DXF import](#7-dxf-import)
8. [Exports](#8-exports)
9. [Worked example: engrave text 0.5 mm deep](#9-worked-example-engrave-text-05-mm-deep)
10. [Worked example: import a DXF outline and extrude it 3 mm](#10-worked-example-import-a-dxf-outline-and-extrude-it-3-mm)
11. [Worked example: offset an outline and round its corners](#11-worked-example-offset-an-outline-and-round-its-corners)
12. [Limits](#12-limits)

## 1. When to use what
| Need | Use |
|---|---|
| Rectangles, circles, polygons, straight slots, line/arc paths | `profiles` (simplest; see [modeling-operations.md](modeling-operations.md#5-profiles)) |
| Free-form outlines, ellipses, arc slots, text, conics | sketch `entities` (this page) |
| Changing an existing sketch: trim, offset, fillet corners, mirror, pattern | `edit_sketch` (tool to preview, operation to commit) |
| A laser-cut or legacy 2D outline the user has as a file | `kadvia:inspect_dxf` → `kadvia:import_dxf` → extrude |
| Engraved or embossed lettering, serial numbers, logos in text | a `text` entity, then a cut or add extrude |

## 2. New sketch entities
Entity coordinates are plane `[u, v]` numbers (the solver's starting point; dimensions take
expressions). Every entity takes `id` and `construction?: true`.

| `type` | Fields | Point refs |
|---|---|---|
| `spline` | `points`, `kind?` (`"fit"` default: smooth curve through the points; `"control"`: NURBS control points with `degree?` (default 3, ≤ 9), `knots?`, `weights?`), `closed?`, `startTangent?` / `endTangent?` (fit splines: tangent handles `[du, dv]`) | `.start`, `.end` (open), `.p0` … `.p<n-1>` |
| `ellipse` | `center`, `major` (the **end point** of the major semi-axis), `minorRadius` | `.center`, `.major` |
| `elliptical_arc` | as `ellipse` + `start`, `end` (counter-clockwise, on the ellipse) | `.start`, `.end`, `.center`, `.major` |
| `slot` | `start`, `end` (centres of the end arcs), `width` | `.start`, `.end` |
| `arc_slot` | `center`, `start`, `end` (counter-clockwise centre-line arc), `width` | `.start`, `.end`, `.center` |
| `text` | `at`, `text` (one line), `height` (capital letter height, mm), `font?` (`"sans"` default, `"mono"`), `bold?`, `italic?`, `angle?` (degrees), `align?` (`"left"`, `"center"`, `"right"` about `at`, on the baseline), `spacing?` (mm) | `.at` (the text id also works as a point) |
| `conic` | `start`, `end`, `control`, `rho` (0 < rho < 1; 0.5 = parabola) | `.start`, `.end`, `.control` |

- **Construction:** `"construction": true` makes any entity reference geometry (no profile). A construction line is a **centerline**: use it as a mirror axis or revolve axis.
- **Text** uses the bundled sans and mono fonts; characters without a glyph are left out. Glyph outlines are profiles (letters with counters such as "O" get their holes). Multi-line text = one `text` entity per line.
- Fit splines have natural ends unless you give a handle; trimmed or split fit splines become control splines.

```json
{"id": "s1", "type": "sketch", "plane": {"base": "XY"},
 "entities": [
   {"id": "e1", "type": "ellipse", "center": [0, 0], "major": [40, 0], "minorRadius": 25},
   {"id": "sl1", "type": "slot", "start": [-15, 0], "end": [15, 0], "width": 8},
   {"id": "cl", "type": "line", "start": [0, -30], "end": [0, 30], "construction": true}
 ],
 "dimensions": [
   {"id": "plate_a", "type": "major_radius", "refs": ["e1"], "value": 40},
   {"id": "plate_b", "type": "minor_radius", "refs": ["e1"], "value": 25},
   {"id": "slot_w", "type": "width", "refs": ["sl1"], "value": 8},
   {"id": "slot_l", "type": "length", "refs": ["sl1"], "value": 30}
 ]}
```
(An elliptical plate with a slot through it; extrude it like any sketch.)

## 3. Constraints and dimensions (additions)
| Constraint | New refs |
|---|---|
| `coincident` / `point_on_circle` | a point and an ellipse or elliptical arc (point on the ellipse) |
| `horizontal`, `vertical`, `parallel`, `perpendicular` | an ellipse = its major axis; a slot = its centre line |
| `tangent` | line + ellipse; a spline or conic **end** + line, arc/circle, ellipse or another spline/conic end (the nearest end is used; fit splines need the handle at that end); line + arc or arc + arc joined end to end |
| `curvature` (new) | a control-spline end + line, arc/circle or control-spline end: tangent and equal curvature (G2) |
| `equal` | two ellipses (both radii), two slots (length and width), two arc slots (radius and width) |
| `fix` on an entity id | every coordinate, radius and width of it |

| Dimension | Refs |
|---|---|
| `major_radius`, `minor_radius` (new) | ellipse / elliptical arc |
| `width` (new) | slot / arc slot |
| `length` | slot: centre-to-centre length (also lines) |
| `radius`, `diameter` | arc slot: centre-line radius (also arcs and circles) |

Large sketches (thousands of entities from a DXF) solve quickly; check them with
`kadvia:solve_sketch` as usual.

## 4. Profiles from curves
- Closed curves are loops of their own: circles, ellipses, closed splines, slots, arc slots and text glyphs. A loop inside another becomes a hole.
- Open curves (lines, arcs, open splines, elliptical arcs, conics) join end to end. A curve end lying on another curve (a T-junction) splits that curve there, so trimmed sketches give the expected outline.
- Splines, ellipses and conics reach the solid as very fine tangent-continuous arcs (well under a micrometre of error on parts of normal size); lines, arcs, circles and slots are exact.

## 5. Editing sketches (edit_sketch)
Two ways to apply the same edits:
- **Preview:** `kadvia:edit_sketch {"model_id": "m1", "sketch": "s1", "edits": [...]}` (`sketch` = a stored sketch id or a sketch feature object). Nothing changes; the result lists the edited entities, constraints and dimensions, whether it solves, the number of closed regions, the ids `created` and the relations removed (`notes`).
- **Commit:** `kadvia:apply_operations` with `{"op": "edit_sketch", "id": "s1", "edits": [...]}`: one undo step; a failing edit rolls back the whole batch.

Edits apply in order to the **solved** geometry. Each is an object with `edit`:

| `edit` | Fields | Effect |
|---|---|---|
| `trim` | `entity`, `at` | Removes the piece between the intersections nearest `at` (no intersection: deletes the entity). Circles become arcs, ellipses elliptical arcs. Power trim = one `trim` per crossed entity |
| `extend` | `entity`, `at` | Extends the line/arc end nearest `at` to the next entity |
| `split` | `entity`, `at` | Splits an open curve at that point |
| `offset` | `entities`, `distance`, `side?` (a point on the wanted side), `both?`, `cap?` (`"arc"`, `"line"`, `"none"`), `constructionBase?` | Connected entities offset as a chain, corners extended or rounded. Closed chains go outward without `side`; open chains to the left of the first entity |
| `fillet` | `corner` (shared end, e.g. `"l1.end"`), `radius`, `dimension?` | Rounds the corner of two lines/arcs (adds tangent relations; `dimension: true` adds the R dimension) |
| `chamfer` | `corner`, `distance`, `distance2?` or `angle?` | Chamfers the corner of two lines |
| `mirror` | `entities`, `axis` (`"x_axis"`, `"y_axis"` or a line id) | Mirrored copies with symmetric relations |
| `move` | `entities`, `by` `[du, dv]`, `copy?` | Translate (or copy) |
| `rotate` | `entities`, `center`, `angle`, `copy?` | Rotate (degrees) |
| `scale` | `entities`, `center`, `factor`, `copy?` | Scale |
| `linear_pattern` | `entities`, `direction`, `count`, `spacing`, `direction2?`, `count2?`, `spacing2?` | Rows (and columns) of copies; `count` includes the original |
| `circular_pattern` | `entities`, `center`, `count`, `angle?` (default 360) | Copies around a point |
| `explode` | `entity` | Slot → 2 lines + 2 arcs; text → splines |
| `construction` | `entities`, `on` | Toggle construction |
| `delete` | `entities` | Delete |
| `project` | `edges` (world points `[x, y, z]`), `body?`, `construction?` | Projected model edges (next section) |

- Relations that no longer hold are removed and listed in `notes` (for example a length dimension on a trimmed line); sensible relations are added (coincident at trimmed ends, tangent for fillets, symmetric for mirrors).
- Slots, arc slots and text must be exploded before trimming or splitting.
- Offset and pattern results are not linked to their source: re-run the edit if the source changes.

## 6. Projected model edges
`{"edit": "project", "edges": [[x, y, z], ...]}` takes the body edge nearest each world point
and projects it onto the sketch plane: straight edges become lines, circles parallel to the
plane circles or arcs, anything else a fit spline. The result is fixed for the solver (fully
defined) and is rebuilt on every regeneration, so it follows parameter changes as long as the
point stays nearest to the same edge. Use it to sketch a lid outline from a box rim, or to cut a
pocket that follows an existing contour.

## 7. DXF import
1. `kadvia:inspect_dxf {"path": "~/Downloads/plate.dxf"}` (optional `units`) → DXF version, layers (entity counts, frozen/off), units, importable entity counts, skipped types, extents in mm, warnings. Nothing changes.
2. `kadvia:import_dxf {"path": ..., "model_id"?, "feature_id"?, "plane"?, "layers"?, "units"?, "scale"?, "origin"?, "tolerance"?, "text"?}`:
   - without `model_id`: a **new part** named after the file with the sketch `dxf1` on XY;
   - with `model_id`: a new sketch (`dxf1`, `dxf2`, … or `feature_id`) in that part, on `plane` (any sketch plane, default `{"base": "XY"}`).
3. Read `sketchId`, the `sketch` status/message (number of closed regions) and `import` (counts, merged points, duplicates removed, skipped, warnings).
4. Extrude the sketch with `kadvia:apply_operations` (an outline with inner loops becomes a plate with holes).

| Option | Meaning |
|---|---|
| `layers` | Layers to import (default: every layer that is not frozen or off) |
| `units` | `mm`, `cm`, `m`, `in`, `ft` or `um`; default from the file (`$INSUNITS`), unitless files are read as mm (with a warning) |
| `scale` | Extra factor after the units |
| `origin` | `"file"` (default: DXF origin at the sketch origin), `"center"` (drawing centred on the origin), `"min"` (lower-left corner at the origin) |
| `tolerance` | End points closer than this (mm) are joined (default 1e-5 × drawing size, at least 0.001 mm) |
| `text` | Import TEXT/MTEXT as sketch text (default `true`). Set `false` when the drawing's labels must not become profiles |

- Read: ASCII R12–2018 and binary DXF. Imported: LINE, ARC, CIRCLE, LWPOLYLINE/POLYLINE (with bulges), SPLINE, ELLIPSE, POINT, TEXT/MTEXT/ATTRIB, INSERT (blocks flattened, arrays expanded). Skipped and counted: HATCH, SOLID, DIMENSION, LEADER, IMAGE, viewports, 3D entities.
- No relations are added: the sketch is under-defined and stays where the file put it. Add dimensions only if the user wants it parametric.
- Exact duplicates and zero-length curves are removed.

## 8. Exports
- `kadvia:export_model` formats: `step` (exact geometry, CAD/CNC), `stl`, `3mf` (mesh package in mm, preferred by most slicers) and `obj` (mesh). The format comes from `format` or the file extension.
- **Sketch as DXF:** the AI tools have no direct sketch-to-DXF export in this version. For a flat outline as DXF, make a drawing of the part (`kadvia:new_drawing` with `"views": ["top"]`, `"scale": "1:1"`, `"auto_dimension": false`) and `kadvia:export_drawing` to `.dxf`; tell the user the file also contains the sheet frame and title block.

## 9. Worked example: engrave text 0.5 mm deep
**User:** "Engrave 'LOT 42' on the top of the plate, 5 mm high, 0.5 mm deep."
The part is a plate centred on the origin with its top face at Z = `thickness`.

```json
[
  {"op": "set_parameter", "name": "engrave_depth", "value": 0.5, "description": "Depth of the engraved text"},
  {"op": "add_feature", "feature": {"id": "label_sk", "type": "sketch",
    "plane": {"face": {"normal": "+Z", "plane": "max_z"}},
    "entities": [{"id": "t1", "type": "text", "at": [0, -2.5], "text": "LOT 42", "height": 5, "align": "center", "bold": true}]}},
  {"op": "add_feature", "feature": {"id": "engrave", "type": "extrude", "sketch": "label_sk",
    "distance": "engrave_depth", "direction": "reverse", "operation": "cut"}}
]
```
- The face plane has the same `[u, v]` as XY (u = +X), so `at: [0, -2.5]` with `align: "center"` centres the 5 mm capitals on the origin.
- `direction: "reverse"` cuts into the material (the face normal points up).
- Verify: `engrave` is `ok`; bbox unchanged; volume dropped by about the letter area × 0.5 (a few tens of mm³); `kadvia:render_views {"views": ["top"]}` shows readable text (not mirrored).
- **Emboss** instead: same sketch, `"direction": "normal"`, `"operation": "add"`.
- Process rules: strokes and depth at least 0.6 mm for FDM; for CNC engraving use a height of at least 3 mm with a small cutter, and mention the cutter size. For text on the side of a part, sketch on that face (for example `{"face": {"normal": "-Y", "plane": "min_y"}}`).

## 10. Worked example: import a DXF outline and extrude it 3 mm
**User:** "Here's ~/Downloads/gasket.dxf. Make it a 3 mm part."

1. `kadvia:inspect_dxf {"path": "~/Downloads/gasket.dxf"}` → for example units `in`, layers `OUTLINE` (12 entities) and `NOTES` (text), extents 4.0 × 2.5 in (101.6 × 63.5 mm). Note the units and the layers to keep.
2. `kadvia:import_dxf {"path": "~/Downloads/gasket.dxf", "layers": ["OUTLINE"], "origin": "center"}` → a new part (`modelId`), `sketchId` `"dxf1"`, sketch message with the region count (an outline with 4 bolt holes = one region with holes).
3. Extrude:
```json
[
  {"op": "set_parameter", "name": "thickness", "value": 3},
  {"op": "add_feature", "feature": {"id": "plate", "type": "extrude", "sketch": "dxf1", "distance": "thickness"}}
]
```
4. Verify: bbox `size` = the DXF extents in mm × 3 (101.6 × 63.5 × 3); volume ≈ outline area × 3; `kadvia:render_views {"views": ["top", "iso"]}`.
5. Offer a material (`kadvia:set_material`), `kadvia:save_part` and `kadvia:export_model` (STEP for the cutter, 3MF/STL for printing).

If the region count is 0 or the extrude fails, the outline has gaps: re-import with a larger
`tolerance` (for example 0.05) as a new sketch (`"feature_id": "dxf2"`, then delete `dxf1`), or
check that the right layers were imported. If the size is 25.4× off, the file's units were
missing or wrong: re-import with `"units": "in"` or `"mm"`.

## 11. Worked example: offset an outline and round its corners
A sketch `s1` holds a closed rectangle of lines `l1`–`l4` joined end to end (`l1.end` meets
`l2.start`, and so on). The user wants R3 corners and a second outline 2 mm outside it, keeping
the original as construction.

1. Round the four corners (commit):
```json
[
  {"op": "edit_sketch", "id": "s1", "edits": [
    {"edit": "fillet", "corner": "l1.end", "radius": 3, "dimension": true},
    {"edit": "fillet", "corner": "l2.end", "radius": 3},
    {"edit": "fillet", "corner": "l3.end", "radius": 3},
    {"edit": "fillet", "corner": "l4.end", "radius": 3}
  ]}
]
```
2. `kadvia:get_part` (or a preview with `kadvia:edit_sketch`) gives the ids of the four new arcs.
   Preview the offset with `kadvia:edit_sketch {"model_id": "m1", "sketch": "s1", "edits": [{"edit": "offset", "entities": ["l1", "l2", "l3", "l4", "<arc ids>"], "distance": 2, "constructionBase": true}]}`:
   it should solve with one closed region (the closed chain goes outward without `side`).
3. Commit the same edit with `{"op": "edit_sketch", "id": "s1", "edits": [...]}` and check the
   feature status and `kadvia:render_views`.

## 12. Limits
- Splines, ellipses and conics become fine arcs in solids and STEP (not exact free-form curves).
- No point-on-spline relations; `curvature` only on control-point splines (not ellipses, conics or fit splines).
- Offsets of splines and ellipses are fit splines, not parametric to the original; offset and pattern results are not linked to their source.
- Projected edges are picked by nearest point (like `near` selectors), not by topology.
- DXF: no ACIS solids, hatch boundaries, dimensions or line types; text uses the bundled fonts.
- Text is one line per entity, without kerning beyond the font's own.
- No direct sketch-to-DXF export through the AI tools (see [Exports](#8-exports)).
