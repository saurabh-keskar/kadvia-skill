---
name: kadvia
description: Designs, edits and inspects mechanical CAD in the Kadvia desktop app through its MCP server. Builds parametric parts from text, photos, drawings or DXF files (sketches with splines and text, extrudes, revolves, standard tapped and counterbored holes, threads, fillets, patterns, lofts, sweeps, shells driven by named parameters), traces calibrated reference images, edits imported STEP files (resize holes, remove fillets), assembles parts with mates (gears, cams, limits; DOF, BOM, interference), makes 2D drawings (tolerances, GD&T, hole callouts, BOM, balloons; PDF/DXF/SVG), assigns materials and appearances for mass and product renders, and checks manufacturability for CNC, 3D printing, molding and sheet metal. Verifies with exact mass properties and renders; saves .kadvia/.kasm/.kdraw or exports STEP/STL/3MF/OBJ. Use when the user asks to model, design, recreate, modify, measure, assemble, draw, render, check or export a CAD part, STEP or DXF file, shares an image of a part to rebuild, or mentions Kadvia.
compatibility: Requires the Kadvia desktop app (macOS or Windows) running, with its MCP server ("kadvia") connected to an MCP-capable AI client.
metadata:
  version: "0.5.0"
---

# Kadvia

Kadvia is a desktop CAD application. The AI controls the **running app** through the
`kadvia` MCP server, and everything it does appears live in the user's Kadvia window.

What it can do:
- **Model parametric parts:** text-to-CAD, image-to-CAD (tracing over calibrated reference images), and edits through parameters.
- **Standard holes and threads:** hole wizard (tapped, clearance, counterbore, countersink, counterdrill; ISO metric and inch sizes) and cosmetic or modeled threads.
- **Sketch tools:** splines, ellipses, slots, arc slots, text, conics, construction lines; trim, extend, offset, fillet, mirror, patterns, projected edges; DXF import.
- **Materials and appearance:** a library of 22 engineering materials (real mass), colours and finishes per part, body or face, and product renders.
- **Edit imported STEP files:** convert to a part, recognise holes and fillets, resize holes, delete, offset and move faces.
- **Assemblies** (`.kasm`): place parts and rigid sub-assemblies, mate them (including gear, rack and pinion, cam and limit mates), check DOF, BOM (top-level, indented, flat) and interference.
- **2D drawings** (`.kdraw`) of parts, assemblies and STEP models: auto-dimensioned views, detail, section and exploded views, ordinate/baseline/chain dimensions, tolerances and ISO fits, GD&T and surface finish, BOM tables and balloons, multiple sheets, title block, PDF/DXF/SVG.
- **Design check:** manufacturability for CNC, FDM/SLA printing, injection molding and sheet metal.
- **Inspect models:** open files, measure exactly, take screenshots and read the user's selection (faces, edges, vertices, bodies).
- **Save and export:** `.kadvia`, `.kasm`, `.kdraw`; STEP, STL, 3MF or OBJ.

## Conventions
- Units: **mm** and **degrees**. Areas are mm² and volumes mm³. Convert only when the user asks.
- **Z is up.** `front` looks along +Y (camera on −Y), `top` looks down −Z and `right` looks along −X.
- Ids: model ids come from `kadvia:kadvia_status`, `kadvia:list_models` and the tools that create or open models. Feature, parameter and body ids come from `kadvia:get_part`; face, edge and vertex ids from `kadvia:get_selection`, `kadvia:recognize_features` or `kadvia:get_drawing`. **Never guess ids.**
- Model kinds: `part` (editable, `.kadvia`), `imported` (STEP, view/measure until converted), `assembly` (`.kasm`), `drawing` (`.kdraw`).
- Paths for files must be absolute (`~/` is allowed).
- Sketch planes: `XY` is the floor (normal +Z), `XZ` is the front wall (normal **−Y**, so offset `o` puts the plane at Y = −o) and `YZ` is the side wall (normal +X).

Details: [references/views-and-conventions.md](references/views-and-conventions.md).

## Tools
| Tool | Use it to |
|---|---|
| `kadvia:kadvia_status` | Check the app is running; list open models and their `kind` |
| `kadvia:new_part` | Start a new, empty parametric part (opens in the user's window) |
| `kadvia:get_part` | Read a part: parameters, features with ids, feature status, bodies, revision |
| `kadvia:modeling_reference` | Exact fields for every op, feature, profile and selector, with examples (topics include `holes`, `sketch_tools`, `materials`, `assemblies`, `drawings`, `references`, `design_check`, `imported`) |
| `kadvia:apply_operations` | **The modeling tool**: a batch of operations as one transaction (one undo step) |
| `kadvia:set_parameters` | Change named dimensions: `{"width": 90}` |
| `kadvia:solve_sketch` / `kadvia:edit_sketch` | Check a constrained sketch / preview sketch edits (trim, offset, fillet, mirror, patterns, …) without changing the part |
| `kadvia:inspect_dxf` / `kadvia:import_dxf` | Read a DXF's layers, units and extents / import it as a sketch (new part or existing one) |
| `kadvia:undo` / `kadvia:redo` | Step back or forward through committed changes (parts, assemblies, drawings) |
| `kadvia:mass_properties` / `kadvia:measure` | Exact volume, area, centre of mass, mass (from the assigned materials); exact lengths, radii, distances and angles of faces/edges |
| `kadvia:list_materials` / `kadvia:set_material` / `kadvia:set_appearance` | Material library; assign a material (part, body, assembly component); colour and finish (part, body, faces; display only) |
| `kadvia:render_views` | See the model: 1–8 standard views as images (optional section, reference images; `quality: "render"` for product shots) |
| `kadvia:check_design` | Manufacturability check for a process, with locations and fixes |
| `kadvia:open_step_file` | Open `.step`/`.stp` (imported), `.kadvia` (part), `.kasm` (assembly) or `.kdraw` (drawing) |
| `kadvia:convert_to_part` / `kadvia:recognize_features` | Make an imported STEP editable; list holes (with wizard size and thread), bosses, fillets and chamfers |
| `kadvia:new_assembly` / `kadvia:get_assembly` / `kadvia:apply_assembly_operations` | Build and edit assemblies (components, sub-assemblies, mates) |
| `kadvia:assembly_bom` / `kadvia:check_interference` | Bill of materials (`structure`: `top`, `indented`, `flat`); overlapping components |
| `kadvia:new_drawing` / `kadvia:get_drawing` / `kadvia:apply_drawing_operations` / `kadvia:export_drawing` | 2D drawings of parts, assemblies or STEP models: create, read edge ids, edit (sheets, dimensions, tolerances, GD&T, BOM, balloons), write PDF/DXF/SVG |
| `kadvia:save_part` / `kadvia:export_model` | Write `.kadvia`/`.kasm`/`.kdraw` / STEP, STL, 3MF or OBJ (only when the user asks or agrees) |
| `kadvia:list_models` / `kadvia:get_model_info` | Bodies, faces, edges, bbox, volume, warnings of any model |
| `kadvia:set_view` / `kadvia:fit_view` / `kadvia:set_display_mode` | Present things in the user's viewport |
| `kadvia:get_selection` | What the user clicked (faces, edges, vertices with their `point`, bodies) |
| `kadvia:close_model` | Remove a model from the window (confirm first; save before closing) |

## Modeling checklist (copy it and tick it off)
- [ ] `kadvia:kadvia_status`: is the app running and the window ready?
- [ ] Understand the request: function, envelope, key dimensions, holes and fasteners, material and process. **Ask** about anything that changes the design and has no sensible default.
- [ ] Plan on paper: a feature list in build order, plus a parameter table (name, value, why).
- [ ] `kadvia:new_part` (new design) or `kadvia:get_part` (existing part).
- [ ] `kadvia:modeling_reference` once per session, before the first batch.
- [ ] Batch 1: **parameters** (`set_parameter` for every key dimension).
- [ ] Batches 2…n: features in small logical steps: base solid → cuts and pockets → holes (hole wizard) → patterns and mirrors → fillets and chamfers.
- [ ] After **every** batch, read the result: all features `ok`, expected body count, bbox `size` matching the envelope, volume moving the right way.
- [ ] `kadvia:render_views` (`["iso","top","front"]` or the views that show the change).
- [ ] `kadvia:set_material` when the material is known, then `kadvia:mass_properties`: compare with your hand estimate and give the mass.
- [ ] Compare against the user's spec item by item; fix and iterate.
- [ ] `kadvia:check_design` with the user's process when the part is meant for production (ask first which process).
- [ ] Report what was built, the parameters the user can change, the checks you ran and your assumptions.
- [ ] Offer `kadvia:save_part` (`.kadvia`, keeps parameters) and `kadvia:export_model` (STEP for CAD/CNC, 3MF or STL for printing).

A minimal first modeling batch (a plate):

```json
[
  {"op": "set_parameter", "name": "width", "value": 100, "description": "Plate length (X)"},
  {"op": "set_parameter", "name": "depth", "value": 60, "description": "Plate width (Y)"},
  {"op": "set_parameter", "name": "thickness", "value": 6},
  {"op": "add_feature", "feature": {"id": "plate_sk", "type": "sketch", "plane": {"base": "XY"},
    "profiles": [{"kind": "rect", "center": [0, 0], "width": "width", "height": "depth"}]}},
  {"op": "add_feature", "feature": {"id": "plate", "type": "extrude", "sketch": "plate_sk",
    "distance": "thickness"}}
]
```

Full operation and feature catalogue (including loft, sweep, shell, draft, face selectors,
the rollback bar and known limits): [references/modeling-operations.md](references/modeling-operations.md).

## Design-intent rules
1. **Every key dimension is a named parameter.** Use short, meaningful snake_case names: `width`, `depth`, `height`, `wall`, `thickness`, `hole_d`, `hole_inset`, `bolt_count`, `pcd`, `corner_r`. Add a `description` with the meaning and direction (for example "along X").
2. **Features use expressions, not copied numbers.** Write `"width/2 - hole_inset"`, not `42`. Derived sizes are expressions too: `{"name": "inner_w", "value": "width - 2*wall"}`.
3. **Counts are unitless** (`"unit": ""`), and angles use `"unit": "deg"`.
4. **Place the part deliberately.** Centre symmetric parts on the origin (rect `center: [0, 0]`, symmetric extrudes) so mirrors and patterns use the origin planes. Put the base on Z = 0.
5. **Give every feature a descriptive id** (`base_sk`, `base`, `mount_holes`, `edge_fillet`). Later operations and the user refer to them.
6. **Holes use standard sizes through the hole wizard:** `"kind": "tapped" | "clearance" | "counterbore" | "countersink"` with `"size": "M6"` (or `"1/4-20 UNC"`) instead of typed diameters. The tables give clearance Ø (normal fit: M3 3.4, M4 4.5, M5 5.5, M6 6.6, M8 9, M10 11), tap drills (M3 2.5, M4 3.3, M5 4.2, M6 5.0, M8 6.8), socket-head counterbores and threads, and drawings get proper callouts. Plain `diameter` holes are for non-fastener holes. Keep hole centres at least 1.5×d from edges. See [references/holes-threads.md](references/holes-threads.md) and [references/design-rules.md](references/design-rules.md).
7. **Fillets and chamfers:**
   - Inside corners of an L-shape get radius `r` and the matching outside corner gets `r + thickness`, so the wall stays uniform.
   - Box corners can be rounded in one feature (`{"all": true}` gives spherical corners). Where convex and concave edges meet (an L-bracket), use separate features.
   - For CNC, inside vertical corners need at least the tool radius (≥ 1 mm, 3 mm is typical).
   - For 3D printing, chamfer bottom edges (no overhang) and fillet top edges.
   - Apply small cosmetic edge breaks (0.5–1 mm) last.
8. **Respect the process minimums** (wall, hole and radius) for the user's process. If you don't know it, ask, or assume and say so.
9. **Prefer robust selectors.** Use `parallel`, `plane`, `type` and `radius` first. Use `near` with expression coordinates (`[["t", 0, "t"]]`) only when no combination of those selects exactly the right edges. Never save designs that rely on `ids`.
10. **Prefer profile features over fillets** where possible. A rect `cornerRadius` is more robust than filleting vertical edges afterwards.

## Text-to-CAD workflow
1. Restate the requirements as a short spec: envelope, features, fasteners, material and process. Fill gaps with stated, conventional defaults (for example 2 mm walls for a printed enclosure). Ask only about things that would make you redo the work, such as which way a flange faces or the hole pattern.
2. Choose the construction:
   - Prismatic parts: sketch → extrude.
   - Turned parts: sketch the half-profile → `revolve`.
   - Blocky parts: primitives (`box`, `cylinder`) with `add`/`cut`.
   - Repeated features: `linear_pattern`, `circular_pattern`, `mirror`.
   - Free-form or lettered outlines: sketch `entities` (splines, ellipses, slots, text); a 2D file the user has: `kadvia:import_dxf`.
3. Write the parameter table, then build batch by batch (checklist above).
4. Verify each requirement with numbers (bbox, `mass_properties`) and images.
5. Report a table of parameters (name = value) so the user knows what they can change.

Worked examples: [examples/l-bracket.md](examples/l-bracket.md) (sketch + extrude, fillets, holes on two planes), [examples/flange.md](examples/flange.md) (circular pattern, chamfers), [examples/enclosure.md](examples/enclosure.md) (cavity cut, bosses, cut-out on the XZ plane).

## Image-to-CAD workflow
1. **Read the image.** Identify the base shape, every feature (holes, slots, bosses, ribs, fillets), the symmetry, and the view direction of each photo or drawing view.
2. **Find the scale.** Use a written dimension, a known object (ruler, coin, bolt head, standard connector) or a standard feature size. Without a scale you only have proportions, so ask for one real dimension.
3. **Trace it in Kadvia** when the user has the image file: `add_reference` on the plane it was taken from (front → `XZ`, top → `XY`, side → `YZ`), then calibrate with `update_reference` `calibration` `{p1, p2, distance}` from the known dimension. Compare with `kadvia:render_views` `include_references: true`.
4. **Estimate dimensions** from the scale (or calibrated image coordinates). Use measurements only from faces seen square-on; perspective shortens receding edges.
5. **Ask the user to confirm the key dimensions before modeling.** Show a short table (parameter, estimate, basis) and say which values are guesses. Ask for any hidden features (back side, hole depth).
6. Model with **every estimated dimension as a parameter**, sketching on the reference image's plane, so corrections are one `kadvia:set_parameters` call.
7. Verify by rendering the same view as the image with `include_references: true` (`front` ≈ a straight-on photo, `iso` ≈ a three-quarter photo). Compare silhouette, proportions and feature counts.
8. Report your assumptions and offer to adjust them.

Details, the calibration formulas and a worked tracing example: [references/image-to-cad.md](references/image-to-cad.md).

## Holes and threads
1. Write wizard holes: `{"type": "hole", "plane": {"face": {"normal": "+Z", "plane": "max_z"}}, "points": [...], "kind": "tapped", "size": "M5", "threadDepth": 10}`. Kinds: `tapped`, `clearance` (`fit`: `close`, `normal`, `loose`), `counterbore`, `countersink`, `counterdrill`, `simple`. Sizes: `"M3"` … `"M64"`, `"M8x1"`, `"1/4-20 UNC"`, `"#10-32 UNF"`.
2. Depth: `depth` (blind; a 118° drill point is added, `drillPoint: 0` for flat), `through: true`, or `endCondition: "up_to_face"` with `upTo`. "M5 × 10 deep" usually means 10 mm of full thread: give `threadDepth: 10` and the drill depth follows (thread + 3 pitches).
3. Threads: tapped holes get a **cosmetic** thread by default (`thread: "modeled"` cuts real geometry, slow: for 3D printing). Shafts and existing holes get a `thread` feature on their cylindrical face.
4. Resize with `update_feature {"id": "m5_holes", "patch": {"size": "M6"}}`. `kadvia:recognize_features` reports wizard holes by size and kind with their thread; drawings get standard hole callouts (`4X Ø6.6 THRU` / `⌴Ø11 ↧6.5`).

Worked examples (four M5 tapped holes 10 deep, counterbored holes for M6 socket head screws), tables and limits: [references/holes-threads.md](references/holes-threads.md).

## Sketch tools and DXF import
- Sketch `entities` add `spline`, `ellipse`, `elliptical_arc`, `slot`, `arc_slot`, `text`, `conic`; any entity can be `"construction": true` (a construction line is a centerline). New relations and dimensions: `tangent` at spline ends, `curvature`, `equal` for ellipses and slots, `major_radius`, `minor_radius`, `width`.
- Text: `{"id": "t1", "type": "text", "at": [0, 0], "text": "LOT 42", "height": 5, "align": "center"}` on a face sketch, then an extrude `"direction": "reverse", "operation": "cut"` engraves it (`"add"` embosses).
- Edit sketches with `trim`, `extend`, `split`, `offset`, `fillet`, `chamfer`, `mirror`, `move`, `rotate`, `scale`, `linear_pattern`, `circular_pattern`, `explode`, `construction`, `delete`, `project`: preview with `kadvia:edit_sketch` (changes nothing), commit with the `{"op": "edit_sketch", "id": "s1", "edits": [...]}` operation.
- DXF: `kadvia:inspect_dxf` (layers, units, extents) → `kadvia:import_dxf` (`layers`, `units`, `origin: "center"`, `text: false` to skip labels; without `model_id` it makes a new part with sketch `dxf1`) → extrude the sketch.

Worked examples (engrave text 0.5 mm deep, import a DXF outline and extrude it 3 mm, offset and fillet a sketch) and limits: [references/sketch-tools.md](references/sketch-tools.md).

## Editing an existing part
1. `kadvia:get_part`: read the parameter names, feature ids and current status. For a STEP file (kind `imported`) there are no parameters: convert it (see [Editing imported STEP files](#editing-imported-step-files)).
2. When the user says "this face", "that hole" or "here", call `kadvia:get_selection` to find out what they mean.
3. **Size changes go through parameters:** `kadvia:set_parameters {"values": {"width": 90}}`. Check the `changes` list: did the bbox grow by exactly what was asked?
4. Structural changes go through `kadvia:apply_operations`:
   - Add features with `add_feature` (use `after` to insert them before the fillets).
   - Change fields with `update_feature` (a shallow merge: a patched `plane`, `profiles`, `points`, `edges` or `faces` replaces the whole value).
   - Use `suppress_feature` to try a design without a feature.
   - Use `delete_feature` only when the feature is truly unwanted.
5. If a new dimension appears, add a parameter for it in the same batch.
6. For risky changes, use `dry_run: true` first. Use `kadvia:undo` to back out a committed step the user doesn't like.
7. **Rollback bar:** `{"op": "set_rollback", "to": "<feature id>"}` regenerates only up to that feature (later ones show `skipped`). While rolled back, `add_feature` without `after` inserts at the bar. `{"op": "set_rollback"}` rolls forward again; always do that before you finish.

## Editing imported STEP files
1. `kadvia:open_step_file` → kind `imported`. `kadvia:convert_to_part` → a **new** part whose first feature holds the solids (the STEP data is stored in it). If conversion fails, nothing is created: the file stays view-only; offer a parametric rebuild.
2. `kadvia:recognize_features` on the part → holes (`holeSizes` such as `{"Ø6 through": 4}`), bosses, fillets, chamfers, with face ids and `near` points.
3. `kadvia:apply_operations` with direct-edit features: `resize_hole {faces, diameter}`, `delete_face {faces}` (removes fillets, chamfers, holes, bosses and heals), `offset_face {faces, distance}`, `move_face {faces, distance, direction?}`, or any regular feature on the imported faces (holes, sketches on `{"face": ...}` planes, cuts, fillets).
4. Select faces with geometric filters you can keep (`{"surface": "cylinder", "concave": true, "diameter": 6}`) or `near` points; ids change after edits.
5. Verify (`kadvia:recognize_features` again, `kadvia:mass_properties`, renders), then `kadvia:export_model` to STEP with the user's OK.

"Make all Ø6 holes Ø6.6 and remove the R3 fillets" is one batch:

```json
[
  {"op": "add_feature", "feature": {"id": "holes_66", "type": "resize_hole",
    "faces": {"surface": "cylinder", "concave": true, "diameter": 6}, "diameter": 6.6}},
  {"op": "add_feature", "feature": {"id": "no_fillets", "type": "delete_face",
    "faces": {"surface": "cylinder", "concave": false, "radius": 3}}}
]
```
Edits keep the topology: anything that would make faces appear, vanish or meet differently is
refused and nothing changes. Workflow, selectors and honest limits: [references/editing-step.md](references/editing-step.md).

## Assemblies
1. Build or open the parts first; read their geometry (mate references use each **part's own coordinates**).
2. `kadvia:new_assembly`, then `kadvia:apply_assembly_operations` with `add_component` (`source`: `{"model": "<open part id>"}`, `{"path": ...}` or `{"step": ...}`; rough `transform.position`). The first component is grounded. A saved `.kasm` as `path` is a **rigid sub-assembly** (mate references use its own coordinates).
3. Add mates in small batches: `coincident`, `concentric`, `distance`, `angle`, `parallel`, `perpendicular`, `tangent`, `fixed`, `lock`; `flip: true` reverses a coincident/distance/angle/tangent sense.
   - Limits: `distance`/`angle` with `min`/`max` instead of `value` (a stroke or opening range).
   - Motion: `gear` (`ratio` = teeth A / teeth B), `rack_pinion` (`value` = π × pitch diameter, mm per turn), `cam` (follower on a cam face). Hold their axes with other mates first.
   - Centring: `symmetric` (`c` = mirror plane), `width` (`a`, `b` slot faces; `c`, `d` tab faces).
4. After each batch read `solve.status`, `solve.dof`, `underConstrained` and each mate's `status` (`ok` / `redundant` / `conflicting` / `error`). A bolt keeping 1 DOF (spin) is fine; so is a gear train keeping 1 DOF.
5. `kadvia:render_views`, `kadvia:check_interference`, `kadvia:assembly_bom` (`structure: "indented"` or `"flat"` with sub-assemblies; masses and materials come from the parts' materials, or `kadvia:set_material` with `component_id` overrides one component). Optional `set_exploded_view {scale}` (display only).
6. `kadvia:save_part` (`.kasm`, after saving the parts) and `kadvia:export_model` (STEP: part solids placed through every level; STL: everything).

Worked examples (bracket bolted to a plate, gear pair), sub-assemblies and limits: [references/assemblies.md](references/assemblies.md).

## 2D drawings
1. `kadvia:new_drawing {"source": "<part, assembly or STEP model id>", "views": [...], "title_block": {...}}`: views, sheet and scale are laid out and auto-dimensioned (first-angle by default; `projection: "third-angle"` for ASME). Assemblies also get a BOM table and balloons (`bom_table`, `balloons`).
2. `kadvia:get_drawing`: check dimension values and `warnings`; read the **edge ids** (`"b0:e12"`, assemblies `"bolt/b0:e4"`; `.start`/`.end`/`.mid`/`.center`) per view, and the `bom` of an assembly.
3. `kadvia:apply_drawing_operations`:
   - `add_dimension` (`horizontal`, `vertical`, `linear`, `diameter`, `radius`, `angle`, `ordinate`; values are measured, never typed) with an optional `tolerance` (`symmetric`, `deviation`, `limits`, `fit` such as `"H7"` or `"H7/g6"`, `basic`, `reference`); `add_dimension_set` (`ordinate`, `baseline`, `chain`).
   - `add_view` (`standard`, `detail`, `section`; `explode` for assemblies; `sheet`).
   - `add_annotation`: `note`, `hole_callout`, `datum`, `feature_control_frame` (flatness, perpendicularity, position, … with datums), `surface_finish`, `balloon`, `bom_table`; `auto_balloon`.
   - `add_sheet` / `remove_sheet` / `move_sheet`, `update_title_block`, `update_sheet`.
4. `kadvia:export_drawing` (`.pdf`: every sheet as a page; `.dxf`/`.svg`: one file per sheet, or `sheet`) and `kadvia:save_part` (`.kdraw`, after saving the source), with the user's OK. The drawing follows later changes of its source.
   - Wizard holes get standard hole callouts automatically and cosmetic threads are drawn as thread lines; BOM tables can show `mass` and `material` from the assigned materials.
5. Add tolerances and GD&T only where the user asks or the function needs them; ask for the fit or zone size instead of inventing tight values.

Worked examples (L-bracket, toleranced flange with GD&T, assembly drawing with BOM and balloons) and limits: [references/drawings.md](references/drawings.md).

## Materials, appearance and renders
1. `kadvia:list_materials` (`query`: "aluminium", "delrin"; `category`: `metal`, `plastic`, `other`) when unsure of the id. Library ids such as `aluminium_6061`, `stainless_304`, `steel_1018`, `abs`, `pla`, `petg`, `nylon_pa12`, `pom`; names and aliases work too; a custom `{"name": ..., "density": g/cm³}` for anything else.
2. `kadvia:set_material {"model_id", "material", "body_id"?, "component_id"?}`: mass, BOM masses and drawing BOM tables then come from the material (no density argument needed).
3. `kadvia:set_appearance {"model_id", "color": "#2a5db0", "finish": "satin", ...}` for the part, one body (`body_id`) or faces (`body_id` + `faces` selector). Finishes: `polished`, `brushed`, `satin`, `matte`, `glossy`, `textured`; also `metalness`, `roughness`, `clearcoat`, `opacity`. Display only: geometry and mass never change. Keep the real material for anodised or painted metal and change only the look.
4. Product shot: `kadvia:render_views {"views": ["iso"], "quality": "render", "environment": "studio" | "outdoor" | "warehouse", "background": "transparent", "width": 1600, "height": 1200}`. Renders show every open model; use the standard quality for checking geometry.

Worked example (anodised blue aluminium product render), finishes, precedence and limits: [references/materials-appearance.md](references/materials-appearance.md).

## Design check
- Run `kadvia:check_design {"model_id": ..., "process": ...}` when a design is done or the user asks whether it can be made. Processes: `cnc`, `print_fdm`, `print_sla`, `injection`, `sheet_metal`, `general`. Ask which one if unclear; pass `build_direction` and `params` (for example `{"toolRadius": 1.5}`) when known.
- Explain errors, then warnings, with `value` vs `limit` and where. Infos are advice.
- Propose a fix in the feature or parameter that made the face (`location.point` works in `near` selectors), **ask before changing the design**, then run the check again.

Rules, limits and a worked fix loop: [references/design-check.md](references/design-check.md).

## Verification: what to check
| After | Check numerically | Check visually (`kadvia:render_views`) |
|---|---|---|
| Base extrude/revolve/box | bbox `size` = intended envelope; volume ≈ area × height | `iso` shape, orientation (Z up) |
| Cut / pocket | volume decreased by about the pocket volume; bbox unchanged | `top`/`iso`: position, depth, it cut the right side |
| Holes | volume decreased by about n·π·r²·depth (plus drill points and counterbores); `kadvia:recognize_features` `holeSizes` shows the wizard size and kind | `top` (or the drilled face view): count, positions, through; a `section` for counterbores |
| Pattern / mirror | volume changed by count × the single feature | `top`: count and spacing; nothing missing |
| Fillet / chamfer | volume changed slightly; edge/face count rose; feature `ok` | `iso`: the right edges rounded |
| Material / appearance | bodies report the material and `mass_g`; geometry and volume unchanged | `quality: "render"` for the look |
| Parameter change | `changes`: bbox/volume moved by the expected amount | the view that shows that dimension |
| Direct edit (imported) | `kadvia:recognize_features`: new hole sizes / fillets gone; volume moved the right way | the edited faces |
| Before export | `kadvia:mass_properties`: one closed body (unless intended), volume > 0, mass if the material is known; `kadvia:check_design` for the process | all four default views |

Always compute a rough hand estimate first. A result 2× off means a wrong plane, a wrong direction or a missed cut.

## When something fails
- A failed `kadvia:apply_operations` (or assembly/drawing batch) changes **nothing**. The error gives the `code`, a message, the failing operation (`opIndex`) and feature or mate (`featureId`), plus a hint.
  - Fix that operation and resend the corrected batch. Don't resend only the tail, because the earlier operations were rolled back too.
  - `geometry`: the fillet is too large, the selector matched nothing, a cut misses the body, or a profile is open or self-intersecting. Reduce sizes, change the selector, or split the batch to isolate the cause.
  - `bad_request`: wrong field, unknown id, bad expression, undefined parameter, unknown hole size or material name (the message lists valid choices). Call `kadvia:get_part` and `kadvia:modeling_reference`.
- After a timeout or lost connection, call `kadvia:get_part` and check the `revision` before retrying.
- *Kadvia is not running* → ask the user to open the app.

More: [references/troubleshooting.md](references/troubleshooting.md).

## Inspecting existing models (STEP)
- Overall size comes from the bbox `size` in `kadvia:get_model_info` (X × Y × Z mm).
- Exact distances, radii and angles: `kadvia:measure` (omit `a`/`b` to measure the user's selection). Imported models are measured on their mesh (`exact: false`).
- Volume is reported per body and in `totals`. It is missing when a body is not a closed solid; say so.
- Mass comes from the assigned material (`kadvia:set_material`, then `kadvia:mass_properties`), or from `density_g_cm3` for a quick what-if (steel 7.85 g/cm³, aluminium 6061 2.70), or by hand: volume × density.
- Selection areas and lengths from `kadvia:get_selection` come from the display mesh, so say "about" for curved faces, or use `kadvia:measure`. A selected vertex (`kind: "vertex"`) comes with its `point`; measure it as `{"kind": "point", "point": [x, y, z]}`.
- Look before you answer: call `kadvia:render_views` whenever the answer depends on the shape.

Example: [examples/inspect-a-step-file.md](examples/inspect-a-step-file.md).
