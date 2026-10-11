# Troubleshooting

| Error / symptom | Meaning | What to do |
|---|---|---|
| "Kadvia is not running" | The desktop app is closed | Ask the user to open Kadvia, then retry |
| `ui_unavailable` | App is starting or the window is not ready | Wait a few seconds and retry once |
| `not_found` | File path or model id does not exist | Use an absolute path; call `kadvia:list_models` for ids |
| `bad_request` | Invalid argument (view name, size, relative path) | Fix the argument as the hint says |
| `unsupported` | File type not supported, or the tool doesn't apply to this kind of model | Kadvia opens `.step`/`.stp`, `.kadvia`, `.kasm` and `.kdraw`; DXF files go through `kadvia:import_dxf`. Modeling tools need kind `part`: convert STEP geometry with `kadvia:convert_to_part`. `kadvia:check_design` doesn't run on assemblies |
| `parse` | The file is not valid STEP or `.kadvia` | Ask the user for a re-export from their CAD tool (AP214 or AP242) |
| `geometry` | No displayable bodies, or import failed | Render, report what is visible, suggest re-export |
| `panic` | The kernel hit geometry it cannot handle | The app is fine; report the file name to the user |
| `timeout` | The window did not answer in time | Retry once; very large files may need longer |
| Warning "faces could not be tessellated" | Some faces will be missing from the display | Render and describe gaps; volume may be omitted |
| Warning "entities could not be converted" | Some STEP data was skipped | Usually colours/annotations; geometry may still be complete |
| Volume missing | Body is not a closed solid (surfaces or gaps) | Say so; do not invent a value |
| Body volume differs from `kadvia:mass_properties` | It should not: both are the same kernel integral (a mesh-only body is the one exception, it has no exact volume) | Re-read `kadvia:get_model_info`; report the body as mesh-only if `exact: false` |

## Modeling errors (`kadvia:apply_operations`, `kadvia:set_parameters`)
A failed batch changes **nothing**. The error names `operations[i]` and the feature id. Fix
that operation and resend the **whole corrected batch**.

| Error / symptom | Likely cause | Fix |
|---|---|---|
| `bad_request` "unknown feature/sketch id" | Typo, a feature added without `id`, or an id from another part | `kadvia:get_part` for the real ids; always set `id` on new features |
| `bad_request` "unknown parameter" / expression error | Parameter not defined yet (or defined later in the batch), a unit in the string (`"10mm"`), or an unsupported function | Define parameters first; use plain expressions (see `kadvia:modeling_reference` topic `expressions`) |
| `bad_request` invalid op | Missing `feature` wrapper, wrong field name, wrong type | Compare with [modeling-operations.md](modeling-operations.md) |
| `geometry` on a fillet/chamfer | Radius too big for the neighbouring faces, or the selector matched edges you did not intend | Smaller radius; narrow the selector (`plane`, `type`, `radius`); fillet before adding holes |
| `geometry` on a fillet at a corner with a curved edge | Three or more selected edges meet and one is curved (or a face there is not flat) | Fillet the curved edges in a separate feature; mixed convex/concave corners between flat faces are fine in one feature |
| `geometry` "share part of a curved face" | Two coaxial cylinders (or spheres) of one radius overlap only partly | Make one 0.01 mm larger, or let one contain the other |
| `geometry` "selector matched no edges" | The selector describes edges that do not exist (yet) | Check bbox planes and radii in the last result; mind feature order |
| `geometry` on a cut/hole | Plane on the wrong side (XZ offset goes to −Y; holes drill against the normal), or the cut misses the body | Put the plane on the entry face; use `"direction": "reverse"` |
| `geometry` on an extrude/revolve | Open or self-intersecting profile; revolve profile crosses the axis | Check path points; keep the revolve profile on one side of the axis |
| Volume unchanged after a cut | The cut went into air (wrong direction or plane) | Flip `direction`; check the plane offset sign |
| bbox twice/half the expected size | Radius vs diameter mix-up, symmetric vs normal extrude, wrong parameter | Read `kadvia:get_part`, fix the parameter |
| Two bodies where one was expected | A feature used `"operation": "new"`, or a boss does not touch the body | Use `"add"`; overlap by 0.5–1 mm |
| Timeout during modeling | Heavy regeneration (many fillets/patterns) | `kadvia:get_part` to see whether the revision changed before retrying; split the batch |
| `undo` says nothing to undo | No committed change since the part was opened | Fine; nothing to revert |
| Features show `skipped` / "rolled back" | The rollback bar is above them | `{"op": "set_rollback"}` rolls forward to the end |

## Holes and threads
| Error / symptom | Likely cause | Fix |
|---|---|---|
| `bad_request` unknown size / invalid pitch | A size outside the tables (`"M7"`, `"M8x0.9"`) | Use a size the message lists; fine pitches as `"M8x1"`; inch as `"1/4-20 UNC"` |
| `bad_request` no table value for the counterbore/countersink | The entry feature has no table entry for that size (for example a counterbore for M18) | Give `counterbore: {"diameter": ..., "depth": ...}` (or `countersink: {...}`) explicitly |
| A blind hole breaks through the bottom | Drill depth (thread + 3 pitches, plus the 118° point) exceeds the thickness | Smaller `threadDepth`, explicit `depth`, `drillPoint: 0`, or make it `through: true` |
| A note says the drill point was left flat | The point would barely break through a face | Fine, or change the depth |
| Modeled thread fails | Very fine pitch for the part, crosses another feature, or a chamfered entry | Use a cosmetic thread (default) |
| `thread` feature refused | The face is not one cylinder, or its diameter doesn't fit any standard thread | Select one hole wall or shaft; give `size`; internal threads need a tap-drill-sized hole |
| A face plane `{"normal": "+Z"}` is refused (several faces) | Counterbore floors and steps also face up; a face plane needs exactly one face | Add `"plane": "max_z"` or a `near` point |

## Materials and renders
| Error / symptom | Likely cause | Fix |
|---|---|---|
| `bad_request` unknown material | Name not in the library | `kadvia:list_materials {"query": ...}`, or a custom `{"name", "density"}` |
| `bad_request` on `set_appearance` | Invalid colour, a value outside 0–1, opacity 0, `faces` without `body_id`, or a face selector that matches nothing | Fix the field as the message says |
| `mass_properties` has no mass, lists `unassigned` | Some bodies have no material | `kadvia:set_material` (part or body), or pass `density_g_cm3` |
| Render looks plain grey | No material or appearance set | Set a material and/or appearance first |
| `environment` refused | It only applies to beauty renders | Add `"quality": "render"` |
| Another part appears in the product shot | `scene: true` was passed, or the model you meant is not the active one | Drop `scene` and pass `model_id`; never close the user's models for a clean render |
| The lid hides the inside in the shot | All bodies of the model are rendered | `"hide": ["<lid body id>"]` or `"isolate": {"bodies": [...]}` (ids from `kadvia:get_model_info`), for the images only |
| Face colour disappeared after an edit | A face style used `ids`, which changed | Use `normal` / `plane` / `surface` selectors |

## Sketches and DXF
| Error / symptom | Likely cause | Fix |
|---|---|---|
| `edit_sketch` trim/split refused on a slot or text | Slots, arc slots and text must be exploded first | `{"edit": "explode", "entity": ...}`, then trim |
| `edit_sketch` result has `notes` | Relations that no longer held were removed | Expected; re-add the dimensions the user needs |
| Imported DXF gives 0 regions / extrude fails | Gaps between end points larger than the tolerance, or open construction lines | Re-import with a larger `tolerance` (e.g. 0.05) or only the outline `layers` |
| Imported part 25.4× too big or small | The file has no or wrong units | Re-import with `"units": "in"` or `"mm"` (check `kadvia:inspect_dxf`) |
| Labels or title block became profiles | TEXT/MTEXT and border layers were imported | `"text": false`, or pick the outline `layers` |
| Many entities `skipped` | HATCH, DIMENSION, LEADER, 3D entities are not imported | Fine for outlines; tell the user what was left out |
| Text characters missing | No glyph in the bundled fonts | Use plain Latin characters |
| `bad_request` "DXF export writes one sketch" | `format: "dxf"` or a `.dxf` path without `sketch` | Add `sketch` (a sketch feature id from `kadvia:get_part`); a drawing with views uses `kadvia:export_drawing` |
| `bad_request` "`sketch` exports a 2D DXF, but the format is …" | `sketch` given with a `.step`/`.stl`/… path or format | Use a `.dxf` path, or drop `sketch` for a 3D export |
| Sketch DXF export refused for a sketch | The sketch has only `profiles` (no `entities`), or the model isn't a part | Export a sketch drawn with `entities` (or imported from DXF); for an outline of a solid make a drawing and `kadvia:export_drawing` |
| Exported DXF has extra lines | Construction geometry is written on layer `CONSTRUCTION` | Tell the user to hide or delete that layer before cutting |

## Imported STEP edits
| Error / symptom | Likely cause | Fix |
|---|---|---|
| `kadvia:convert_to_part` fails, nothing created | No shell of the file can be converted or displayed (open surface model, entities lost) | It stays view/measure/export only; offer to rebuild it as a parametric part from measurements |
| `kadvia:convert_to_part` warns "could not be converted to exact solids", `meshOnlyBodies` | Some shells only display (`exact: false`); they were kept, not dropped | View, measure and export them as usual; edit only the exact bodies (tell the user which) |
| `geometry` "body … is mesh-only" | A feature targets a mesh-only body | Target an exact body (`target`), or rebuild that body parametrically |
| `kadvia:recognize_features` finds fewer chamfers than expected | Strips at 90° to both neighbours (rib ends) and coplanar steps are planes, not chamfers; only strips tilted 10–80° bisecting a corner count | Use `include_planes: true` or `near` points to select those faces |
| `kadvia:check_design` hole `standard` reads "Ø5.5 (A or B)" | The diameter matches several table entries | Read `standardCandidates` and tell the user the options; for wizard holes the `feature` id gives the real kind |
| `import` feature `bodies` index out of range | Indices count the file's shells that yield a body (exact or mesh-only), from 0, in file order | Use the count from the error; body `b<i>` = shell `i` |
| `geometry` "not a blend strip or a feature face" | `delete_face` on a face that is neither a fillet/chamfer strip nor a whole hole/boss/pocket | Select the whole feature (all its faces), or use `offset_face`/`move_face` instead |
| `geometry` "face N cannot be extended" | A fillet next to a free-form or toroidal face, or a neighbour that can't grow | That blend can't be removed by extension; tell the user |
| `geometry` on `offset_face` / `move_face` | The edit would change which faces meet (pushed past another face, offset larger than a radius, feature moved off its face) | Use a smaller distance, or include the neighbouring faces in the selection |
| `geometry` on `resize_hole` | A selected face is not a cylinder, or the new size breaks the neighbours | Narrow the selector (`surface: "cylinder"`, `concave: true`, `diameter`) |
| Selector matches too much | `normal: "+Z"` also matches upward faces inside the part; a radius filter matches holes and fillets alike | Add `plane`, `concave`, `axis` or `near` |

## Assemblies
| Error / symptom | Likely cause | Fix |
|---|---|---|
| Batch refused, a mate named in `featureId` | The mate reference didn't resolve or conflicts with other mates | Check the selector in **part** coordinates; try `flip: true`; check `value` |
| Part lands upside down / on the wrong side | Normals opposed vs aligned | `update_mate` with `flip: true` |
| `dof` > 0 that you didn't expect | A motion is still free (`underConstrained` lists the components) | Add a `parallel`, `distance` or second `concentric` mate |
| A mate `redundant` | It is implied by others | Usually fine; remove duplicates |
| Interference sliver ~0.05 mm | Pin exactly the size of its hole, or two gear blanks touching at their pitch circles (mesh tolerance) | Ignore; real overlaps are much deeper |
| "a gear mate needs `ratio`" (or `rack_pinion` needs `value`, `symmetric` needs `c`, `width` needs `c` and `d`) | A required field of that mate type is missing | Add it: ratio = teeth A / teeth B; rack value = π × pitch diameter |
| Gears turn but drift off their axes, or `dof` is higher than expected | A gear or rack mate doesn't hold the axis | Hinge each gear (concentric or point-on-axis coincident + a face coincident) and put the rack on a slide |
| A limit mate never shows as constraining | Inside its range a limit mate adds nothing (by design) | Fine; it holds only at `min`/`max` |
| Mate into a sub-assembly picks the wrong body | References use the sub-assembly's coordinates; without `body` the nearest body wins | Use sub-assembly coordinates and `"body": "<child>.<body>"` |
| Sub-assembly can't be inserted from an open model | It has no file yet or has unsaved changes | Save the sub-assembly (`.kasm`) first, or use its `path` |
| `kadvia:solve_assembly` drag moves nothing (0 components moved) | The component is fully constrained or grounded, or the target asks for a motion its mates don't allow | Read `solve.dof` and `underConstrained`; drag a component that can move, with a `target` along its free motion |
| The dragged pose is gone afterwards | `kadvia:solve_assembly` never commits | Write the returned transform with `update_component`, or drive a mate with `update_mate` |
| `kadvia:mate_options` refused | `a` or `b` is not a mate reference with a `component` | Use the `add_mate` form: `{"component": "c1", "face": {...}}` (one of `face`, `edge`, `vertex`, `point`, `axis`, `plane`) |

## Viewing, selection and rebuild
| Error / symptom | Likely cause | Fix |
|---|---|---|
| `bad_request` "give either view or from, not both" | `kadvia:set_view` got a standard view and a direction | Keep one |
| `bad_request` "nothing to change" | `kadvia:set_view` got none of `view`, `from`, `projection`, `section`, `model_id` | Give at least one |
| `bad_request` `model_id` "needs fit" | `kadvia:set_view` got `model_id` with `fit: false` | Leave `fit` out (or true) when zooming to a model |
| `bad_request` `scene` with `model_id` / `isolate` / `hide` | `scene: true` renders everything and takes no model or body filter | Drop `scene`, or drop the filters |
| `not_found` on `kadvia:render_views` | Unknown `model_id`, or a body id in `isolate` / `hide` that the model doesn't have | `kadvia:list_models` for model ids, `kadvia:get_model_info` for body ids |
| `bad_request` "nothing left to render" | `hide` removed every body, or `isolate` listed none that exist | Hide fewer bodies or fix the ids |
| `bad_request` non-zero direction | A `from` of `[0, 0, 0]` (or non-numbers) | Use a real direction from the model toward the camera, e.g. `[0, 0, -1]` |
| The section view keeps appearing in renders | A section turned on with `kadvia:set_view` stays on | `kadvia:set_view {"section": {"axis": "off"}}`, or pass `"section": {"axis": "off"}` to `kadvia:render_views` |
| `bad_request` "a face needs its numeric id" | A `face`/`edge`/`vertex` item in `kadvia:set_selection` without `id` | Add the numeric id (only `body` items go without one) |
| `kadvia:set_selection` selected fewer items than sent | Unknown ids are skipped (ids changed after an edit, wrong `body_id`) | Get fresh ids (`kadvia:get_selection`, `kadvia:recognize_features`); assembly bodies are `"<componentId>/<bodyId>"` |
| `kadvia:rebuild_part` refused on an imported model | STEP models have no features | Close and reopen the file to reload it, or `kadvia:convert_to_part` |
| A part still shows the old geometry after a linked STEP file changed | Cached steps were reused | `kadvia:rebuild_part {"force": true}` re-reads files inserted with `"link": true` (STEP data stored in the part, the default, never follows the file: insert it again) |

## Drawings
| Error / symptom | Likely cause | Fix |
|---|---|---|
| Batch refused: dimension cannot be measured | Wrong edge id, or a ref that doesn't fit the `kind` (a circle for `vertical`, a line for `diameter`) | `kadvia:get_drawing` for the current ids; match the `kind` to the edge `shape` |
| A warning about a dimension whose edge disappeared | The part changed and the edge is gone | `remove_dimension` and add it again with a new id |
| `kadvia:save_part` on a drawing fails | The source part has no file yet | Save the part (`.kadvia`) first, then the drawing (`.kdraw`) |
| Views overlap or run into the title block | Automatic layout after adding views or changing the sheet | `move_view`, a bigger `sheet_size`, a smaller `scale`, or move views to a new sheet (`add_sheet`, `move_view {id, sheet}`) |
| Batch refused on a `fit` tolerance | The fit or size is outside the ISO 286 tables (letters D–P / d–p, IT5–IT11, ≤ 500 mm) | Use a supported fit, or a `deviation` tolerance with the values |
| Batch refused on a feature control frame | Datums on a form tolerance (flatness, …), or none on an orientation/runout tolerance; more than 3 datums | Remove or add `datums` to match the characteristic |
| Balloon shows the wrong or no number | The ref isn't on the intended component, or the item isn't in the BOM | Use an edge id with that component's prefix (`bolt/b0:e4.mid`); `auto_balloon` to fill missing items |
| `auto_balloon` refused on `spacing` or `side` | `spacing` outside 0–100 mm, or `side` not one of `around`, `left`, `right`, `top`, `bottom` | Fix the value (default spacing 2 mm) |
| Balloons crowd one corner or overlap the BOM | Many items on one side of the view | `auto_balloon {"replace": true, "spacing": 4}` or `"side": "left"` / `"right"`; placement stays clear of views, tables and the title block |
| DXF export wrote several files | The drawing has several sheets | Expected (`name-1.dxf`, …); pass `sheet` for one file |
| An older Kadvia can't open the `.kdraw` | It uses newer drawing features (sheets, tolerances, GD&T, assembly sources) | Send PDF/DXF, or have the other person update Kadvia |
