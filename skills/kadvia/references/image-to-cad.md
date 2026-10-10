# Image-to-CAD

Rebuilding a part from a photo, sketch, screenshot or drawing. There are two ways to use the
image, and they combine well:
- **Read it:** an AI model that accepts images looks at the picture and estimates dimensions.
- **Trace over it in Kadvia:** place the image file in the part as a **reference image** (a
  tracing underlay on a sketch plane), calibrate its scale from one known dimension, then
  sketch over it and compare the model with the picture in `kadvia:render_views`. This works
  even for text-only models, because positions come from the calibrated image coordinates.

The skill is in **extracting dimensions honestly** and making every guess easy to correct.

## Contents
1. [Workflow](#1-workflow)
2. [Reading the image](#2-reading-the-image)
3. [Finding the scale](#3-finding-the-scale)
4. [Estimating dimensions](#4-estimating-dimensions)
5. [Confirming with the user](#5-confirming-with-the-user)
6. [Modeling and verifying](#6-modeling-and-verifying)
7. [Common traps](#7-common-traps)
8. [Reference images in Kadvia](#8-reference-images-in-kadvia)
9. [Worked example: tracing a calibrated photo](#9-worked-example-tracing-a-calibrated-photo)

## 1. Workflow
- [ ] Classify the image: an orthographic drawing (with or without dimensions), a hand sketch, a photo (straight-on or perspective), or a render or screenshot of another CAD model.
- [ ] List the features (base shape, holes, slots, bosses, ribs, pockets, fillets, chamfers), the symmetry and the counts.
- [ ] Choose the construction (sketch → extrude, revolve, primitives) and the orientation in Kadvia.
- [ ] If the user has the image as a file: place it as a reference image on the matching plane (front view → `XZ`, top view → `XY`, side view → `YZ`) and **calibrate** it with one known dimension ([section 8](#8-reference-images-in-kadvia)).
- [ ] Find the scale; estimate every key dimension with its basis (from the calibrated image coordinates when you traced it).
- [ ] **Ask the user to confirm** the key dimensions (table below). Don't model guesses silently.
- [ ] Model with every dimension as a parameter, in small batches, sketching on the reference image's plane.
- [ ] Render the matching view with `include_references: true` and compare with the image; check counts and proportions.
- [ ] Report the assumptions, and say what is still unverified (hidden faces, depths, tolerances).

## 2. Reading the image
- **Base shape first.** What would you machine or print first? Usually that is a plate, block, cylinder or L-profile.
- **Features by type:**
  - Through holes (you can see light or background through them).
  - Blind holes and pockets (dark bottom).
  - Counterbores (a stepped ring), countersinks (a cone).
  - Slots, bosses, ribs, fillets (soft shading), chamfers (a narrow flat band).
- **Symmetry:** if the part looks symmetric, assume it is (and say so). That halves the number of guesses and lets you use `mirror` and patterns.
- **Counts and patterns:** count the holes and check whether they are equally spaced (linear) or on a circle (circular, with a PCD).
- **Hidden geometry:** the back, the inside and hole depths are often unknowable. List them as questions.
- **Standard parts:** recognise screws, bearings, connectors (USB-C, barrel jacks), profiles (2020 extrusion) and PCBs. Their sizes are known and make good scale references.

## 3. Finding the scale
In order of reliability:
1. **A dimension written on the drawing**, or one the user gave in the message.
2. **A ruler, calliper or cutting mat** visible in the photo, in the same plane as the feature.
3. **A known object** in the same plane:
   - Coins: euro 1 € Ø23.25 mm, US quarter Ø24.26 mm.
   - Credit card: 85.6 × 54.0 mm.
   - A4 paper: 210 × 297 mm.
4. **A standard feature** on the part itself:
   - Fastener heads: M3 socket head Ø5.5, M4 Ø7, M5 Ø8.5, M6 Ø10. Hex AF: M5 8 mm, M6 10 mm, M8 13 mm.
   - Connectors: USB-C receptacle opening about 9 × 3.3 mm. RJ45 opening about 11.7 × 14 mm.
   - 2020 extrusion: 20 mm. 608 bearing: OD 22, ID 8, width 7.
5. **Nothing:** you only have proportions. Ask the user for one overall dimension before modeling.

## 4. Estimating dimensions
- Measure in pixels on a face **seen square-on** and convert with the scale: `size = px × (ref_mm / ref_px)`.
- Perspective shortens edges that recede from the camera. Don't measure them. Use a different view or a symmetric partner instead.
- Estimate ratios (for example "hole Ø ≈ ¼ of the plate width") and use them as a sanity check.
- **Snap to sensible values:**
  - Whole millimetres (or 0.5 mm) for envelopes.
  - Standard hole sizes for fasteners (see [design-rules.md](design-rules.md)).
  - Standard plate and sheet thicknesses: 2, 3, 4, 5, 6, 8, 10, 12 mm; sheet 1, 1.5, 2 mm.
- Record each estimate with its **basis** and a confidence level: measured, derived, standard or guessed.

## 5. Confirming with the user
Before the first modeling batch, show a compact table and ask one question:

> From the photo I read the bracket as follows (scale from the M5 screw heads, Ø8.5 mm):
>
> | Parameter | Estimate | Basis |
> |---|---|---|
> | `width` | 60 mm | measured (7.1 × head Ø) |
> | `leg_x` / `leg_z` | 50 / 40 mm | measured |
> | `t` | 5 mm | guessed (standard plate) |
> | `hole_d` | 5.5 mm | standard M5 clearance |
> | `hole_spacing` | 30 mm | measured |
>
> The back face isn't visible, so I assumed no features there. Shall I model it with these
> values, or do you have the real dimensions?

- If the user says "just go", model with the estimates and repeat that they are estimates.
- If they give some values, use them and keep the rest as estimates.
- Their answer should take one line. Don't ask a separate question for every dimension.

## 6. Modeling and verifying
- **Orientation:**
  - Put the part's largest flat face or natural base on XY at Z = 0.
  - What the photo shows "from the front" should face −Y, so it matches Kadvia's `front` view.
  - A drawing's front view maps to Kadvia `front`, its top view to `top`, and its right-side view to `right`.
- Parameterise everything you estimated. Then a correction is a single `kadvia:set_parameters` call.
- **Verify:**
  - `kadvia:render_views` with the view that matches the image (`front` for a straight-on photo or drawing view, `iso` for a three-quarter photo). Compare the silhouette, feature positions and counts.
  - Check that ratios match: if the hole looks like ¼ of the width in the image, check `hole_d / width` in the model.
  - `kadvia:mass_properties` if the user knows the real weight. Matching mass is a strong check on thickness.
- In the report, list assumptions, unverified items and the parameters to adjust.

## 7. Common traps
- Guessing thickness from a photo taken straight on (the thickness is invisible), so ask.
- Mirrored images (selfie camera) and drawings in first-angle projection, where the views are swapped compared with third-angle. Check the projection symbol, or ask.
- Fillets seen in silhouette look like chamfers, and the other way round. Look at the shading.
- Countersunk vs counterbored holes. A cone vs a step changes the screw.
- Assuming a hole goes through when its far side is hidden.
- Using an object as the scale when it is closer to the camera than the part, which makes the part look smaller.

## 8. Reference images in Kadvia
A reference image is a picture placed on a plane of the part. It is stored in the part but
**never affects geometry** (or mass properties). The user sees it behind the model; you see it
only in `kadvia:render_views` with `"include_references": true`. Live reference:
`kadvia:modeling_reference {"topic": "references"}`.

Operations (in `kadvia:apply_operations`):

| `op` | Fields |
|---|---|
| `add_reference` | `reference` (below) |
| `update_reference` | `id`, `patch` (shallow merge; `null` removes a field, so send the whole `placement` object when you change it) |
| `remove_reference` | `id` |

| Reference field | Meaning |
|---|---|
| `id`, `name` | Optional (default `ref1`, `ref2`, …; the name defaults to the file name) |
| `image` | `{"path": "/abs/photo.jpg"}` (PNG or JPEG) or `{"dataUrl": "data:image/png;base64,..."}` (≤ 2 MB) |
| `plane` | Any sketch plane: `{"base": "XY" \| "XZ" \| "YZ", "offset"?}`, `{"face": {...}}`, explicit or a reference plane |
| `placement` | `{"origin"?: [u, v], "width": mm, "rotation"?: deg}`: the image **centre** in plane coordinates, its width in mm, counter-clockwise rotation. Height = width × pixel height / pixel width |
| `opacity` | 0–1 (default 1); 0.5–0.6 makes tracing easier |
| `visible` | Default true |
| `mode` | `"behind"` (default, never hides the model) or `"depth"` |
| `calibration` | `{"p1": [u, v], "p2": [u, v], "distance": mm}` (two-point scale) |

**Plane:** put a front photo or front drawing view on `XZ` (u = +X, v = +Z), a top-down photo
on `XY` (u = +X, v = +Y), a side view on `YZ` (u = +Y, v = +Z).

**Pixel → plane coordinates** (rotation 0; pixel (px, py) measured from the top-left of a
W × H image):
- `u = origin[0] + (px / W − 0.5) × width`
- `v = origin[1] + (0.5 − py / H) × width × H / W`

**Two-point calibration:** `p1` and `p2` are the plane coordinates of the two ends of a known
dimension *as the image is currently placed*, and `distance` is its real length. Kadvia scales
the image about `p1` so that |p2 − p1| = distance (and stores the scaled `p2`). The user can do
the same with the two-point tool in the app; if they did, read the result with `kadvia:get_part`
(`document.references`).

Tips:
- Always ask for one real dimension. Without it the calibration (and the model) is only proportional.
- Calibrate on the longest known dimension, measured on a face seen square-on.
- Perspective photos: calibrate and trace only on the plane of the face nearest square to the camera; elsewhere use the photo for proportions only.
- `report.references` in the modeling result gives each reference's resolved plane (or an `error`).

## 9. Worked example: tracing a calibrated photo
**User:** "Model this bracket from my photo ~/Pictures/bracket-front.jpg. The base is 100 mm wide."

The photo is 1600 × 1200 px, taken straight from the front. In the image, the base's bottom
edge runs from pixel (320, 980) to pixel (1320, 980).

1. `kadvia:new_part {"name": "Bracket from photo"}` → `m1`.
2. Place the photo on `XZ` with a guessed width of 200 mm (so the image height is 150 mm),
   centred at [0, 40]:

```json
{"model_id": "m1", "operations": [
  {"op": "add_reference", "reference": {"id": "photo", "name": "Front photo",
    "image": {"path": "/Users/me/Pictures/bracket-front.jpg"},
    "plane": {"base": "XZ"}, "placement": {"origin": [0, 40], "width": 200}, "opacity": 0.6}}
]}
```
3. The base ends in plane coordinates, with the formulas above:
   - left: u = 0 + (320/1600 − 0.5)·200 = **−60**, v = 40 + (0.5 − 980/1200)·150 = **−7.5**;
   - right: u = 0 + (1320/1600 − 0.5)·200 = **65**, v = −7.5.

   They are 125 mm apart, but the real width is 100 mm, so calibrate:

```json
{"model_id": "m1", "operations": [
  {"op": "update_reference", "id": "photo",
    "patch": {"calibration": {"p1": [-60, -7.5], "p2": [65, -7.5], "distance": 100}}}
]}
```
   Kadvia scales the image by 100/125 = 0.8 about p1: the width becomes 160 mm and the
   origin [−12, 30.5].
4. Optional: move the image so the base's left end sits on the world origin (shift by
   +60, +7.5), and restate the calibration at the new position so it stays consistent:

```json
{"model_id": "m1", "operations": [
  {"op": "update_reference", "id": "photo",
    "patch": {"placement": {"origin": [48, 38], "width": 160},
              "calibration": {"p1": [0, 0], "p2": [100, 0], "distance": 100}}}
]}
```
5. `kadvia:render_views {"views": ["front"], "include_references": true}`: check that the
   base's left end is at the origin and its right end at X = 100.
6. Read the other key points the same way (now 1 image px = 160/1600 = 0.1 mm), show the
   user the dimension table ([section 5](#5-confirming-with-the-user)), and after
   confirmation sketch on `XZ` with parameters, then extrude (`"direction": "symmetric"` with
   the depth the user gives, because the photo can't show it).
7. Compare again with `include_references: true` (and `opacity` 0.4 if the photo hides the
   model's edges). Adjust parameters until the silhouette matches, and report what is still
   assumed (depth, hidden features).
