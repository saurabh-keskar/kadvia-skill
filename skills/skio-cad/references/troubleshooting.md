# Troubleshooting

| Error / symptom | Meaning | What to do |
|---|---|---|
| "Skio CAD is not running" | The desktop app is closed | Ask the user to open Skio CAD, then retry |
| `ui_unavailable` | App is starting or the window is not ready | Wait a few seconds and retry once |
| `not_found` | File path or model id does not exist | Use an absolute path; call `skio:list_models` for ids |
| `bad_request` | Invalid argument (view name, size, relative path) | Fix the argument as the hint says |
| `unsupported` | File type not supported | Only `.step` / `.stp` can be opened today |
| `parse` | The file is not valid STEP | Ask the user for a re-export from their CAD tool (AP214 or AP242) |
| `geometry` | No displayable bodies, or import failed | Render, report what is visible, suggest re-export |
| `panic` | The kernel hit geometry it cannot handle | The app is fine; report the file name to the user |
| `timeout` | The window did not answer in time | Retry once; very large files may need longer |
| Warning "faces could not be tessellated" | Some faces will be missing from the display | Render and describe gaps; volume may be omitted |
| Warning "entities could not be converted" | Some STEP data was skipped | Usually colours/annotations; geometry may still be complete |
| Volume missing | Body is not a closed solid (surfaces or gaps) | Say so; do not invent a value |
