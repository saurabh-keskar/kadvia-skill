# Troubleshooting

| Error / symptom | Meaning | What to do |
|---|---|---|
| "Kadvia is not running" | The desktop app is closed | Ask the user to open Kadvia, then retry |
| `ui_unavailable` | App is starting or the window is not ready | Wait a few seconds and retry once |
| `not_found` | File path or model id does not exist | Use an absolute path; call `kadvia:list_models` for ids |
| `bad_request` | Invalid argument (view name, size, relative path) | Fix the argument as the hint says |
| `unsupported` | File type not supported, or editing a non-part | `.step`/`.stp`/`.kadvia` can be opened; only parts (kind `part`) can be edited, so rebuild STEP geometry as a new part |
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
