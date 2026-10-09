# Example: Hub flange (text-to-CAD)

Covers a sketch + extrude, a cylinder primitive with `add`, a through hole on the axis, a
circular pattern of holes, chamfers selected by radius, and clearance checks from the design
rules.

**User:** "Design a steel flange: OD 140, 14 mm thick, with a hub of Ø70 × 20 mm, a 50 mm bore,
and 6 × M10 bolt holes on a 100 mm PCD. Break the edges."

## Plan
- Centre the part on the Z axis with the flange face on Z = 0 and the hub on top.
- Build order: disc (sketch + extrude) → hub (cylinder, add) → bore (hole through from the hub top)
  → one bolt hole → circular pattern → 1 mm chamfers on the outer and bore edges.
- Check the design rules before modeling:
  - **Washer clearance to the hub:** PCD/2 − washer OD/2 = 50 − 10 = 40, which is at least
    hub_d/2 + 1 = 36. ✓
  - **Edge distance:** (140 − 100)/2 = 20, which is at least 1.5 × 11 = 16.5. ✓

## Calls
1. `skio:new_part {"name": "Hub flange"}` → `m1`.
2. Parameters:

```json
{"model_id": "m1", "operations": [
  {"op": "set_parameter", "name": "flange_d", "value": 140, "description": "Flange outer diameter"},
  {"op": "set_parameter", "name": "flange_t", "value": 14, "description": "Flange thickness"},
  {"op": "set_parameter", "name": "hub_d", "value": 70},
  {"op": "set_parameter", "name": "hub_h", "value": 20, "description": "Hub height above the flange"},
  {"op": "set_parameter", "name": "bore_d", "value": 50},
  {"op": "set_parameter", "name": "pcd", "value": 100, "description": "Bolt circle diameter"},
  {"op": "set_parameter", "name": "bolt_d", "value": 11, "description": "M10 clearance (medium)"},
  {"op": "set_parameter", "name": "bolt_count", "value": 6, "unit": ""},
  {"op": "set_parameter", "name": "edge_break", "value": 1}
]}
```

3. Disc, hub and bore:

```json
{"model_id": "m1", "operations": [
  {"op": "add_feature", "feature": {"id": "disc_sk", "type": "sketch", "plane": {"base": "XY"},
    "profiles": [{"kind": "circle", "center": [0, 0], "diameter": "flange_d"}]}},
  {"op": "add_feature", "feature": {"id": "disc", "type": "extrude", "sketch": "disc_sk",
    "distance": "flange_t"}},
  {"op": "add_feature", "feature": {"id": "hub", "type": "cylinder", "radius": "hub_d/2",
    "height": "hub_h", "base": [0, 0, "flange_t"], "axis": "Z", "operation": "add"}},
  {"op": "add_feature", "feature": {"id": "bore", "type": "hole",
    "plane": {"base": "XY", "offset": "flange_t + hub_h"}, "points": [[0, 0]],
    "diameter": "bore_d", "through": true}}
]}
```

Check: there is 1 body with bbox size **[140, 140, 34]** centred on X/Y. The volume is
π·70²·14 + π·35²·20 − π·25²·34 ≈ **225 723 mm³**.

4. Bolt holes:

```json
{"model_id": "m1", "operations": [
  {"op": "add_feature", "feature": {"id": "bolt_hole", "type": "hole",
    "plane": {"base": "XY", "offset": "flange_t"}, "points": [["pcd/2", 0]],
    "diameter": "bolt_d", "through": true}},
  {"op": "add_feature", "feature": {"id": "bolt_ring", "type": "circular_pattern",
    "features": ["bolt_hole"], "axis": {"origin": [0, 0, 0], "direction": [0, 0, 1]},
    "count": "bolt_count"}}
]}
```

Check: each hole removes π·5.5²·14 ≈ 1 330 mm³, so 6 holes bring the volume to
**≈ 217 741 mm³**. Render `top` and count **6** evenly spaced holes clear of the hub.

5. Edge breaks:

```json
{"model_id": "m1", "operations": [
  {"op": "add_feature", "feature": {"id": "outer_chamfer", "type": "chamfer",
    "edges": {"type": "circle", "radius": "flange_d/2"}, "distance": "edge_break"}},
  {"op": "add_feature", "feature": {"id": "bore_chamfer", "type": "chamfer",
    "edges": {"type": "circle", "radius": "bore_d/2"}, "distance": "edge_break"}}
]}
```

Check: each selector matches 2 edges (top and bottom). The volume drops by about 600 mm³ to
**≈ 217 144 mm³**.

6. `skio:render_views {"views": ["iso", "top", "front"]}` and
   `skio:mass_properties {"model_id": "m1", "density_g_cm3": 7.85}` → about **1.70 kg**.

## Report (excerpt)
> Flange m1: Ø140 × 14 mm with a Ø70 × 20 mm hub (34 mm overall), a Ø50 bore and 6 × Ø11
> holes on a Ø100 PCD, all edges chamfered 1 mm. The volume is 217 144 mm³, about 1.70 kg in
> steel. The parameters include `bolt_count`, `pcd`, `bolt_d`, `hub_d`, `hub_h`, `bore_d` and
> `flange_t`.

## Follow-up edit
**User:** "Use 8 bolts instead."

```json
{"model_id": "m1", "values": {"bolt_count": 8}}
```
(`skio:set_parameters`) The volume should drop by 2 × 1 330 ≈ 2 661 mm³. Render `top` and count 8.
