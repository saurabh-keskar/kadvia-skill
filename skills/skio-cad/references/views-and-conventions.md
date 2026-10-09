# Views and conventions

## Coordinate system
- Right-handed, **Z up**, units millimetres.
- Bounding boxes are `{min:[x,y,z], max:[x,y,z]}`; `size` = max − min.

## Standard views (`skio:set_view`, `skio:render_views`)
| View | Camera looks along | Screen right | Screen up |
|---|---|---|---|
| `front` | +Y (camera on −Y) | +X | +Z |
| `back` | −Y | −X | +Z |
| `right` | −X (camera on +X) | +Y | +Z |
| `left` | +X | −Y | +Z |
| `top` | −Z (camera above) | +X | +Y |
| `bottom` | +Z | +X | −Y |
| `iso` | from (+X, −Y, +Z) toward the model | | +Z |

SolidWorks users often think Y-up ("Front plane = XY"). In Skio, the floor plane is XY and
height is Z. Translate when users describe directions in SolidWorks terms.

## Sketch planes (modeling)
| Skio plane | Faces the view | u, v | Normal | Offset `o` puts it at |
|---|---|---|---|---|
| `XY` | `top` | +X, +Y | +Z | Z = o |
| `XZ` | `front` | +X, +Z | −Y | Y = −o |
| `YZ` | `right` | +Y, +Z | +X | X = o |

SolidWorks "Top plane" ≈ Skio `XY`, "Front plane" ≈ Skio `XZ`, "Right plane" ≈ Skio `YZ`
(with Z up instead of Y up).

## Ids
- Models: `skio:new_part`, `skio:open_step_file`, `skio:list_models` (never guess).
- Features and parameters: the ids/names you chose, listed by `skio:get_part`.
- Bodies: `b0`, `b1`, ... in modeling results and `skio:get_part`.

## Display modes (`skio:set_display_mode`)
| Mode | Good for |
|---|---|
| `shaded-edges` | Default; reading shape and features |
| `shaded` | Smooth surfaces, appearance |
| `wireframe` | Seeing internal edges and hidden features |
| `hidden-lines` | Drawing-like line views |

## Screenshot tips (`skio:render_views`)
- Default 800×600 is enough for shape checks; use 1200–2048 px only for small details.
- Ask for the views that answer the question (e.g. `top` for hole patterns, `front`/`right` for heights).
- Renders fit the whole scene. If several models are open, close or mention the others.
