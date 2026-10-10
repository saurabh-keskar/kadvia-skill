# Example: L-bracket (text-to-CAD)

Covers a sketch on the XZ plane, a symmetric extrude, bend fillets with `near` selectors,
holes on two different planes, derived parameters and a parametric edit.

**User:** "Make an L-bracket from 5 mm aluminium: 60 mm wide, a 50 mm base leg and a 40 mm upright,
5 mm inside bend radius, two M5 holes in each leg."

## Plan
- Build an L-profile in the XZ plane (u = X, v = Z) and extrude it symmetrically along Y.
  - The base leg lies along +X on Z = 0.
  - The upright sits at X = 0..t.
- Fillet the inside bend with r and the outside bend with r + t, so the wall stays 5 mm.
  These two edges can't be described with `parallel`/`plane` alone, so `near` uses
  expression coordinates that follow the parameters.
- Add M5 medium-clearance holes (Ø5.5), 15 mm from the leg ends, spaced `width − 2·hole_edge`
  apart, so a change in width moves the holes with it.
- Do the fillets before the holes, as the gotchas recommend.

## Calls
1. `kadvia:kadvia_status`, then `kadvia:new_part {"name": "L-bracket"}` → model `m1`.
2. `kadvia:modeling_reference {}` (once per session).
3. Parameters:

```json
{"model_id": "m1", "operations": [
  {"op": "set_parameter", "name": "width", "value": 60, "description": "Bracket width (Y)"},
  {"op": "set_parameter", "name": "leg_x", "value": 50, "description": "Base leg length (X)"},
  {"op": "set_parameter", "name": "leg_z", "value": 40, "description": "Upright height (Z)"},
  {"op": "set_parameter", "name": "t", "value": 5, "description": "Plate thickness"},
  {"op": "set_parameter", "name": "bend_r", "value": 5, "description": "Inside bend radius"},
  {"op": "set_parameter", "name": "hole_d", "value": 5.5, "description": "M5 clearance (medium)"},
  {"op": "set_parameter", "name": "hole_inset", "value": 15, "description": "Hole centre from leg end"},
  {"op": "set_parameter", "name": "hole_edge", "value": 15, "description": "Hole centre from side edge (Y)"},
  {"op": "set_parameter", "name": "hole_spacing", "value": "width - 2*hole_edge", "description": "Hole spacing (Y)"}
]}
```

4. Base solid:

```json
{"model_id": "m1", "operations": [
  {"op": "add_feature", "feature": {"id": "l_sk", "type": "sketch", "plane": {"base": "XZ"},
    "profiles": [{"kind": "path", "start": [0, 0], "segments": [
      {"line": ["leg_x", 0]}, {"line": ["leg_x", "t"]}, {"line": ["t", "t"]},
      {"line": ["t", "leg_z"]}, {"line": [0, "leg_z"]}
    ]}]}},
  {"op": "add_feature", "feature": {"id": "body", "type": "extrude", "sketch": "l_sk",
    "distance": "width", "direction": "symmetric"}}
]}
```

Check: every feature is `ok` and there is 1 body. bbox min [0, −30, 0], max [50, 30, 40], size
**[50, 60, 40]**. Volume = (50·5 + 35·5)·60 = **25 500 mm³**.

5. Bend fillets:

```json
{"model_id": "m1", "operations": [
  {"op": "add_feature", "feature": {"id": "inner_bend", "type": "fillet",
    "edges": {"near": [["t", 0, "t"]]}, "radius": "bend_r"}},
  {"op": "add_feature", "feature": {"id": "outer_bend", "type": "fillet",
    "edges": {"near": [[0, 0, 0]]}, "radius": "bend_r + t"}}
]}
```

Check: the volume drops by about 966 mm³ to **≈ 24 534 mm³** (the inside fillet adds 5.4 mm²
of area, the outside one removes 21.5 mm², over a width of 60 mm). The face count rises by 2.

6. Holes. The base-leg holes are drilled down from the top face of the base (XY at Z = t). The
upright holes are drilled from its inner face (YZ at X = t) toward −X:

```json
{"model_id": "m1", "operations": [
  {"op": "add_feature", "feature": {"id": "base_holes", "type": "hole",
    "plane": {"base": "XY", "offset": "t"},
    "points": [["leg_x - hole_inset", "-hole_spacing/2"], ["leg_x - hole_inset", "hole_spacing/2"]],
    "diameter": "hole_d", "through": true}},
  {"op": "add_feature", "feature": {"id": "upright_holes", "type": "hole",
    "plane": {"base": "YZ", "offset": "t"},
    "points": [["-hole_spacing/2", "leg_z - hole_inset"], ["hole_spacing/2", "leg_z - hole_inset"]],
    "diameter": "hole_d", "through": true}}
]}
```

Check: the volume drops by 4·π·2.75²·5 ≈ 475 mm³ to **≈ 24 059 mm³**.

7. `kadvia:render_views {"views": ["iso", "front", "top", "left"]}`. Check that `front` shows the
L-profile with both radii, `top` shows 2 holes in the base leg, and `left` shows 2 holes in
the upright.
8. `kadvia:mass_properties {"model_id": "m1", "density_g_cm3": 2.70}` gives a volume of about
24 059 mm³ and a mass of about **65 g**.

## Report (excerpt)
> Built the L-bracket (model m1): 60 × 50 × 40 mm, 5 mm thick, inside bend R5 and outside R10,
> with 4 × Ø5.5 M5 clearance holes. Its volume is 24 059 mm³, so it weighs about 65 g in 6061.
> You can change `width`, `leg_x`, `leg_z`, `t`, `bend_r`, `hole_d`, `hole_inset` and
> `hole_edge`; the hole spacing follows the width.

## Follow-up edit
**User:** "Make it 80 wide and use M6 screws."

```json
{"model_id": "m1", "values": {"width": 80, "hole_d": 6.6}}
```
(`kadvia:set_parameters`) Check that `changes` shows the size going from 50 × 60 × 40 to
**50 × 80 × 40**. `hole_spacing` is now 50, so render `top` and confirm the holes are still 15 mm
from the edges.

## Next steps
- **Design check:** `kadvia:check_design {"model_id": "m1", "process": "cnc"}` before sending it
  out (see [../references/design-check.md](../references/design-check.md)).
- **Drawing:** a dimensioned A4 drawing of this bracket is worked through in
  [../references/drawings.md](../references/drawings.md#9-worked-example-drawing-of-the-l-bracket).
