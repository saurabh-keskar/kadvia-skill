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
