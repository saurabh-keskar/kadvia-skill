# Example: 3D-printed electronics enclosure (text-to-CAD)

Covers derived parameters, a cavity cut from a sketch on an offset plane, rounded corners
from `cornerRadius` instead of fillets, screw bosses from a multi-circle sketch, blind pilot
holes, a cut-out on the XZ (front) plane, and print-oriented design rules.

**User:** "I need a PETG box for electronics: 80 × 50 × 30 mm outside, 2 mm walls, open top,
four corner bosses for M3 self-tapping lid screws, and a USB-C opening in the front."

## Plan
- Put the base on Z = 0 and centre the box on X/Y. Front = the −Y face, matching Kadvia's front view.
- **Shell:** an outer rounded rectangle extruded to the full height, minus an inner rounded
  rectangle cut down from the top, leaving the floor. Inner corner radius = `corner_r − wall`
  keeps the wall uniform.
- **Bosses:** Ø7 circles on the floor plane, pushed 1 mm into the walls (overlap, not tangency),
  extruded to the rim. They get Ø2.5 × 10 mm pilot holes (the M3 self-tapping value from the
  design rules).
- **USB-C:** a 9.5 × 3.5 mm slot sketched on the front face. That face is `XZ` with offset
  `depth/2`, because XZ offsets go toward −Y. Cut toward +Y through the wall.
- **Print rules:** 2 mm walls (above the 1.2 mm FDM minimum) and a 0.5 mm bottom chamfer against
  elephant's foot.

## Calls
1. `kadvia:new_part {"name": "Electronics box"}` → `m1`.
2. Parameters (including derived ones):

```json
{"model_id": "m1", "operations": [
  {"op": "set_parameter", "name": "length", "value": 80, "description": "Outside length (X)"},
  {"op": "set_parameter", "name": "depth", "value": 50, "description": "Outside depth (Y)"},
  {"op": "set_parameter", "name": "height", "value": 30, "description": "Outside height (Z)"},
  {"op": "set_parameter", "name": "wall", "value": 2},
  {"op": "set_parameter", "name": "floor", "value": 2},
  {"op": "set_parameter", "name": "corner_r", "value": 4, "description": "Outside corner radius"},
  {"op": "set_parameter", "name": "boss_d", "value": 7},
  {"op": "set_parameter", "name": "pilot_d", "value": 2.5, "description": "M3 self-tapping pilot"},
  {"op": "set_parameter", "name": "pilot_depth", "value": 10},
  {"op": "set_parameter", "name": "usb_w", "value": 9.5},
  {"op": "set_parameter", "name": "usb_h", "value": 3.5},
  {"op": "set_parameter", "name": "usb_z", "value": 8, "description": "USB opening centre height"},
  {"op": "set_parameter", "name": "inner_l", "value": "length - 2*wall"},
  {"op": "set_parameter", "name": "inner_d", "value": "depth - 2*wall"},
  {"op": "set_parameter", "name": "boss_x", "value": "inner_l/2 - boss_d/2 + 1"},
  {"op": "set_parameter", "name": "boss_y", "value": "inner_d/2 - boss_d/2 + 1"}
]}
```

3. Shell:

```json
{"model_id": "m1", "operations": [
  {"op": "add_feature", "feature": {"id": "outer_sk", "type": "sketch", "plane": {"base": "XY"},
    "profiles": [{"kind": "rect", "center": [0, 0], "width": "length", "height": "depth",
      "cornerRadius": "corner_r"}]}},
  {"op": "add_feature", "feature": {"id": "outer", "type": "extrude", "sketch": "outer_sk",
    "distance": "height"}},
  {"op": "add_feature", "feature": {"id": "cavity_sk", "type": "sketch",
    "plane": {"base": "XY", "offset": "height"},
    "profiles": [{"kind": "rect", "center": [0, 0], "width": "inner_l", "height": "inner_d",
      "cornerRadius": "corner_r - wall"}]}},
  {"op": "add_feature", "feature": {"id": "cavity", "type": "extrude", "sketch": "cavity_sk",
    "distance": "height - floor", "direction": "reverse", "operation": "cut"}}
]}
```

Check: there is 1 body with bbox size **[80, 50, 30]**. The volume is about 119 588 − 97 792 ≈
**21 796 mm³**. Render `iso`: you should see an open box with a 2 mm floor.

4. Bosses and pilot holes:

```json
{"model_id": "m1", "operations": [
  {"op": "add_feature", "feature": {"id": "boss_sk", "type": "sketch",
    "plane": {"base": "XY", "offset": "floor"},
    "profiles": [
      {"kind": "circle", "center": ["-boss_x", "-boss_y"], "diameter": "boss_d"},
      {"kind": "circle", "center": ["boss_x", "-boss_y"], "diameter": "boss_d"},
      {"kind": "circle", "center": ["boss_x", "boss_y"], "diameter": "boss_d"},
      {"kind": "circle", "center": ["-boss_x", "boss_y"], "diameter": "boss_d"}
    ]}},
  {"op": "add_feature", "feature": {"id": "bosses", "type": "extrude", "sketch": "boss_sk",
    "distance": "height - floor", "operation": "add"}},
  {"op": "add_feature", "feature": {"id": "pilots", "type": "hole",
    "plane": {"base": "XY", "offset": "height"},
    "points": [["-boss_x", "-boss_y"], ["boss_x", "-boss_y"], ["boss_x", "boss_y"], ["-boss_x", "boss_y"]],
    "diameter": "pilot_d", "depth": "pilot_depth"}}
]}
```

Check: the bosses add about 4 × 30.9 mm² × 28 ≈ 3 460 mm³ (only the part inside the cavity
counts) and the pilots remove about 196 mm³, giving **≈ 25 060 mm³**. Render `top` to see 4
bosses with holes, merged into the corners.

5. USB-C opening in the front wall:

```json
{"model_id": "m1", "operations": [
  {"op": "add_feature", "feature": {"id": "usb_sk", "type": "sketch",
    "plane": {"base": "XZ", "offset": "depth/2"},
    "profiles": [{"kind": "slot", "center": [0, "usb_z"], "length": "usb_w - usb_h", "width": "usb_h"}]}},
  {"op": "add_feature", "feature": {"id": "usb_cut", "type": "extrude", "sketch": "usb_sk",
    "distance": "2*wall", "direction": "reverse", "operation": "cut"}}
]}
```

Check: the volume drops by about 61 mm³ (30.6 mm² × 2 mm) to **≈ 24 999 mm³**. Render
`front`: the slot should be centred in X with its centre 8 mm above the base. If the volume
does not change, the cut went toward −Y (into air): check the direction.

6. Bottom chamfer (print rule):

```json
{"model_id": "m1", "operations": [
  {"op": "add_feature", "feature": {"id": "bottom_chamfer", "type": "chamfer",
    "edges": {"plane": "min_z"}, "distance": 0.5}}
]}
```

Check: the feature is `ok` and the volume drops by about 32 mm³ to **≈ 24 967 mm³**.

7. `kadvia:render_views {"views": ["iso", "front", "top", "right"]}` and
   `kadvia:mass_properties {"model_id": "m1", "density_g_cm3": 1.27}` → about 31.7 g as solid
   PETG. A printed part is lighter, depending on infill.
8. When the user agrees: `kadvia:save_part {"model_id": "m1", "path": "~/Documents/electronics-box.kadvia"}`
   and `kadvia:export_model {"model_id": "m1", "path": "~/Documents/electronics-box.stl"}`.

## Report (excerpt)
> Enclosure m1: 80 × 50 × 30 mm outside, 2 mm walls and floor, R4 outside corners, 4 corner
> bosses with Ø2.5 × 10 mm pilots for M3 self-tapping screws, a 9.5 × 3.5 mm USB-C slot at
> 8 mm height in the front wall, and a 0.5 mm bottom chamfer. The volume is about 24 967 mm³
> (about 32 g solid PETG). Change `length`, `depth`, `height`, `wall` or `usb_z` and everything
> follows. A lid is a separate part; I can model one that fits.
