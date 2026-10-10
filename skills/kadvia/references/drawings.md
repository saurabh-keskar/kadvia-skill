# 2D drawings (`.kdraw`)

A drawing is a 2D engineering drawing of a **part** (kind `part`), an **assembly** (kind
`assembly`) or an **imported STEP model** (kind `imported`), opened as its own model of kind
`drawing`. Views are projected from the exact geometry with hidden lines (across every
component of an assembly), silhouettes and hole centrelines. A drawing can have several
sheets, toleranced dimensions, GD&T symbols, surface finish symbols and, for assemblies, a
BOM table, item balloons and exploded views. The drawing **follows its source**: change the
part or assembly and the views, dimension values and BOM update. The live reference is
`kadvia:modeling_reference {"topic": "drawings"}`.

## Contents
1. [Tools](#1-tools)
2. [Workflow](#2-workflow)
3. [Sources: parts, assemblies, STEP models](#3-sources-parts-assemblies-step-models)
4. [Sheets and projection](#4-sheets-and-projection)
5. [Views](#5-views)
6. [Edge ids and view coordinates](#6-edge-ids-and-view-coordinates)
7. [Dimensions and dimension sets](#7-dimensions-and-dimension-sets)
8. [Tolerances and fits](#8-tolerances-and-fits)
9. [GD&T and surface finish](#9-gdt-and-surface-finish)
10. [Notes, callouts, title block](#10-notes-callouts-title-block)
11. [BOM tables, balloons, exploded views](#11-bom-tables-balloons-exploded-views)
12. [Operations](#12-operations)
13. [Worked example: drawing of the L-bracket](#13-worked-example-drawing-of-the-l-bracket)
14. [Worked example: toleranced flange with GD&T](#14-worked-example-toleranced-flange-with-gdt)
15. [Worked example: assembly drawing with BOM and balloons](#15-worked-example-assembly-drawing-with-bom-and-balloons)
16. [Limits](#16-limits)

## 1. Tools
| Tool | Use it to |
|---|---|
| `kadvia:new_drawing` | Create a drawing (`source`: the model id of an open part, assembly or imported STEP model, or an absolute `.kadvia`, `.kasm`, `.step`/`.stp` path); views, sheet and scale are laid out automatically; `auto_dimension` defaults to true; assemblies also get `bom_table` and `balloons` (both default true) |
| `kadvia:get_drawing` | Read the sheet(s), views (id, sheet, orientation, scale, position), every view's **edge ids**, holes, dimension values and tolerances, annotations, the BOM (assemblies) and warnings |
| `kadvia:apply_drawing_operations` | Edit sheets, views, dimensions, annotations and the title block as one transaction (one undo step); `dry_run: true` checks without committing |
| `kadvia:export_drawing` | Write PDF (vector, one page per sheet), DXF R12 (for CAD/CAM, laser cutting) or SVG; DXF/SVG of a multi-sheet drawing write one file per sheet unless `sheet` picks one |
| `kadvia:save_part` with a `.kdraw` path | Save the drawing itself (save the source first: the drawing references its file) |
| `kadvia:undo` / `kadvia:redo` | Work on drawings too |

`kadvia:open_step_file` opens a saved `.kdraw` (and its source).

## 2. Workflow
- [ ] Finish and check the source first (part: feature status, `kadvia:check_design` if relevant; assembly: solve status, `kadvia:check_interference`).
- [ ] `kadvia:new_drawing` with `source`, the views you need (default `["front", "top", "right", "iso"]`), `title_block` fields and, if the user has a standard, `projection` and `sheet_size`.
- [ ] `kadvia:get_drawing`: compare the auto dimensions' values with the design intent; read `warnings` (and `bom` for an assembly).
- [ ] `kadvia:apply_drawing_operations`: add what the auto dimensions missed (thicknesses, key functional distances), tolerances on the functional sizes, datums and feature control frames if the user works with GD&T, detail/section views for small or hidden features, notes, title block fields, extra sheets.
- [ ] Look at it: the user sees the sheet in Kadvia. Describe what is on it; read values from `kadvia:get_drawing` (dimensions are measured, never typed).
- [ ] Offer `kadvia:export_drawing` (PDF for people, DXF for CAM) and `kadvia:save_part` (`.kdraw`).

## 3. Sources: parts, assemblies, STEP models
| Source | Body ids in edge ids | Auto dimension | Extras |
|---|---|---|---|
| Part | `b0`, `b1`, … | overall sizes, hole diameters and positions, radii | |
| Assembly | `<component>/<body>`, e.g. `bolt/b0`; inside a sub-assembly `<sub>/<child>.<body>`, e.g. `s1/pin2.b0` | overall sizes only | A3 sheet by default, a BOM table above the title block, balloons on the iso view; exploded views |
| Imported STEP | `b0`, `b1`, … in file order | as for a part | dimensions measure the exact solids like a part's |

- Assembly drawings remove hidden lines across components (a pin inside a hole is hidden behind the plate). STEP components of an assembly, and STEP files the app cannot read as exact solids, are drawn from their display meshes (polyline edges, with a warning).
- To save the drawing, the source needs a file: a saved `.kadvia`, a saved `.kasm` (save its parts first) or the STEP file itself.

## 4. Sheets and projection
- `sheet_size`: `A4`, `A3`, `A2`, `A1`, `A0`, `Letter`, `Tabloid`; `orientation`: `landscape` (default) or `portrait`. Default: A4 if the views fit at 1:1, otherwise A3 (assemblies: A3).
- `scale`: sheet mm per model mm, a number (`0.5`) or a string (`"1:2"`, `"2:1"`). Default: the largest standard scale that fits.
- `projection`: `first-angle` (ISO, default: top view below the front view, right view to its left) or `third-angle` (ASME: top view above, right view to the right). Ask if the user's shop works to ASME.
- Sheet coordinates are mm from the bottom-left corner, y up. A 10 mm frame, zone marks and a 180 × 40 mm title block (bottom right) are drawn automatically.

**Several sheets.** A drawing starts with one sheet, id `s1`.
- `add_sheet {sheet?: {id?, name?, size?, orientation?, scale?}, index?}` adds one (size, orientation and scale override the drawing's defaults; the projection method is shared by all sheets).
- `remove_sheet {id}` removes a sheet with its views, their dimensions and its notes (the last sheet stays); `move_sheet {id, index}` reorders (0-based).
- A view goes on a sheet with `view.sheet` (or later `move_view {id, sheet}`); notes and BOM tables with `annotation.sheet`. Dimensions and view annotations follow their view. Everything defaults to the first sheet.
- `update_sheet {patch, sheet?}`: without `sheet` it changes the drawing defaults (`size`, `orientation`, `scale`, `projection`); with `sheet` that sheet's `name`, `size`, `orientation`, `scale`.
- A detail or section view may sit on another sheet than its parent; the detail circle or cutting line is drawn on the parent's sheet.
- The title block shows "SHEET n / N". `kadvia:get_drawing` lists `sheets` (id, index, name, size in mm) when there is more than one, and every view says which `sheet` it is on.

## 5. Views
| Type | Fields |
|---|---|
| `standard` | `orientation` (`front`, `back`, `left`, `right`, `top`, `bottom`, `iso`), optional `position` [x, y] (sheet mm, view centre; omitted = automatic), `scale`, `showHidden` (default true, iso false), `showTangent` (default false), `showCenterlines` (default true, iso false), `label`, `sheet`; assemblies: `explode` |
| `detail` | `parent` (view id), `center` [x, y] and `radius` in the parent's view coordinates, `scale`, optional `name` (letter; "A", "B", … by default), `sheet` |
| `section` | `parent` (an orthographic view), `plane: {point: [x, y], direction: "vertical" \| "horizontal", flip?}`: the cut line in the parent view, optional `name`, `sheet`. Vertical cuts look along −x of the parent, horizontal ones along −y |

Directions match the 3D views (Z up): `front` looks along +Y (view x = X, y = Z), `top` looks
down −Z (x = X, y = Y), `right` looks along −X (x = Y, y = Z).

## 6. Edge ids and view coordinates
Each view has 2D coordinates in **model mm** (the projection of the source; front view: x = X,
y = Z). `kadvia:get_drawing` lists the drawn edges of each view as
`{id, shape: "line" | "circle" | "arc" | "curve" | "point", a, b, center?, radius?, visible}`
and its holes as `{center, diameter, thru, depth, edge}`.
- `"b0:e12"` = edge 12 of body b0; `"b0:s3.0"` = the silhouette (outline) of curved face 3.
- Assemblies prefix the component: `"bolt/b0:e4"`; inside a sub-assembly `"s1/pin2.b0:e7"`.
- Point refs append `.start`, `.end`, `.mid` or `.center` (circles and arcs): `"b0:e7.center"`.
- Edge ids come from the current source geometry. **Read them from `kadvia:get_drawing`; never guess.** Pick an edge by its `shape`, `a`/`b` end points, `radius` and `visible` flag.
- Free-form and elliptical edges are listed only with `all_edges: true`.
- After a topology change in the source (a new hole, say), a dimension whose edge no longer exists reports an error instead of measuring the wrong thing.

## 7. Dimensions and dimension sets
`{id?, view, kind, refs, text?, position?, decimals?, axis?, tolerance?}`. Values are
**measured** from the geometry, never typed.

| `kind` | `refs` | Measures |
|---|---|---|
| `horizontal` | two points, or one line | Δx in the view |
| `vertical` | two points, or one line | Δy in the view |
| `linear` | two points, one line (its length) or two parallel lines (their distance) | true distance |
| `diameter` / `radius` | one circle or arc seen along its axis | Ø / R |
| `angle` | two straight edges | degrees |
| `ordinate` | `[origin, point]` with `axis: "x"` or `"y"` | signed distance from the origin |

- A bare line ref in a two-ref dimension uses its end nearest the other ref; a bare circle uses its centre.
- `text` overrides the label; `<>` stands for the value: `"2X <> THRU"` (a tolerance is kept).
- `position` (view coordinates) places the dimension line and text; omit it for automatic placement outside the view.
- `decimals` defaults to 2 (trailing zeros dropped).
- Dimension a feature in the view where it is seen true size (a hole as a circle, a thickness as a line seen edge-on). Values on views not parallel to the feature are projected lengths.
- Ordinates sharing a view, axis and origin are drawn as one set with a common `0`.

**Dimension sets** dimension many points from one origin in one op:

```json
{"op": "add_dimension_set", "view": "v2", "type": "ordinate", "axis": "x",
 "origin": "b0:e2.start", "refs": ["b0:e18.center", "b0:e20.center", "b0:e22.center"]}
```
- `type`: `ordinate` (one ordinate per ref), `baseline` (dimensions from the origin to each ref, stacked outward) or `chain` (consecutive dimensions origin → p1 → p2 … on one line).
- `axis`: `"x"` for horizontal distances, `"y"` for vertical ones. Optional `position` (where the set's text line goes) and `decimals`.
- `origin` and `refs` are point refs; a bare line ref uses its end nearest the origin, a circle its centre.

**Auto dimension** (on `kadvia:new_drawing`, or the `auto_dimension` op) adds per orthographic
view (not exploded views), skipping what another view already shows: overall width and height,
one diameter per hole size with count and THRU/DEPTH (`"2X Ø6.6 THRU"`), hole positions from
the left/bottom outline, and one radius per arc size (`"2X R3"`). Holes made with the hole
wizard get one standard **hole callout** per hole feature and view instead of diameter dimensions
(see [section 10](#10-notes-callouts-title-block)). Assembly drawings get overall sizes only.

## 8. Tolerances and fits
Add `tolerance` to any dimension (in `add_dimension`, or later with
`update_dimension {id, patch: {tolerance: ...}}`, for example on an auto dimension). Every export
shows it.

| `type` | Fields | Shows |
|---|---|---|
| `symmetric` | `value` | `25 ±0.1` |
| `deviation` | `upper`, `lower` (signed mm) | `25` with `+0.1` over `-0.05` |
| `limits` | `upper`, `lower` (signed deviations) | the limit sizes stacked: `25.1` over `24.95` |
| `fit` | `fit`: `"H7"`, `"g6"` or `"H7/g6"`; `show`: `class` (default), `deviations` or `limits` | `Ø25 H7`, `Ø25 H7 (+0.021/0)`, `Ø25 H7 (25.021/25.000)` |
| `basic` | — | the value in a frame (theoretically exact; use it for the sizes a position tolerance refers to) |
| `reference` | — | `(25)` |

- Fits follow ISO 286: holes D E F G H JS K M N P, shafts d e f g h js k m n p, grades IT5–IT11 (K, M, N, P holes up to IT8), nominal sizes up to 500 mm. A fit outside these tables is a dimension error (the batch is refused).
- `tolerance.decimals` sets the tolerance decimals (default 3).
- Typical fits: `H7/g6` sliding (locating pins, shafts that turn), `H7/h6` locating clearance, `H7/k6` transition (gears, couplings), `H7/p6` press fit. Ask the user which fit the function needs; don't invent tight tolerances (see [design-rules.md](design-rules.md)).
- General tolerances still go in the title block `tolerance` field (`"ISO 2768-m"`).

## 9. GD&T and surface finish
**Datum feature:** `{"type": "datum", "view": "v1", "refs": ["b0:e3"], "text": "A"}`: the letter
in a frame joined to a filled triangle on the edge.

**Feature control frame:**
`{"type": "feature_control_frame", "view", "refs": [edge] | "dimension": "<dimension id>",
"characteristic", "tolerance", "diameter"?, "modifier"?, "datums"?, "position"?}`

| Field | Values |
|---|---|
| `characteristic` | form: `straightness`, `flatness`, `circularity`, `cylindricity`; profile: `profile_line`, `profile_surface`; orientation: `perpendicularity`, `parallelism`, `angularity`; location: `position`, `concentricity`, `symmetry`; runout: `runout`, `total_runout` |
| `tolerance` | zone width in mm |
| `diameter` | `true` for a cylindrical zone (Ø before the value, e.g. hole positions and axis perpendicularity) |
| `modifier` | `"M"` (MMC) or `"L"` (LMC) on the zone |
| `datums` | up to 3: `["A"]`, `["A", "B"]`, `["A-B"]`, `["A", "B(M)"]` |
| `dimension` | attach the frame under that dimension's text (a hole's position under its Ø); otherwise give `view` + `refs` and a leader points at the edge |

- Form tolerances (straightness, flatness, circularity, cylindricity) take **no** datums; orientation, runout, concentricity and symmetry **need** datums. A wrong combination is refused.
- Define a datum (with `datum`) before referring to it.
- The symbols are drawn as geometry, so PDF, DXF and SVG all show them without special fonts.

**Surface finish (ISO 1302):** `{"type": "surface_finish", "view", "refs": [edge], "text": "Ra 1.6",
"process"?: "any" | "removal" | "no_removal"}` (`removal` = machined, a bar on the tick;
`no_removal` = a circle).

## 10. Notes, callouts, title block
- Note: `{"type": "note", "text": "BREAK SHARP EDGES\nDEBURR", "position": [x, y], "height"?: h, "sheet"?}` (sheet mm).
- Hole callout: `{"type": "hole_callout", "view": "v1", "refs": ["<circle edge id>"], "position"?, "text"?}` → "2X Ø6.6 THRU".
  - On any circle of a **wizard hole** (drill, counterbore, countersink or counterdrill circle) it shows the standard callout, one line per element, with the count of equal holes of that feature in the view: `4X Ø6.6 THRU` / `⌴Ø11 ↧6.5`; `Ø4.2 ↧12.5` / `M5x0.8 - 6H ↧10`; `M8x1.25 - 6H THRU`; `Ø5.5 THRU` / `⌵Ø11.2 X 90°`. The ↧, ⌴ and ⌵ symbols are drawn as geometry.
  - `text` overrides work as for dimensions: `"<>"` = the automatic lines, `"\n"` separates lines.
- Cosmetic threads are drawn as thin thread lines (ISO 6410-1): across the axis at the root diameter (internal threads hidden-dashed, or thin solid where a section cuts them) with a thread-limit line; in end views a thin ¾ circle. See [holes-threads.md](holes-threads.md#6-reading-holes-back-resizing-drawings).
- Centre mark `{"type": "center_mark", "view", "refs": [circle]}` and centreline `{"type": "centerline", "view", "refs": [a, b]}`. Holes get both automatically.
- Title block fields (`title_block` on `kadvia:new_drawing`, or `update_title_block`): `title`, `partNumber`, `material`, `author`, `date`, `revision`, `company`, `checked`, `tolerance` (for example `"ISO 2768-m"`). The title defaults to the source's name, the date to today, the revision to "A".
- Automatic positions of symbols, balloons and frames avoid the views, dimension texts, the frame, the title block, BOM tables and each other.

## 11. BOM tables, balloons, exploded views
Assembly drawings only.
- **BOM table:** `{"type": "bom_table", "position"?: [x, y], "structure"?: "top" | "indented" | "flat", "columns"?: [...], "density"?, "sheet"?}`.
  - `position` = the top-right corner in sheet mm (default just above the title block).
  - `columns` from `item`, `name`, `quantity`, `file`, `volume`, `mass`, `material` (default item, name, quantity). `mass` uses the parts' materials (set with `kadvia:set_material`) unless `density` (g/cm³) is given; `material` shows the material names.
  - `structure`: `top` (sub-assemblies as one row), `indented` (their contents under their row, numbered 3.1, 3.2) or `flat` (parts only, quantities summed over every level).
- **Balloon:** `{"type": "balloon", "view", "refs": ["bolt/b0:e4.mid"], "position"?, "text"?}`: a circled item number with a leader ending in a dot on that component. The number comes from the BOM (the structure of the first BOM table), so it never drifts from the table.
- **Auto balloon:** `{"op": "auto_balloon", "view"?, "replace"?}` adds one balloon per BOM item on its longest visible straight edge, spread on a ring around the view (default the iso view); items already ballooned in that view are skipped.
- **Exploded view:** a standard view with `explode`, e.g. `{"orientation": "iso", "explode": 1.5}`, moves every component away from the assembly centre by 1.5 × (its centre − the assembly centre). The caption reads "EXPLODED VIEW"; detail and section views of it are exploded too; auto dimension skips it. (This is independent of the 3D `set_exploded_view`.)
- `kadvia:get_drawing` returns the drawing's `bom`: `{item, name, quantity, components}` per top-level row. It follows the assembly when components are added or removed.

## 12. Operations
`kadvia:apply_drawing_operations {"model_id": "...", "operations": [...]}`:

| `op` | Fields |
|---|---|
| `add_view` / `update_view` / `move_view` / `remove_view` | `view` / `id, patch` / `id, position?, sheet?` / `id` (removing a view also removes its dimensions, annotations and dependent views) |
| `add_dimension` / `update_dimension` / `remove_dimension` | `dimension` / `id, patch` / `id` (removing also removes frames attached to it) |
| `add_dimension_set` | `view, type, axis, origin, refs, position?, decimals?` |
| `add_annotation` / `update_annotation` / `remove_annotation` | `annotation` / `id, patch` / `id` |
| `auto_balloon` | `view?`, `replace?` |
| `update_title_block` | `patch` |
| `update_sheet` | `patch` (`size`, `orientation`, `scale`, `projection`), `sheet?` (then `name`, `size`, `orientation`, `scale` of that sheet) |
| `add_sheet` / `remove_sheet` / `move_sheet` | `sheet?, index?` / `id` / `id, index` |
| `auto_dimension` | `views?`, `replace?` |
| `rename_drawing` | `name` |

Patches are shallow merges (`null` removes a field). If any op is invalid or a new or changed
dimension cannot be measured, **nothing changes** and the error names the op (`opIndex`).
After `update_sheet` views keep their positions; move them with `move_view`, or create a new
drawing for a fresh automatic layout.

## 13. Worked example: drawing of the L-bracket
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

## 14. Worked example: toleranced flange with GD&T
**User:** "Make a production drawing of the flange: the bore is an H7 fit for a bearing seat,
the flange thickness ±0.1, the bottom face is the mounting face, and the bolt holes need a
position tolerance."

Model `m1` is the flange from [../examples/flange.md](../examples/flange.md): OD 140 × 14
on Z = 0, hub Ø70 to Z = 34, bore Ø50, 6 × Ø11 on PCD 100 (one hole at (50, 0)), 1 mm
chamfers. Saved as `~/cad/flange.kadvia`.

1. `kadvia:new_drawing {"source": "m1", "views": ["front", "top", "iso"], "title_block":
   {"title": "Hub flange", "partNumber": "FL-140", "material": "S235", "tolerance": "ISO 2768-m"}}`
   → drawing `m2` with `v1` front (x = X, y = Z), `v2` top (x = X, y = Y), `v3` iso.
2. `kadvia:get_drawing {"model_id": "m2"}`. In `v2` the auto dimensions include the bore
   diameter and `6X Ø11 THRU`; note their ids (here `d3` and `d4`). Find the edges you need:
   - `v2`: the bore circle (`shape: "circle"`, `radius` 25), and the bolt-hole circles centred at (50, 0) and (−50, 0) (`radius` 5.5);
   - `v1`: the bottom line (y = 0, x from −69 to 69 because of the chamfers), and the flange top line on the right (y = 14, from x = 35 to 69).
3. One batch (the ids stand for the ones you read):

```json
{"model_id": "m2", "operations": [
  {"op": "update_dimension", "id": "d3", "patch": {"tolerance": {"type": "fit", "fit": "H7", "show": "deviations"}}},
  {"op": "add_dimension", "dimension": {"id": "thk", "view": "v1", "kind": "vertical",
    "refs": ["b0:e1", "b0:e9"], "tolerance": {"type": "symmetric", "value": 0.1}}},
  {"op": "add_dimension", "dimension": {"id": "pcd", "view": "v2", "kind": "linear",
    "refs": ["b0:e40.center", "b0:e46.center"], "tolerance": {"type": "basic"}}},
  {"op": "add_annotation", "annotation": {"type": "datum", "view": "v1", "refs": ["b0:e1"], "text": "A"}},
  {"op": "add_annotation", "annotation": {"type": "datum", "view": "v2", "refs": ["b0:e31"], "text": "B"}},
  {"op": "add_annotation", "annotation": {"type": "feature_control_frame", "view": "v1", "refs": ["b0:e1"],
    "characteristic": "flatness", "tolerance": 0.05}},
  {"op": "add_annotation", "annotation": {"type": "feature_control_frame", "dimension": "d3",
    "characteristic": "perpendicularity", "tolerance": 0.02, "diameter": true, "datums": ["A"]}},
  {"op": "add_annotation", "annotation": {"type": "feature_control_frame", "dimension": "d4",
    "characteristic": "position", "tolerance": 0.2, "diameter": true, "modifier": "M", "datums": ["A", "B"]}},
  {"op": "add_annotation", "annotation": {"type": "surface_finish", "view": "v1", "refs": ["b0:e1"],
    "text": "Ra 1.6", "process": "removal"}}
]}
```
   Here `b0:e1` is the bottom line, `b0:e9` the flange top line, `b0:e31` the bore circle and
   `b0:e40`/`b0:e46` the two opposite bolt-hole circles.
4. `kadvia:get_drawing` again and check:
   - the bore reads Ø50 H7 with (+0.025/0), the thickness `14 ±0.1`, the PCD `100` in a frame;
   - the frames: flatness 0.05 (no datum), perpendicularity Ø0.02 | A under the bore, position Ø0.2 Ⓜ | A | B under the bolt holes;
   - no `error` on any dimension and no warnings.
5. Export the PDF with the user's OK. The flatness and perpendicularity values are suggestions:
   say so and let the user confirm them.

Why it is built this way: datum A is the mounting face (what the flange sits on), datum B the
bore (what locates it); the hole pattern is located from both, with the PCD as a basic size so
the position tolerance controls it.

## 15. Worked example: assembly drawing with BOM and balloons
**User:** "Make an assembly drawing of the bracket on the plate with a parts list and item
numbers, plus an exploded view on a second sheet. PDF please."

Assembly `m4` is the bracket bolted to the plate from
[assemblies.md](assemblies.md#7-worked-example-bracket-bolted-to-a-plate) (Plate, Bracket,
Bolt). Save the three parts and then the assembly (`~/cad/bracket-on-plate.kasm`) with the
user's OK, so the drawing can be saved too.

1. `kadvia:new_drawing {"source": "m4", "title_block": {"title": "Bracket on plate",
   "partNumber": "ASM-001"}}` → drawing `m5`: an A3 sheet with front, top, right and iso
   views (hidden lines across the parts: the bolt shaft is dashed inside the holes), overall
   sizes, a BOM table above the title block and balloons on the iso view.
2. `kadvia:get_drawing {"model_id": "m5"}`: `sourceKind` is `assembly`; `bom` lists
   1 Plate, 2 Bracket, 3 Bolt with quantity 1 each; the iso view has three balloons. Edge ids
   carry the component: `plate/b0:e…`, `bracket/b0:e…`, `bolt/b0:e…`.
3. Add a section through the bolt, a mass column and the exploded sheet:

```json
{"model_id": "m5", "operations": [
  {"op": "add_view", "view": {"id": "secA", "type": "section", "parent": "v2",
    "plane": {"point": [40, 25], "direction": "horizontal"}, "name": "A"}},
  {"op": "add_sheet", "sheet": {"id": "s2", "name": "Exploded", "size": "A3"}},
  {"op": "add_view", "view": {"id": "ex", "orientation": "iso", "explode": 1.5, "sheet": "s2"}},
  {"op": "auto_balloon", "view": "ex"},
  {"op": "add_annotation", "annotation": {"type": "bom_table", "sheet": "s2",
    "columns": ["item", "name", "quantity", "mass"], "density": 7.85}},
  {"op": "add_annotation", "annotation": {"type": "note", "sheet": "s2",
    "text": "TIGHTEN M6 TO 10 Nm", "position": [20, 40]}}
]}
```
   The section cuts the top view along y = 25 (through the bolt at (40, 25)), so the bolt is
   seen in its holes; every cut face is hatched alike.
4. `kadvia:get_drawing`: `sheets` lists `s1` and `s2` ("Exploded"); view `ex` is on `s2`
   with the caption "EXPLODED VIEW" and three balloons numbered like the BOM.
5. With the user's OK: `kadvia:export_drawing {"model_id": "m5", "path": "~/cad/bracket-on-plate.pdf"}`
   (two pages) and `kadvia:save_part {"model_id": "m5", "path": "~/cad/bracket-on-plate.kdraw"}`.
   A DXF export writes `bracket-on-plate-1.dxf` and `bracket-on-plate-2.dxf`, or one file
   with `"sheet": "s2"`.

Follow-up: "Use two bolts." → add the second bolt and its mates in the **assembly**; the BOM
row becomes Bolt × 2 and the balloons stay correct. Run `auto_balloon` on a view if a new item
appears without a balloon.

## 16. Limits
- No broken views and no weld symbols.
- Section views hatch every cut face the same way (also every component of an assembly).
- Silhouettes of free-form faces follow the triangulation; dimension values on views not parallel to a feature are projected lengths.
- Mesh-only bodies (STEP components of an assembly, or STEP files that cannot be read as exact solids) are drawn as polylines from the display mesh.
- Assembly drawings get overall sizes only from auto dimension; add the rest by hand. Exploded views are not auto-dimensioned.
- Fits cover the ISO 286 letters and grades listed in [section 8](#8-tolerances-and-fits) up to 500 mm; a feature control frame takes at most 3 datums.
- DXF R12 writes dimensions and symbols as plain lines and text (no associative dimensions), one file per sheet.
- Wizard hole callouts and thread lines appear on part drawings only, not on assembly or STEP drawings.
- Drawings that use the newer features (several sheets, assembly or STEP sources, tolerances, GD&T, balloons) cannot be opened by older Kadvia versions; tell the user if they share `.kdraw` files with someone on an older version (PDF/DXF are always fine).
