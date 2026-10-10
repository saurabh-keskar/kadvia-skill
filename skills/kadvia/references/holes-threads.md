# Holes and threads (hole wizard, thread feature)

A `hole` with a `kind` and/or a `size` is a **wizard hole**: Kadvia takes its diameters, depths
and thread from standard tables, so "M6 tapped" or "counterbore for M6" is all you need to
write. Threads are added by tapped holes or by the `thread` feature on any cylindrical face. The
live reference is `kadvia:modeling_reference {"topic": "holes"}`.

## Contents
1. [When to use what](#1-when-to-use-what)
2. [Wizard hole fields](#2-wizard-hole-fields)
3. [Kinds and the sizes they produce](#3-kinds-and-the-sizes-they-produce)
4. [Depth and end conditions](#4-depth-and-end-conditions)
5. [Thread feature](#5-thread-feature)
6. [Reading holes back, resizing, drawings](#6-reading-holes-back-resizing-drawings)
7. [Worked example: four M5 tapped holes 10 deep](#7-worked-example-four-m5-tapped-holes-10-deep)
8. [Worked example: counterbored holes for M6 socket head screws](#8-worked-example-counterbored-holes-for-m6-socket-head-screws)
9. [More snippets](#9-more-snippets)
10. [Limits](#10-limits)

## 1. When to use what
| The user asks for | Use |
|---|---|
| "M5 tapped", "threaded M8 hole", "1/4-20 hole" | `kind: "tapped"`, `size` |
| "clearance for M6", "M6 bolt hole", "through hole for a #10 screw" | `kind: "clearance"`, `size` (and `fit` if they say close/loose) |
| "counterbored for M6 socket head", "sunk cap screw" | `kind: "counterbore"`, `size` |
| "countersunk for M5 flat head" | `kind: "countersink"`, `size` |
| "Ø7 hole" (a plain number, no screw) | plain hole: `diameter` (no `kind`/`size`) |
| "thread this shaft M10", "thread the existing Ø8.5 hole" | `thread` feature on the cylindrical face |
| "make the holes M8 instead" | `update_feature {id, patch: {"size": "M8"}}` |

Prefer the wizard over typing diameters: the sizes are standard, the drawing gets proper hole
callouts and thread lines, and a size change is one patch.

## 2. Wizard hole fields
```json
{"id": "m6_holes", "type": "hole", "plane": {"face": {"normal": "+Z", "plane": "max_z"}},
 "points": [[-20, -10], [20, -10]], "kind": "tapped", "size": "M6", "depth": 15}
```
| Field | Meaning |
|---|---|
| `plane`, `points` | As for every hole: the plane on the **entry face**, drilling against its normal; `points` are `[u, v]` in the plane frame (on a top face u = +X, v = +Y). Every point gets the same hole |
| `kind` | `simple`, `clearance`, `tapped`, `counterbore`, `countersink`, `counterdrill`. Default `clearance` when only `size` is given (or the kind of the `counterbore` / `countersink` / `counterdrill` object you pass) |
| `size` | ISO metric `"M1"` … `"M64"`, fine pitch `"M8x1"`; inch `"1/4-20 UNC"`, `"#10-32 UNF"`, `"1/4 UNF"`, `"#10"`, `"1/2"` |
| `standard` | `"iso"` or `"ansi"`; default from the size (`M…` is ISO, otherwise inch) |
| `fit` | Clearance-based kinds: `"close"`, `"normal"` (default), `"loose"` |
| `depth`, `through` | `depth` = full-diameter depth (the drill point is extra); `through: true` = through all |
| `endCondition`, `upTo` | `"blind"` (default), `"through_all"`, `"up_to_face"` with `upTo` = a planar face selector |
| `threadDepth` | Tapped: thread length (default `depth − 3·pitch`) |
| `thread` | Tapped: `"cosmetic"` (default), `"modeled"` (real helical geometry) or `"none"` |
| `threadClass`, `leftHand` | Default `6H` (ISO) / `2B` (inch); `leftHand: true` for left-hand threads |
| `drillPoint` | Tip angle of blind holes, default 118°; `0` = flat bottom |
| `diameter` | Overrides the table hole diameter |
| `counterbore` `{diameter?, depth?}`, `countersink` `{diameter?, angle?}`, `counterdrill` `{diameter?, depth?, angle?}` | Override the table values. A tapped hole may add a `countersink` at its entry |

Every number may be an expression (`"depth": "thickness"`, points `[["hole_x", "hole_y"]]`).
Wizard holes work in `linear_pattern`, `circular_pattern` and `mirror` like any hole; the copies
keep their wizard data (and a mirrored modeled thread keeps its hand).

## 3. Kinds and the sizes they produce
All values mm. ISO metric coarse unless noted.

| kind | Hole Ø | Entry | Thread |
|---|---|---|---|
| `tapped` | Tap drill: M3 2.5, M4 3.3, M5 4.2, M6 5.0, M8 6.8, M10 8.5, M12 10.2 (fine pitch: D − P rounded up to 0.1; 1/4-20 UNC Ø5.105) | optional countersink (default Ø1.1·D, 90° / 82° inch) | designation, class, `threadDepth` |
| `clearance` | ISO 273 close / normal / loose: M3 3.2/3.4/3.6, M4 4.3/4.5/4.8, M5 5.3/5.5/5.8, M6 6.4/6.6/7.0, M8 8.4/9/10, M10 10.5/11/12 | – | – |
| `counterbore` | clearance | socket head cap screw (ISO 4762): M3 Ø6.5 × 3.3, M4 Ø8 × 4.4, M5 Ø9.5 × 5.4, M6 Ø11 × 6.5, M8 Ø14 × 8.6, M10 Ø17.5 × 10.8 | – |
| `countersink` | clearance | 90° (ISO 10642): M3 Ø6.72, M4 Ø8.96, M5 Ø11.2, M6 Ø13.44, M8 Ø17.92; inch flat heads 82° | – |
| `counterdrill` | clearance | Ø1.5·D, depth 0.5·D, 118° cone | – |
| `simple` | `diameter` (or the nominal Ø of `size`) | – | – |

- Inch clearance holes add a fixed allowance to the nominal size (they are not a full inch fastener table).
- An unknown size, an invalid pitch (`"M8x0.9"`) or a size without a table entry for the requested entry (for example a counterbore for M18) is a `bad_request` that names the valid choices or asks for the explicit value: give `counterbore: {"diameter": ..., "depth": ...}` then.

## 4. Depth and end conditions
- **Blind** (default): give `depth`. The drill point (118° cone) is added below it, so the hole is deeper at the centre by `r / tan(59°)` ≈ 0.6·r. Use `"drillPoint": 0` for a flat bottom.
- **Tapped blind without `depth`:** the thread is `threadDepth` (default 2·D) and the drill goes `threadDepth + 3·pitch`, rounded up to 0.5 mm. So `"threadDepth": 10` on M5 drills 12.5 mm.
- **Through:** `through: true` (or `endCondition: "through_all"`).
- **Up to a face:** `"endCondition": "up_to_face", "upTo": {"normal": "-Z", "plane": "min_z"}`; the depth is measured at each location to that face's plane along the hole axis (the face must not be parallel to the axis).
- "M5 × 10 deep" almost always means **10 mm of full thread**: write `threadDepth: 10` and let the drill depth follow, or ask if the part is thin. Check that the drill depth plus the point fits the material (blind holes must not break through by accident).

## 5. Thread feature
Adds a thread to an existing cylindrical face: a hole wall (internal thread) or a shaft or boss
(external thread).

```json
{"id": "shaft_thread", "type": "thread",
 "faces": {"surface": "cylinder", "concave": false, "diameter": 10}, "length": 20}
```
| Field | Meaning |
|---|---|
| `faces` | Face selector for the cylindrical faces of **one** cylinder (split walls are joined) |
| `size`, `standard` | Default: the coarse/UNC size that fits (holes by tap drill, shafts by major Ø within about 8 %) |
| `length`, `offset` | Thread length (default the whole face) and distance from the start end |
| `reverse` | Start at the other end. Default start: the open end (top of a hole, free end of a shaft; with two open ends, the upper one) |
| `threadClass` | Default 6H / 6g (ISO), 2B / 2A (inch) |
| `modeled` | `false` (default) = cosmetic; `true` = cut a real 60° helical groove |
| `leftHand`, `target` | Left-hand thread; body id |

- Internal threads need the hole smaller than the major diameter (a tap-drill hole); external threads need a shaft between the minor and about 1.05× the major diameter.
- **Cosmetic** threads (default) leave the geometry alone: the viewer shows a thread texture, drawings draw thread lines and callouts, `recognize_features` reports them. Use them for machined parts.
- **Modeled** threads cut real geometry. They take seconds per thread (at most 400 turns): use them only for 3D printing or close-up renders.

## 6. Reading holes back, resizing, drawings
- `kadvia:recognize_features` on a part lists wizard holes under `holeSizes` by size and kind (for example `{"M5 tapped (m5_holes)": 4}`). Each hole carries `wizard` (`feature`, `kind`, `standard`, `size`, `fit`, `role`: `drill` / `counterbore` / `counterdrill`, `callout`) and `thread` (`"M5x0.8 - 6H"`); each body lists its `threads`.
- **Resize** by patching the size; all diameters follow: `{"op": "update_feature", "id": "m5_holes", "patch": {"size": "M6"}}`. Don't use `resize_hole` on a wizard hole (that is for imported geometry).
- **Drawings:** auto dimension adds one hole callout per wizard hole feature and view instead of diameter dimensions. A `hole_callout` on any circle of a wizard hole shows the standard callout with the count of equal holes, for example `4X Ø6.6 THRU` / `⌴Ø11 ↧6.5`, `Ø4.2 ↧12.5` / `M5x0.8 - 6H ↧10`, or `Ø5.5 THRU` / `⌵Ø11.2 X 90°`. Cosmetic threads are drawn as thin thread lines (hidden-dashed for internal threads, a ¾ circle in end views). See [drawings.md](drawings.md#10-notes-callouts-title-block).
- `kadvia:check_design` recognises standard sizes (rule `hole_standard_size`).

## 7. Worked example: four M5 tapped holes 10 deep
**User:** "Add four M5 tapped holes, 10 deep, 8 mm in from the corners of the top face."
The part is a 60 × 40 × 15 block centred on the origin with its base on Z = 0 (parameters
`width` 60, `depth` 40, `height` 15).

1. Plan: "10 deep" = 10 mm of full thread, so `threadDepth: 10`; the drill goes 10 + 3 × 0.8 =
   12.4 → **12.5 mm**, plus a 118° point (≈ 1.3 mm). 13.8 mm < 15 mm, so the holes stay blind.
   Edge distance 8 mm ≥ 1.5 × 4.2. Tell the user these assumptions.
2. Batch:
```json
[
  {"op": "set_parameter", "name": "hole_inset", "value": 8, "description": "Hole centre to the X and Y edges"},
  {"op": "set_parameter", "name": "thread_len", "value": 10, "description": "Full thread length of the M5 holes"},
  {"op": "add_feature", "feature": {"id": "m5_holes", "type": "hole",
    "plane": {"face": {"normal": "+Z", "plane": "max_z"}},
    "kind": "tapped", "size": "M5", "threadDepth": "thread_len",
    "points": [["-(width/2 - hole_inset)", "-(depth/2 - hole_inset)"], ["width/2 - hole_inset", "-(depth/2 - hole_inset)"],
               ["width/2 - hole_inset", "depth/2 - hole_inset"], ["-(width/2 - hole_inset)", "depth/2 - hole_inset"]]}}
]
```
3. Verify:
   - feature `m5_holes` is `ok`; the bbox is unchanged;
   - volume dropped by about 4 × (π × 2.1² × 12.5 + the cone) ≈ **716 mm³**;
   - `kadvia:recognize_features` → `{"M5 tapped (m5_holes)": 4}`, `thread` `"M5x0.8 - 6H"`;
   - `kadvia:render_views {"views": ["top", "iso"]}`: four holes, symmetric; the threaded walls show the thread texture in the user's window.
4. Report: "4 × M5 tapped, 10 mm full thread, Ø4.2 drill 12.5 deep. Change `thread_len` or `hole_inset`, or patch `size` to M6."

## 8. Worked example: counterbored holes for M6 socket head screws
**User:** "100 × 70 × 12 mm plate with four counterbored holes for M6 socket head screws, 12 mm
from the edges."

```json
[
  {"op": "set_parameter", "name": "width", "value": 100, "description": "Plate length (X)"},
  {"op": "set_parameter", "name": "depth", "value": 70, "description": "Plate width (Y)"},
  {"op": "set_parameter", "name": "thickness", "value": 12},
  {"op": "set_parameter", "name": "hole_inset", "value": 12, "description": "Hole centre to the edges"},
  {"op": "add_feature", "feature": {"id": "plate_sk", "type": "sketch", "plane": {"base": "XY"},
    "profiles": [{"kind": "rect", "center": [0, 0], "width": "width", "height": "depth"}]}},
  {"op": "add_feature", "feature": {"id": "plate", "type": "extrude", "sketch": "plate_sk", "distance": "thickness"}},
  {"op": "add_feature", "feature": {"id": "cb_holes", "type": "hole",
    "plane": {"face": {"normal": "+Z", "plane": "max_z"}},
    "kind": "counterbore", "size": "M6", "through": true,
    "points": [["-(width/2 - hole_inset)", "-(depth/2 - hole_inset)"], ["width/2 - hole_inset", "-(depth/2 - hole_inset)"],
               ["width/2 - hole_inset", "depth/2 - hole_inset"], ["-(width/2 - hole_inset)", "depth/2 - hole_inset"]]}}
]
```
- The table gives Ø6.6 through (normal fit) with a ⌴Ø11 × 6.5 counterbore: the screw head sits 0.5 mm below the surface.
- Verify: plate volume 84 000 mm³ minus 4 × π × (3.3² × 5.5 + 5.5² × 6.5) ≈ 3 223 mm³ → about **80 777 mm³**; `kadvia:recognize_features` lists 4 holes with `role` `drill` and `counterbore`; `kadvia:render_views {"views": ["iso"], "section": {"axis": "y", "offset": 23}}` shows the stepped section.
- A drawing of it gets the callout `4X Ø6.6 THRU` / `⌴Ø11 ↧6.5` automatically.
- After counterbores exist, `{"normal": "+Z"}` alone also matches the counterbore floors: keep `"plane": "max_z"` in face planes and selectors.

## 9. More snippets
Close-fit inch countersunk holes for #10 flat heads:
```json
{"id": "csk", "type": "hole", "plane": {"face": {"normal": "+Z", "plane": "max_z"}},
 "kind": "countersink", "size": "#10", "fit": "close", "through": true, "points": [[0, 0]]}
```
A modeled (real geometry) M8 thread for a 3D-printed part, 12 mm of thread in a 16 mm drill:
```json
{"id": "m8", "type": "hole", "plane": {"face": {"normal": "+Z", "plane": "max_z"}}, "kind": "tapped",
 "size": "M8", "depth": 16, "threadDepth": 12, "thread": "modeled", "points": [[0, 0]]}
```
A tapped hole that ends on the bottom face, with a 90° entry countersink:
```json
{"id": "m6_thru", "type": "hole", "plane": {"face": {"normal": "+Z", "plane": "max_z"}}, "kind": "tapped",
 "size": "M6", "endCondition": "up_to_face", "upTo": {"normal": "-Z", "plane": "min_z"},
 "countersink": {"angle": 90}, "points": [[0, 0]]}
```
External M10 thread on the first 20 mm of a Ø10 shaft:
```json
{"id": "m10_thread", "type": "thread", "faces": {"surface": "cylinder", "concave": false, "diameter": 10},
 "size": "M10", "length": 20}
```

## 10. Limits
- No NPT/BSP pipe threads, tapered threads or multi-start threads; thread tolerances are the class label only.
- Inch clearance holes use fixed allowances, not a full inch fastener table.
- `up_to_face` in a rotated pattern uses the face plane found for the original location.
- Modeled threads may fail on very fine pitches relative to the part, where they cross other features (cross holes, countersinks) or on chamfered entries; the error says so. Use a cosmetic thread there.
- Assembly and STEP drawings show no wizard callouts or thread lines (part drawings do).
