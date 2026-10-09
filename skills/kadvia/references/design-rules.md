# Design rules (basic DFM)

Rules of thumb for making parts that can actually be produced. Use them as defaults when the
user doesn't specify, and say which ones you assumed. The supplier's or machine's own limits
always win.

## Contents
1. [Metric screws: holes, counterbores, nuts](#1-metric-screws-holes-counterbores-nuts)
2. [Hole placement](#2-hole-placement)
3. [3D printing](#3-3d-printing)
4. [CNC machining](#4-cnc-machining)
5. [Sheet metal](#5-sheet-metal)
6. [Injection molding (awareness)](#6-injection-molding-awareness)
7. [Fillets and chamfers](#7-fillets-and-chamfers)
8. [Densities](#8-densities)

## 1. Metric screws: holes, counterbores, nuts

| Size | Pitch | Clearance fine / **medium** / coarse (ISO 273) | Tap drill | Socket head Ø × height (ISO 4762) | Counterbore Ø × depth | Hex nut AF × thickness (ISO 4032) | Washer OD (ISO 7089) |
|---|---|---|---|---|---|---|---|
| M3 | 0.5 | 3.2 / **3.4** / 3.6 | 2.5 | 5.5 × 3 | 6.5 × 3.3 | 5.5 × 2.4 | 7 |
| M4 | 0.7 | 4.3 / **4.5** / 4.8 | 3.3 | 7 × 4 | 8 × 4.4 | 7 × 3.2 | 9 |
| M5 | 0.8 | 5.3 / **5.5** / 5.8 | 4.2 | 8.5 × 5 | 9.5 × 5.4 | 8 × 4.7 | 10 |
| M6 | 1.0 | 6.4 / **6.6** / 7.0 | 5.0 | 10 × 6 | 11 × 6.5 | 10 × 5.2 | 12 |
| M8 | 1.25 | 8.4 / **9.0** / 10.0 | 6.8 | 13 × 8 | 14 × 8.6 | 13 × 6.8 | 16 |
| M10 | 1.5 | 10.5 / **11.0** / 12.0 | 8.5 | 16 × 10 | 17.5 × 10.8 | 16 × 8.4 | 20 |
| M12 | 1.75 | 13.0 / **13.5** / 14.5 | 10.2 | 18 × 12 | 20 × 13 | 18 × 10.8 | 24 |

- **Clearance holes:** use *medium* by default, *fine* for precise location, and *coarse* for loose assembly or printed parts.
- **Tapped holes:** model the tap-drill diameter, which is what gets drilled before threading. Thread engagement is at least 1×d in steel, 1.5×d in aluminium and 2×d in plastic. Make blind tapped holes about 0.5×d deeper than the thread.
- **Counterbores:** `{"counterbore": {"diameter": 11, "depth": 6.5}}` for M6 socket heads. The depth is the head height plus about 0.5 mm.
- **Hex nut trap (pocket):** use a polygon profile with `sides: 6` and `radius: "(nut_af + 0.3)/sqrt(3)"`. The radius is measured centre to vertex, and 0.3 mm is the printing clearance. Pocket depth = nut thickness + 0.2.
- **Bolt head clearance:** keep washer OD / 2 plus about 1 mm between the bolt centre and any wall, hub or boss.

## 2. Hole placement
- Hole centre to part edge: at least 1.5×d, 2×d recommended (at least 2×t in sheet metal).
- Hole centre to hole centre: at least 2×d; at least 2.5×d for bolted joints that carry load.
- Wall left around a tapped hole: at least 0.5×d.
- Holes in thin walls: the wall should be at least about 0.5×d thick to hold a thread.

## 3. 3D printing

| Rule | FDM (0.4 mm nozzle) | SLA / resin | SLS / MJF (nylon) |
|---|---|---|---|
| Minimum wall | 0.8 mm (2 perimeters); **1.2–2 mm typical**, 2–3 mm structural | 0.6–1.0 mm | 0.7–1.0 mm |
| Minimum hole Ø | 2 mm | 0.5–1 mm | 1.5 mm |
| Holes print undersize by | 0.1–0.3 mm → add 0.2 mm or ream | ~0.1 mm | ~0.1–0.2 mm |
| Clearance between mating parts (per side) | 0.2 (snug) – 0.4 (sliding) | 0.1–0.2 | 0.3–0.5 |
| Overhangs | ≤ 45° from vertical without support; bridges ≤ ~10 mm | supports needed | self-supporting |

- **Orientation:** put the largest flat face on the bed (Z = 0 in Kadvia). Vertical holes come out rounder than horizontal ones.
- **Bottom edges:** chamfer them 0.4–0.6 mm against elephant's foot, rather than filleting.
- **Screws into plastic:**
  - Self-tapping M3: pilot hole 2.5 mm in a boss of OD 6–8 mm.
  - Heat-set insert M3: hole about 4.0 mm, but follow the insert's datasheet. Make the hole depth the insert length + 1 mm.
- **Bosses and ribs:** join them to walls (overlap 0.5–1 mm) with a small fillet. Ribs are 60–80% of wall thickness.
- **Text and emboss:** at least 0.6 mm stroke and depth.

## 4. CNC machining
- **Minimum wall:** 0.8 mm in metals (1.0 mm or more preferred) and 1.5 mm in plastics.
- **Internal vertical corners:** they always have a radius. Make it at least the cutter radius: 1 mm minimum, 3 mm typical (Ø6 cutter), and about ⅓ of the pocket depth or more. Model it explicitly with a `rect` `cornerRadius` or a fillet on `{"parallel": "Z"}` pocket edges.
- **Pocket depth:** at most 4× the pocket width (6× is possible but costly).
- **Hole depth:** at most 4×d typical, up to 10×d drilled. Use standard drill diameters.
- **Threads:** depth 1.5×d. Avoid threads smaller than M3 in aluminium.
- **External edges:** break them with a 0.2–0.5 mm chamfer. Sharp outside corners are fine.
- **Access:** keep features reachable from as few sides as possible (ideally top and bottom). Avoid undercuts.
- **Tolerances:** ±0.1 mm by default. Ask before specifying anything tighter.

## 5. Sheet metal
Kadvia v1 has no bend or flange features. Model sheet-metal parts as a solid with **uniform
thickness `t`**, for example an extruded path profile, and follow these rules so a fabricator
can unfold the part.
- **Inside bend radius:** at least 1×t for mild steel and 5052 aluminium, at least 1.5–2×t for 6061-T6 and stainless. Outside radius = inside radius + t.
- **Minimum flange length:** at least 4×t, or 3×t + inside radius.
- **Holes:** diameter at least t (and at least 1 mm). Hole edge to part edge at least 2×t. Hole edge to bend at least 2.5×t + bend radius.
- **Bend relief:** width at least t, depth at least the bend radius + t, where a bend meets an edge.
- **Common gauges:** 0.8, 1.0, 1.2, 1.5, 2.0, 2.5, 3.0 mm.

## 6. Injection molding (awareness)
- Keep walls uniform at 1.5–3 mm (ABS 1.2–3.5 mm). Make ribs 50–60% of the wall thickness.
- Inside radius should be at least 0.5×t, and outside radius = inside radius + t.
- Molded parts need draft, typically 1–2°, which Kadvia v1 cannot model. Mention it to the user for molded parts.

## 7. Fillets and chamfers

| Situation | Guideline |
|---|---|
| L-bracket inside bend | r = 0.5–1×t; outside = r + t (uniform wall) |
| CNC pocket inside corners | ≥ cutter radius, typically 3 mm; ≥ ⅓ pocket depth |
| Stress relief at inside corners | r ≥ 0.5×t, and larger for fatigue-loaded parts |
| Printed top edges | 0.5–2 mm fillet |
| Printed bottom edges | 0.4–0.6 mm chamfer (not a fillet) |
| Hand-safe edges (any process) | 0.5–1 mm fillet or chamfer |
| Hole entry (for screws and pins) | 0.3–1 mm chamfer |

- The fillet radius must be smaller than the adjacent faces. On a plate of thickness t, a top-edge fillet must be less than t.
- Prefer profile corner radii (`cornerRadius`) for vertical corners. They are more robust than filleting afterwards.

## 8. Densities
For `kadvia:mass_properties` `density_g_cm3`:

| Material | g/cm³ |
|---|---|
| Aluminium 6061 / 7075 | 2.70 / 2.81 |
| Steel (mild, 1018/S235) | 7.85 |
| Stainless 304 / 316 | 8.00 |
| Brass | 8.50 |
| Copper | 8.96 |
| Titanium Ti-6Al-4V | 4.43 |
| PLA / PETG / ABS | 1.24 / 1.27 / 1.04 |
| Nylon PA12 (SLS) | 1.01 |
| Acetal (POM) | 1.41 |
| Polycarbonate | 1.20 |

FDM parts are not solid. Real mass is roughly 50–80% of the solid value, depending on infill.
