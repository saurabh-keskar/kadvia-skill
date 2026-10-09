# Example: inspect a STEP file

**User:** "Open ~/Downloads/bracket.step in Kadvia. How big is it and roughly how much would it weigh in aluminium?"

**Claude:**
1. `kadvia:kadvia_status` → app running, window ready, nothing open.
2. `kadvia:open_step_file { "path": "~/Downloads/bracket.step" }` → model `m1`, 1 body, 117 faces,
   bbox size 152.4 × 101.6 × 25.4 mm, volume 118,403 mm³, no warnings.
3. `kadvia:render_views { "views": ["iso", "top", "front"] }` → checks the part looks complete
   (no missing faces) and notes features: two slots, four mounting holes, a central boss.
4. Answer:

> The bracket is **152.4 × 101.6 × 25.4 mm** (6 × 4 × 1 in). Its volume is **118,403 mm³**,
> so in aluminium 6061 (2.70 g/cm³) it weighs about **118.4 cm³ × 2.70 g/cm³ ≈ 320 g**.
> From the renders: four corner mounting holes, two slots and a hexagonal boss in the middle.

Why this works: numbers come from tool results (with units), the shape description comes
from looking at the renders, and the mass formula is shown.
