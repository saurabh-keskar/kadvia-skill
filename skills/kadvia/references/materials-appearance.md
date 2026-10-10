# Materials, appearance and product renders

A **material** is engineering data: density (for mass), Young's modulus, Poisson's ratio, yield
and tensile strength, plus a default look. An **appearance** is display only: colour and finish
never change geometry, volume or mass. Both are stored in the part (and in the components of an
assembly), go through undo/redo and are saved with `.kadvia` / `.kasm` files. The live reference
is `kadvia:modeling_reference {"topic": "materials"}`.

## Contents
1. [Tools](#1-tools)
2. [Material library](#2-material-library)
3. [set_material](#3-set_material)
4. [set_appearance](#4-set_appearance)
5. [Finishes and typical looks](#5-finishes-and-typical-looks)
6. [Mass, BOM and drawings](#6-mass-bom-and-drawings)
7. [Product renders](#7-product-renders)
8. [Operations inside apply_operations](#8-operations-inside-apply_operations)
9. [Worked example: anodised blue aluminium product render](#9-worked-example-anodised-blue-aluminium-product-render)
10. [Limits](#10-limits)

## 1. Tools
| Tool | Use it to |
|---|---|
| `kadvia:list_materials {query?, category?}` | List the library (id, name, category, density g/cm³, modulus GPa, yield/tensile MPa, default look). `category`: `metal`, `plastic`, `other`. Changes nothing |
| `kadvia:set_material {model_id, material, body_id?, component_id?}` | Assign a material to the whole part (default of every body), one body, or override it for one assembly component; `material: null` removes it |
| `kadvia:set_appearance {model_id, body_id?, faces?, color?, finish?, metalness?, roughness?, clearcoat?, opacity?, name?, reset?, replace?}` | Change the look of the part, one body, or some faces of a body |
| `kadvia:mass_properties {model_id, body_id?, density_g_cm3?}` | Volume, area, centre of mass and **mass from the assigned materials** |
| `kadvia:render_views {views, quality: "render", environment?, background?}` | Beauty render (product shot) with materials and appearances |

Each `set_material` / `set_appearance` call is one undo step and returns the bodies with their
material and mass.

## 2. Material library
22 materials with nominal room-temperature datasheet values (unfilled grades for plastics):

| Category | Ids |
|---|---|
| Metals | `aluminium_6061` (2.70), `aluminium_7075` (2.81), `steel_1018` (7.87), `steel_4140` (7.85), `stainless_304` (8.00), `stainless_316` (8.00), `brass` (8.50), `copper` (8.89), `titanium_ti6al4v` (4.43) |
| Plastics | `abs` (1.04), `pla` (1.24), `petg` (1.27), `nylon_pa12` (1.01), `polycarbonate` (1.20), `pom` (1.41), `polypropylene` (0.905), `pvc` (1.38), `peek` (1.30) |
| Other | `wood_oak` (0.75), `rubber` (0.93), `glass` (2.50), `carbon_fibre` (1.55) |

Densities in g/cm³. Names and common aliases work too, ignoring case, punctuation and US/UK
spelling: "Aluminum 6061", "mild steel", "stainless", "Delrin", "nylon", "carbon fiber". An
unknown name is a `bad_request` that suggests close matches. When the user names a material
that isn't in the library, use a **custom material**: `{"name": "PU foam", "density": 0.05}`
(density in g/cm³, 0–30; optional `youngsModulus` GPa, `yieldStrength` MPa, `appearance`).

## 3. set_material
```
kadvia:set_material {"model_id": "m1", "material": "aluminium_6061"}
kadvia:set_material {"model_id": "m1", "body_id": "b1", "material": "rubber"}
kadvia:set_material {"model_id": "a1", "component_id": "lid", "material": "polycarbonate"}
kadvia:set_material {"model_id": "m1", "material": null}
```
- The material of a body is the body override, else the part material.
- An assembly component uses its part's materials unless the component has its own override; components with different overrides become separate BOM rows. Overrides on components inside a rigid sub-assembly are not applied when it is placed.
- The material brings its own default look (brushed aluminium, satin ABS, …) until you set an appearance.
- Set a material whenever the user names one: mass, BOM masses and drawing BOM tables then come out right without a density argument.

## 4. set_appearance
```
kadvia:set_appearance {"model_id": "m1", "color": "#2a5db0", "finish": "satin", "name": "Anodised blue"}
kadvia:set_appearance {"model_id": "m1", "body_id": "b0", "faces": {"normal": "+Z", "plane": "max_z"}, "finish": "polished"}
kadvia:set_appearance {"model_id": "m1", "reset": true}
```
| Field | Meaning |
|---|---|
| `color` | sRGB `"#rrggbb"` (also `"#rgb"`), or a basic name: black, white, grey, silver, red, orange, yellow, gold, green, teal, blue, navy, purple, pink, brown |
| `finish` | `polished`, `brushed`, `satin`, `matte`, `glossy`, `textured` (sets roughness / clear coat unless given explicitly) |
| `metalness` | 0 = paint/plastic, 1 = bare metal |
| `roughness` | 0 = mirror … 1 = fully diffuse |
| `clearcoat` | Clear lacquer layer 0–1 |
| `opacity` | 1 = opaque (default); 0.05–0.9 = see-through. 0 is refused |
| `name` | Display name of the look |
| `body_id` | Style one body (default: the whole part) |
| `faces` | Face selector for some faces of `body_id` (required with `faces`): `{"normal": "+Z"}`, `{"plane": "max_x"}`, `{"surface": "cylinder", "concave": true}` (hole walls), `{"ids": [3, 7]}` (from `kadvia:get_selection`) |
| `reset` | `true` removes the appearance at this level (the material's or part's look shows again) |
| `replace` | `true` starts from scratch instead of merging into the existing look |

- **Precedence:** neutral grey → the material's look → part appearance → body appearance → face appearances (later face entries win on shared faces).
- **Fields merge:** setting only `color` keeps the finish.
- Face selectors are re-resolved on every rebuild, so prefer `normal` / `plane` / `surface` over `ids` (ids change with the topology).

## 5. Finishes and typical looks
| finish | Roughness | Other | Good for |
|---|---|---|---|
| `polished` | 0.06 | | chrome, polished stainless, mirrors |
| `brushed` | 0.32 | directional highlights | brushed aluminium / stainless |
| `satin` | 0.45 | | anodised aluminium, moulded plastic |
| `matte` | 0.85 | | powder coat, raw 3D prints, rubber |
| `glossy` | 0.20 | clear coat 1 | car paint, gloss plastic, carbon fibre |
| `textured` | 0.75 | fine bump | bead-blasted / textured plastic |

| Look | Material + appearance |
|---|---|
| Anodised aluminium (any colour) | `aluminium_6061` + `color`, `finish: "satin"`, `metalness` 0.6–1 |
| Black powder-coated steel | `steel_1018` + `color: "#1a1a1a"`, `finish: "matte"`, `metalness: 0` |
| Brushed stainless | `stainless_304` + `finish: "brushed"` |
| Printed PETG in orange | `petg` + `color: "#ff6f00"`, `finish: "matte"` |
| Clear polycarbonate cover | `polycarbonate` + `opacity: 0.3` (tinted: 0.6) |
| Rubber feet on body b1 | `set_material` `body_id: "b1"`, `rubber` (its look is already matte black) |

Keep the real material for coated or anodised metal (mass stays right) and change only the
appearance.

## 6. Mass, BOM and drawings
- `kadvia:mass_properties` returns `mass_g` / `mass_kg` with `massSource: "assigned material(s)"` when every body has a material; several bodies give per-body rows and a mass-weighted centre of mass (`material: "mixed"` when they differ). Bodies without one are listed in `unassigned` (the centre of mass is then volume-weighted).
- `density_g_cm3` still overrides everything (`massSource: "density_g_cm3 argument"`); use it for "what would it weigh in brass?" without changing the part.
- `kadvia:assembly_bom` rows get `mass` per unit and `material` from the materials (a `density_g_cm3` argument overrides them); `totalMass` appears when every row has a mass.
- Drawing BOM tables: the `mass` column uses the materials unless `density` is set; a `material` column shows material names. The title block `material` field is plain text: fill it in (for example `"AL 6061-T6"`).
- FDM prints are not solid: say that the real mass is roughly 50–80 % of the solid value, depending on infill.

## 7. Product renders
`kadvia:render_views` with `quality: "render"` makes a beauty render: physically based
materials and appearances, environment lighting, tone mapping, a soft ground shadow, no edges,
supersampled.

| Field | Values |
|---|---|
| `quality` | `"standard"` (default: CAD look with edges, for checking geometry) or `"render"` |
| `environment` | Only with `quality: "render"`: `"studio"` (default, soft boxes), `"outdoor"` (sky and sun), `"warehouse"` (overhead strip lights) |
| `background` | `"environment"` (default for renders, soft backdrop), `"white"`, `"gradient"`, `"transparent"` (PNG with alpha, for compositing) |
| `views`, `width`, `height` | Standard views, fitted; 64–2048 px |

- Use 1–2 views at 1200–2048 px for a product shot, and the standard quality for checking geometry (edges help).
- Renders show every open model: close or mention the others first.
- Parts without a material or appearance render in a neutral grey; set at least a material for a convincing shot.
- `environment` without `quality: "render"` is refused.

## 8. Operations inside apply_operations
The tools send these operations; you can also batch them with modeling operations (one undo
step for everything):

```json
[
  {"op": "set_material", "material": "stainless_304"},
  {"op": "set_material", "body": "b1", "material": {"name": "PU foam", "density": 0.05, "appearance": {"color": "#ffcc00", "finish": "textured"}}},
  {"op": "set_appearance", "appearance": {"color": "#2a5db0", "finish": "satin", "name": "Anodised blue"}},
  {"op": "set_appearance", "body": "b0", "faces": {"normal": "+Z", "plane": "max_z"}, "appearance": {"finish": "polished"}},
  {"op": "set_appearance", "body": "b0", "faces": {"normal": "+Z", "plane": "max_z"}, "appearance": null}
]
```
- `set_appearance` takes `appearance` (fields merge; `null` fields remove; `appearance: null` removes that level), `replace: true`, and with `body`, `clearFaces: true` drops the body's face styles first. `faces` needs `body`.
- `appearance: null` with `faces` removes the face style with the identical selector.
- An unknown material, an invalid colour or range, opacity 0, an unknown body id or a face selector that matches nothing makes the whole batch fail (`bad_request` with `opIndex`).

## 9. Worked example: anodised blue aluminium product render
**User:** "Make the bracket anodised blue aluminium and give me a nice product shot on a
transparent background. How much does it weigh?"

1. `kadvia:kadvia_status` → the part `m1` is open (close other models, with the user's OK, so they don't appear in the shot).
2. `kadvia:set_material {"model_id": "m1", "material": "aluminium_6061"}` → bodies report the material and mass, for example `mass_g: 66.9`.
3. `kadvia:set_appearance {"model_id": "m1", "color": "#2a5db0", "finish": "satin", "metalness": 0.8, "name": "Anodised blue"}` (geometry and mass unchanged).
4. `kadvia:render_views {"views": ["iso"], "quality": "render", "environment": "studio", "background": "transparent", "width": 1600, "height": 1200}` (it renders the open scene). Look at the image: is the colour right, are the faces lit, is anything clipped?
5. Optional second angle: `"views": ["front"]` or `"environment": "outdoor"`.
6. Report: material Aluminium 6061-T6, mass from step 2 (or `kadvia:mass_properties`), the look you set (colour, finish), and that the PNG has a transparent background. Offer `kadvia:save_part` so the material and look are kept.

Two-tone variant: black anodised top face only.
```json
[
  {"op": "set_appearance", "body": "b0", "faces": {"normal": "+Z", "plane": "max_z"},
   "appearance": {"color": "#111111", "finish": "satin", "metalness": 0.8}}
]
```

## 10. Limits
- No image textures or UV mapping: brushed and textured finishes are procedural; there is no texture library.
- Materials are isotropic engineering placeholders: no temperature dependence and no stress analysis (FEA) yet.
- Assembly component overrides apply to the whole component; overrides inside rigid sub-assemblies are not applied when placed.
- The render's ground shadow is a shadow map (not ray traced), and the ground shows no reflections.
- Imported STEP geometry comes without a material: set one after `kadvia:convert_to_part` (STEP components of an assembly have none).
