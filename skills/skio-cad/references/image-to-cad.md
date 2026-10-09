# Image-to-CAD

Rebuilding a part from a photo, sketch, screenshot or drawing. Claude reads the image
directly; Skio needs no special tool. The skill is in **extracting dimensions honestly** and
making every guess easy to correct.

## Contents
1. [Workflow](#1-workflow)
2. [Reading the image](#2-reading-the-image)
3. [Finding the scale](#3-finding-the-scale)
4. [Estimating dimensions](#4-estimating-dimensions)
5. [Confirming with the user](#5-confirming-with-the-user)
6. [Modeling and verifying](#6-modeling-and-verifying)
7. [Common traps](#7-common-traps)

## 1. Workflow
- [ ] Classify the image: an orthographic drawing (with or without dimensions), a hand sketch, a photo (straight-on or perspective), or a render or screenshot of another CAD model.
- [ ] List the features (base shape, holes, slots, bosses, ribs, pockets, fillets, chamfers), the symmetry and the counts.
- [ ] Choose the construction (sketch → extrude, revolve, primitives) and the orientation in Skio.
- [ ] Find the scale; estimate every key dimension with its basis.
- [ ] **Ask the user to confirm** the key dimensions (table below). Don't model guesses silently.
- [ ] Model with every dimension as a parameter, in small batches.
- [ ] Render the matching view and compare with the image; check counts and proportions.
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
  - What the photo shows "from the front" should face −Y, so it matches Skio's `front` view.
  - A drawing's front view maps to Skio `front`, its top view to `top`, and its right-side view to `right`.
- Parameterise everything you estimated. Then a correction is a single `skio:set_parameters` call.
- **Verify:**
  - `skio:render_views` with the view that matches the image (`front` for a straight-on photo or drawing view, `iso` for a three-quarter photo). Compare the silhouette, feature positions and counts.
  - Check that ratios match: if the hole looks like ¼ of the width in the image, check `hole_d / width` in the model.
  - `skio:mass_properties` if the user knows the real weight. Matching mass is a strong check on thickness.
- In the report, list assumptions, unverified items and the parameters to adjust.

## 7. Common traps
- Guessing thickness from a photo taken straight on (the thickness is invisible), so ask.
- Mirrored images (selfie camera) and drawings in first-angle projection, where the views are swapped compared with third-angle. Check the projection symbol, or ask.
- Fillets seen in silhouette look like chamfers, and the other way round. Look at the shading.
- Countersunk vs counterbored holes. A cone vs a step changes the screw.
- Assuming a hole goes through when its far side is hidden.
- Using an object as the scale when it is closer to the camera than the part, which makes the part look smaller.
