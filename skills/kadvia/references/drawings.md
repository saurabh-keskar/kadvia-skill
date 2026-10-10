# 2D drawings (`.kdraw`)

A drawing is a 2D engineering drawing of a part (kind `part`), opened as its own model of kind
`drawing`. Views are projected from the part's exact geometry with hidden lines, silhouettes
and hole centrelines. The drawing **follows the part**: change the part and the views and
dimension values update. The live reference is `kadvia:modeling_reference {"topic": "drawings"}`.

## Contents
1. [Tools](#1-tools)
2. [Workflow](#2-workflow)
3. [Sheet and projection](#3-sheet-and-projection)
4. [Views](#4-views)
5. [Edge ids and view coordinates](#5-edge-ids-and-view-coordinates)
6. [Dimensions](#6-dimensions)
7. [Notes, callouts, title block](#7-notes-callouts-title-block)
8. [Operations](#8-operations)
9. [Worked example: drawing of the L-bracket](#9-worked-example-drawing-of-the-l-bracket)
10. [Limits](#10-limits)

## 1. Tools
| Tool | Use it to |
|---|---|
| `kadvia:new_drawing` | Create a drawing of a part (`source`: an open part's model id or an absolute `.kadvia` path); views, sheet and scale are laid out automatically; `auto_dimension` defaults to true |
| `kadvia:get_drawing` | Read the sheet, views (id, orientation, scale, position), every view's **edge ids**, holes, dimension values and warnings |
| `kadvia:apply_drawing_operations` | Edit views, dimensions, notes, title block and sheet as one transaction (one undo step); `dry_run: true` checks without committing |
| `kadvia:export_drawing` | Write PDF (vector, for printing), DXF R12 (for CAD/CAM, laser cutting) or SVG |
| `kadvia:save_part` with a `.kdraw` path | Save the drawing itself (save the part first: the drawing references its file) |
| `kadvia:undo` / `kadvia:redo` | Work on drawings too |

`kadvia:open_step_file` opens a saved `.kdraw` (and its part).

## 2. Workflow
- [ ] Finish and check the part first (feature status, `kadvia:check_design` if relevant).
- [ ] `kadvia:new_drawing` with `source`, the views you need (default `["front", "top", "right", "iso"]`), `title_block` fields and, if the user has a standard, `projection` and `sheet_size`.
- [ ] `kadvia:get_drawing`: compare the auto dimensions' values with the design intent; read `warnings`.
- [ ] `kadvia:apply_drawing_operations`: add what the auto dimensions missed (thicknesses, key functional distances), detail/section views for small or hidden features, notes, title block fields.
- [ ] Look at it: the user sees the sheet in Kadvia. Describe what is on it; read values from `kadvia:get_drawing` (dimensions are measured, never typed).
- [ ] Offer `kadvia:export_drawing` (PDF for people, DXF for CAM) and `kadvia:save_part` (`.kdraw`).

## 3. Sheet and projection
- `sheet_size`: `A4`, `A3`, `A2`, `A1`, `A0`, `Letter`, `Tabloid`; `orientation`: `landscape` (default) or `portrait`. Default: A4 if the views fit at 1:1, otherwise A3.
- `scale`: sheet mm per model mm, a number (`0.5`) or a string (`"1:2"`, `"2:1"`). Default: the largest standard scale that fits.
- `projection`: `first-angle` (ISO, default: top view below the front view, right view to its left) or `third-angle` (ASME: top view above, right view to the right). Ask if the user's shop works to ASME.
- Sheet coordinates are mm from the bottom-left corner, y up. A 10 mm frame and a 180 × 40 mm title block (bottom right) are drawn automatically.

## 4. Views
| Type | Fields |
|---|---|
| `standard` | `orientation` (`front`, `back`, `left`, `right`, `top`, `bottom`, `iso`), optional `position` [x, y] (sheet mm, view centre; omitted = automatic), `scale`, `showHidden` (default true, iso false), `showTangent` (default false), `showCenterlines` (default true, iso false), `label` |
| `detail` | `parent` (view id), `center` [x, y] and `radius` in the parent's view coordinates, `scale`, optional `name` (letter; "A", "B", … by default) |
| `section` | `parent` (an orthographic view), `plane: {point: [x, y], direction: "vertical" \| "horizontal", flip?}`: the cut line in the parent view, optional `name`. Vertical cuts look along −x of the parent, horizontal ones along −y |

Directions match the 3D views (Z up): `front` looks along +Y (view x = X, y = Z), `top` looks
down −Z (x = X, y = Y), `right` looks along −X (x = Y, y = Z).

## 5. Edge ids and view coordinates
Each view has 2D coordinates in **model mm** (the projection of the part; front view: x = X,
y = Z). `kadvia:get_drawing` lists the drawn edges of each view as
`{id, shape: "line" | "circle" | "arc" | "curve" | "point", a, b, center?, radius?, visible}`
and its holes as `{center, diameter, thru, depth, edge}`.
- `"b0:e12"` = edge 12 of body b0; `"b0:s3.0"` = the silhouette (outline) of curved face 3.
- Point refs append `.start`, `.end`, `.mid` or `.center` (circles and arcs): `"b0:e7.center"`.
- Edge ids come from the current part geometry. **Read them from `kadvia:get_drawing`; never guess.** Pick an edge by its `shape`, `a`/`b` end points and `visible` flag.
- Free-form and elliptical edges are listed only with `all_edges: true`.

## 6. Dimensions
`{id?, view, kind, refs, text?, position?, decimals?}`. Values are **measured** from the
geometry, never typed.

| `kind` | `refs` | Measures |
|---|---|---|
| `horizontal` | two points, or one line | Δx in the view |
| `vertical` | two points, or one line | Δy in the view |
| `linear` | two points, one line (its length) or two parallel lines (their distance) | true distance |
| `diameter` / `radius` | one circle or arc seen along its axis | Ø / R |
| `angle` | two straight edges | degrees |

- A bare line ref in a two-ref dimension uses its end nearest the other ref; a bare circle uses its centre.
- `text` overrides the label; `<>` stands for the value: `"2X <> THRU"`.
- `position` (view coordinates) places the dimension line and text; omit it for automatic placement outside the view.
- Dimension a feature in the view where it is seen true size (a hole as a circle, a thickness as a line seen edge-on). Values on views not parallel to the feature are projected lengths.

**Auto dimension** (on `kadvia:new_drawing`, or the `auto_dimension` op) adds per orthographic
view, skipping what another view already shows: overall width and height, one diameter per
hole size with count and THRU/DEPTH (`"2X Ø6.6 THRU"`), hole positions from the left/bottom
outline, and one radius per arc size (`"2X R3"`).

## 7. Notes, callouts, title block
- Note: `{"type": "note", "text": "BREAK SHARP EDGES\nDEBURR", "position": [x, y], "height"?: h}` (sheet mm).
- Hole callout: `{"type": "hole_callout", "view": "v1", "refs": ["<circle edge id>"], "position"?, "text"?}` → "2X Ø6.6 THRU".
- Centre mark `{"type": "center_mark", "view", "refs": [circle]}` and centreline `{"type": "centerline", "view", "refs": [a, b]}`. Holes get both automatically.
- Title block fields (`title_block` on `kadvia:new_drawing`, or `update_title_block`): `title`, `partNumber`, `material`, `author`, `date`, `revision`, `company`, `checked`, `tolerance` (for example `"ISO 2768-m"`). The title defaults to the part name, the date to today, the revision to "A".

## 8. Operations
`kadvia:apply_drawing_operations {"model_id": "...", "operations": [...]}`:

| `op` | Fields |
|---|---|
| `add_view` / `update_view` / `move_view` / `remove_view` | `view` / `id, patch` / `id, position` / `id` (removing a view also removes its dimensions and dependent views) |
| `add_dimension` / `update_dimension` / `remove_dimension` | `dimension` / `id, patch` / `id` |
| `add_annotation` / `update_annotation` / `remove_annotation` | `annotation` / `id, patch` / `id` |
| `update_title_block` | `patch` |
| `update_sheet` | `patch` (`size`, `orientation`, `scale`, `projection`) |
| `auto_dimension` | `views?`, `replace?` |
| `rename_drawing` | `name` |

Patches are shallow merges (`null` removes a field). If any op is invalid or a new dimension
cannot be measured, **nothing changes** and the error names the op. After `update_sheet`
views keep their positions; move them with `move_view`, or create a new drawing for a fresh
automatic layout.

## 9. Worked example: drawing of the L-bracket
**User:** "Make a drawing of the bracket for the machinist, A4, with the material and a
general tolerance, and give me a PDF and a DXF."

Model `m1` is the L-bracket from [../examples/l-bracket.md](../examples/l-bracket.md)
(50 × 60 × 40 mm, 5 mm thick, R5/R10 bend, 4 × Ø5.5 holes). Save it first
(`kadvia:save_part {"model_id": "m1", "path": "~/cad/l-bracket.kadvia"}`, with the user's OK)
so the drawing can be saved later.

1. Create the drawing:

```json
{"source": "m1", "views": ["front", "top", "right", "iso"], "sheet_size": "A4",
 "title_block": {"title": "L-bracket", "partNumber": "LB-060", "material": "AL 6061-T6",
                 "author": "A. Engineer", "tolerance": "ISO 2768-m"}}
```
(`kadvia:new_drawing`) → drawing `m2`, first-angle, auto-dimensioned.

2. `kadvia:get_drawing {"model_id": "m2"}`. Check:
   - the views (here `v1` front, `v2` top, `v3` right, `v4` iso; use the ids it returns);
   - overall sizes 50, 60 and 40 appear, the hole callouts read `2X Ø5.5 THRU`, the bend radii `R5` and `R10`;
   - `warnings` is empty.
3. The auto dimensions don't give the plate thickness, and the bend is small at this scale.
   In `v1` (front: x = X, y = Z) the end face of the base leg is the vertical line from
   (50, 0) to (50, 5); find its id in the `v1` edge list. Then:

```json
{"model_id": "m2", "operations": [
  {"op": "add_dimension", "dimension": {"id": "t", "view": "v1", "kind": "vertical",
    "refs": ["b0:e4"], "text": "<> TYP"}},
  {"op": "add_view", "view": {"id": "detA", "type": "detail", "parent": "v1",
    "center": [5, 5], "radius": 12, "scale": 2, "name": "A"}},
  {"op": "add_view", "view": {"id": "secB", "type": "section", "parent": "v2",
    "plane": {"point": [25, 15], "direction": "horizontal"}, "name": "B"}},
  {"op": "add_annotation", "annotation": {"type": "note",
    "text": "BREAK SHARP EDGES 0.3\nANODIZE CLEAR", "position": [20, 60]}}
]}
```
   (`b0:e4` stands for the id you read from `kadvia:get_drawing`.) The detail view enlarges
   the bend around (5, 5) in front-view coordinates; the section cuts the top view along
   y = 15, through the base-leg holes at (35, ±15), showing they go through.
4. `kadvia:get_drawing` again: the `t` dimension reads **5**, no warnings. Move views with
   `move_view` if a new view overlaps the title block.
5. With the user's OK: `kadvia:export_drawing {"model_id": "m2", "path": "~/cad/l-bracket.pdf"}`
   and `{"model_id": "m2", "path": "~/cad/l-bracket.dxf"}` (the format comes from the
   extension), then `kadvia:save_part {"model_id": "m2", "path": "~/cad/l-bracket.kdraw"}`.

Follow-up: "Make it 80 wide." → `kadvia:set_parameters {"model_id": "m1", "values": {"width": 80}}`
on the **part**; `kadvia:get_drawing {"model_id": "m2"}` now shows 80 instead of 60. Re-export
the PDF if the user wants the new revision (and bump `revision` in the title block).

## 10. Limits
- One part per drawing and one sheet per drawing; assembly drawings are not supported yet.
- No ordinate dimensions, tolerances/GD&T symbols or broken views yet. Put tolerances in the title block `tolerance` field or in notes.
- Section views hatch every cut face the same way (one part).
- Silhouettes of free-form faces follow the triangulation; dimension values on views not parallel to a feature are projected lengths.
