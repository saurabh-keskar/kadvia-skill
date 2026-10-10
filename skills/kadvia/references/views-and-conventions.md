# Views and conventions

## Coordinate system
- Right-handed, **Z up**, units millimetres.
- Bounding boxes are `{min:[x,y,z], max:[x,y,z]}`; `size` = max − min.

## Standard views (`kadvia:set_view`, `kadvia:render_views`)
| View | Camera looks along | Screen right | Screen up |
|---|---|---|---|
| `front` | +Y (camera on −Y) | +X | +Z |
| `back` | −Y | −X | +Z |
| `right` | −X (camera on +X) | +Y | +Z |
| `left` | +X | −Y | +Z |
| `top` | −Z (camera above) | +X | +Y |
| `bottom` | +Z | +X | −Y |
| `iso` | from (+X, −Y, +Z) toward the model | | +Z |

Some CAD tools are Y-up (their "front plane" is XY). In Kadvia the floor plane is XY and
height is Z. Translate when users describe directions in Y-up terms.

## Sketch planes (modeling)
| Kadvia plane | Faces the view | u, v | Normal | Offset `o` puts it at |
|---|---|---|---|---|
| `XY` | `top` | +X, +Y | +Z | Z = o |
| `XZ` | `front` | +X, +Z | −Y | Y = −o |
| `YZ` | `right` | +Y, +Z | +X | X = o |

A "top plane" is Kadvia `XY`, a "front plane" is Kadvia `XZ` and a "right plane" is Kadvia `YZ`
(with Z up).

## Ids
- Models: `kadvia:new_part`, `kadvia:new_assembly`, `kadvia:new_drawing`, `kadvia:convert_to_part`, `kadvia:open_step_file`, `kadvia:list_models` (never guess).
- Model kinds: `part` (`.kadvia`), `imported` (STEP), `assembly` (`.kasm`), `drawing` (`.kdraw`).
- Features and parameters: the ids/names you chose, listed by `kadvia:get_part`.
- Bodies: `b0`, `b1`, ... in modeling results and `kadvia:get_part`; in assemblies `<componentId>/<bodyId>`; inside a sub-assembly the body is `<child>.<body>` (`s1/plate.b0`).
- Face and edge ids (from `kadvia:get_selection`, `kadvia:recognize_features`, `kadvia:check_design`) belong to the current geometry and change after edits: use them right away (for example in `kadvia:measure`), and use selectors in saved features.
- Selected vertices (`kind: "vertex"`) come with their `point` [x, y, z]; their ids are only valid for the current revision. Measure them as `{"kind": "point", "point": [...]}` or use the point in a `near` selector.
- Drawing edge ids (`"b0:e12"`; assembly drawings `"bolt/b0:e12"`) come from `kadvia:get_drawing`.

## Display modes (`kadvia:set_display_mode`)
| Mode | Good for |
|---|---|
| `shaded-edges` | Default; reading shape and features |
| `shaded` | Smooth surfaces, colours and finishes set with `kadvia:set_appearance` |
| `wireframe` | Seeing internal edges and hidden features |
| `hidden-lines` | Drawing-like line views |

## Screenshot tips (`kadvia:render_views`)
- Default 800×600 is enough for shape checks; use 1200–2048 px only for small details.
- Ask for the views that answer the question (e.g. `top` for hole patterns, `front`/`right` for heights).
- Renders fit the whole scene. If several models are open, close or mention the others.
- `section: {"axis": "x" | "y" | "z", "offset"?, "flip"?}` looks inside the part (cut faces are capped); `{"axis": "off"}` renders without the user's section view.
- `include_references: true` shows reference images (for image-to-CAD tracing); they are hidden by default.
- `background: "white"` gives drawing-like images; `"transparent"` a PNG with alpha.
- `quality: "render"` gives a beauty render (materials and appearances, environment lighting, soft shadow, no edges) for product shots, with `environment` `studio` (default), `outdoor` or `warehouse`; keep the default `standard` quality for checking geometry. See [materials-appearance.md](materials-appearance.md#7-product-renders).
