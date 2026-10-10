# Troubleshooting

| Error / symptom | Meaning | What to do |
|---|---|---|
| "Kadvia is not running" | The desktop app is closed | Ask the user to open Kadvia, then retry |
| `ui_unavailable` | App is starting or the window is not ready | Wait a few seconds and retry once |
| `not_found` | File path or model id does not exist | Use an absolute path; call `kadvia:list_models` for ids |
| `bad_request` | Invalid argument (view name, size, relative path) | Fix the argument as the hint says |
| `unsupported` | File type not supported, or the tool doesn't apply to this kind of model | Kadvia opens `.step`/`.stp`, `.kadvia`, `.kasm` and `.kdraw`. Modeling tools need kind `part`: convert STEP geometry with `kadvia:convert_to_part`. `kadvia:check_design` doesn't run on assemblies |
| `parse` | The file is not valid STEP or `.kadvia` | Ask the user for a re-export from their CAD tool (AP214 or AP242) |
| `geometry` | No displayable bodies, or import failed | Render, report what is visible, suggest re-export |
| `panic` | The kernel hit geometry it cannot handle | The app is fine; report the file name to the user |
| `timeout` | The window did not answer in time | Retry once; very large files may need longer |
| Warning "faces could not be tessellated" | Some faces will be missing from the display | Render and describe gaps; volume may be omitted |
| Warning "entities could not be converted" | Some STEP data was skipped | Usually colours/annotations; geometry may still be complete |
| Volume missing | Body is not a closed solid (surfaces or gaps) | Say so; do not invent a value |

## Modeling errors (`kadvia:apply_operations`, `kadvia:set_parameters`)
A failed batch changes **nothing**. The error names `operations[i]` and the feature id. Fix
that operation and resend the **whole corrected batch**.

| Error / symptom | Likely cause | Fix |
|---|---|---|
| `bad_request` "unknown feature/sketch id" | Typo, a feature added without `id`, or an id from another part | `kadvia:get_part` for the real ids; always set `id` on new features |
| `bad_request` "unknown parameter" / expression error | Parameter not defined yet (or defined later in the batch), a unit in the string (`"10mm"`), or an unsupported function | Define parameters first; use plain expressions (see `kadvia:modeling_reference` topic `expressions`) |
| `bad_request` invalid op | Missing `feature` wrapper, wrong field name, wrong type | Compare with [modeling-operations.md](modeling-operations.md) |
| `geometry` on a fillet/chamfer | Radius too big for the neighbouring faces, or the selector matched edges you did not intend | Smaller radius; narrow the selector (`plane`, `type`, `radius`); fillet before adding holes |
| `geometry` "selector matched no edges" | The selector describes edges that do not exist (yet) | Check bbox planes and radii in the last result; mind feature order |
| `geometry` on a cut/hole | Plane on the wrong side (XZ offset goes to −Y; holes drill against the normal), or the cut misses the body | Put the plane on the entry face; use `"direction": "reverse"` |
| `geometry` on an extrude/revolve | Open or self-intersecting profile; revolve profile crosses the axis | Check path points; keep the revolve profile on one side of the axis |
| Volume unchanged after a cut | The cut went into air (wrong direction or plane) | Flip `direction`; check the plane offset sign |
| bbox twice/half the expected size | Radius vs diameter mix-up, symmetric vs normal extrude, wrong parameter | Read `kadvia:get_part`, fix the parameter |
| Two bodies where one was expected | A feature used `"operation": "new"`, or a boss does not touch the body | Use `"add"`; overlap by 0.5–1 mm |
| Timeout during modeling | Heavy regeneration (many fillets/patterns) | `kadvia:get_part` to see whether the revision changed before retrying; split the batch |
| `undo` says nothing to undo | No committed change since the part was opened | Fine; nothing to revert |
| Features show `skipped` / "rolled back" | The rollback bar is above them | `{"op": "set_rollback"}` rolls forward to the end |

## Imported STEP edits
| Error / symptom | Likely cause | Fix |
|---|---|---|
| `kadvia:convert_to_part` fails, nothing created | The file has no closed solid the kernel can edit (surface model, entities lost in conversion) | It stays view/measure/export only; offer to rebuild it as a parametric part from measurements |
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
| DXF export wrote several files | The drawing has several sheets | Expected (`name-1.dxf`, …); pass `sheet` for one file |
| An older Kadvia can't open the `.kdraw` | It uses newer drawing features (sheets, tolerances, GD&T, assembly sources) | Send PDF/DXF, or have the other person update Kadvia |
