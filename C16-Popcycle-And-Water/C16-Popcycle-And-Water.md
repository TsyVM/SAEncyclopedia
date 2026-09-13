# Chapter 16 — popcycle & water

> **Goal of this chapter:** close the last two undecoded environment files — resolving C14.3's open
> question about `water.dat`'s width split, and finding that `popcycle.dat` **states a rule the shipped
> data breaks in 8.5 % of its rows.**

**Subsystem category:** World / environment data
**Depends on:** [C14 — Peds, Weapons & Stats](../C14-Peds-And-Weapons/C14-Peds-And-Weapons.md),
[C15 — The Timecycle](../C15-Timecycle/C15-Timecycle.md)
**Ties:** [C5](../C5-CWorld/C5-CWorld.md), [C8](../C8-Geometry/C8-Geometry.md), [C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md), [C12](../C12-Path-Network/C12-Path-Network.md), [C13](../C13-Vehicle-Data/C13-Vehicle-Data.md), [C14](../C14-Peds-And-Weapons/C14-Peds-And-Weapons.md), [C15](../C15-Timecycle/C15-Timecycle.md)
**RE status:** Verified
**Confidence:** ✅ Verified over both full files
**Closes:** [C14.3 §5](../C14-Peds-And-Weapons/03-stats-and-crosschecks.md)
**Corrects:** [C14 §14.1](../C14-Peds-And-Weapons/C14-Peds-And-Weapons.md)

---

## Deep-dive pages

- [C16.1 — water.dat: quads and triangles](01-water-dat.md): the width split is design, and the water
  plane touches the world boundary exactly.
- [C16.2 — popcycle.dat and its broken invariant](02-popcycle-dat.md): a file that documents its own
  rule, and 41 rows that violate it.
- [C16.3 — How popcycle.dat drives the runtime population](03-popcycle-runtime-integration.md): the zone+time-of-day lookup chain; group label to ped model resolution via pedgrp.dat; the day/night transition hysteresis; water plane runtime role in CWaterLevel; world boundary correspondence; modding zone populations at specific times.

---

## 16.1 Two questions, both answered

[C14.3 §5](../C14-Peds-And-Weapons/03-stats-and-crosschecks.md) left both files open and flagged
`water.dat`'s **301/6 width split** as "either design or defect — worth checking which before
assuming."

It is **design**:

```
29 fields = 4 vertices × 7 values + 1 flag   → a quad
22 fields = 3 vertices × 7 values + 1 flag   → a triangle
```

✅ Verified: **301 quads + 6 triangles = 1,222 vertices**, matching `301×4 + 6×3` exactly.

The lesson C14.3 gestured at holds — a two-width split is not automatically a defect. Here it is
polygon arity, and the arithmetic settles it in one line.

## 16.2 The water plane touches the boundary exactly

| Axis | Range |
|---|---|
| X | **−3000.0 … 3000.0** |
| Y | **−3000.0 … 3000.0** |
| Z | −5.0 … 1082.7 |
| Vertices outside the grid | **0 of 1,222** |

✅ *Verified.*

This is the **fourth** independent confirmation of the world extent derived in
[C5.2](../C5-CWorld/02-sector-index-arithmetic.md) from three floating-point instructions — and the
first that lands on the boundary *exactly* rather than approaching it:

| Source | Records | Extreme |
|---|---:|---|
| [C11.3](../C11-IDE-And-IPL/03-what-placements-prove.md) — object placements | 36,569 | −2993.8 (6.2 short) |
| [C12](../C12-Path-Network/C12-Path-Network.md) — path nodes | 68,237 | −2992.0 (8.0 short) |
| **C16 — water vertices** | **1,222** | **−3000.0 (exact)** |

Objects and paths approach the limit; **the water plane *is* the limit.** That is what you would expect
of a surface authored to fill the world rather than placed within it, and it means the ±3000 constant
is not merely a bound the content respects — it is a value the content was built from.

## 16.3 popcycle documents itself, then breaks its own rule

Like `timecyc.dat` ([C15.2](../C15-Timecycle/02-self-documenting-columns.md)), `popcycle.dat` explains
itself in comments — including an explicit invariant:

> `// This number should add up to 100%. If it doesn't the game will scale it to 100% and print a warning`

Testing that claim against all 480 rows:

| Tail sum | Rows |
|---:|---:|
| **100** | **439** |
| 115 | 12 |
| 150 | 8 |
| 95 | 7 |
| 80 | 4 |
| 90 | 4 |
| 120 | 3 |
| 105 | 2 |
| 180 | 1 |

✅ **41 of 480 rows (8.5 %) violate the invariant the file itself states.**

The file even documents the consequence — the game rescales and warns — so this is not silent
corruption. But it does mean **retail San Andreas ships data that trips its own diagnostic**, and any
tool validating this file against its documented rule will report 41 failures on unmodified content.

Full analysis in [C16.2](02-popcycle-dat.md).

## 16.4 ⚠️ Correcting C14's inventory

[C14 §14.1](../C14-Peds-And-Weapons/C14-Peds-And-Weapons.md) listed:

| File | C14 said | Actual |
|---|---|---|
| `popcycle.dat` | 691 rows, 480 at **26** fields | **480 data rows, all at 24 fields** |

Two errors, one cause. C14's quick census stripped `#` comments but **not `//` comments**, so:

- the 691 "rows" were 480 data + 211 comment lines;
- the 26 "fields" counted an **inline `//` comment** present on *every* data row — the time-of-day
  label — as two extra tokens.

✅ Corrected: **480 rows, 24 fields, 100 % of rows carrying an inline comment.**

The general point: `popcycle.dat` uses `//` where most `data/` files use `#`
([C15.1](../C15-Timecycle/01-weathers-and-hours.md) is the other `//` user), and a census that assumes
one comment convention will mis-measure the other. **Comment style is per-file, not per-directory.**

## 16.5 The structure

```
480 rows = 20 zone types × 2 (Weekday / Weekend) × 12 two-hour slots
```

✅ Verified by counting banners: **20 `Weekday` and 20 `Weekend`** markers, and the file's own header
states *"a set of values for each time of the day in 2 hour increments"* — 24 h ÷ 2 = 12.

---

### Key takeaways

- ✅ **`water.dat`'s 301/6 split is design, not defect**: 29 fields = 4 vertices × 7 + flag (quad),
  22 = 3 × 7 + 1 (triangle). **1,222 vertices**, matching exactly.
- **The water plane spans exactly −3000.0 … 3000.0** — the **fourth** confirmation of the world grid,
  and the first to hit the boundary precisely rather than approach it.
- `popcycle.dat` **states an invariant in its own comments** and **41 of 480 rows (8.5 %) break it** —
  the shipped game trips its own documented warning.
- ⚠️ **C14's inventory was wrong on both counts** for this file — 480 rows not 691, 24 fields not 26 —
  because the census stripped `#` comments but not `//`.
- **Comment style is per-file**: `popcycle.dat` and `timecyc.dat` use `//`, most others use `#`.
- Structure: **20 zones × 2 day-types × 12 two-hour slots = 480**.

**Next:** [C16.1 — water.dat: quads and triangles](01-water-dat.md)

## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Loaded / consumed by:** [C5](../C5-CWorld/C5-CWorld.md), [C8](../C8-Geometry/C8-Geometry.md), [C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md), [C12](../C12-Path-Network/C12-Path-Network.md), [C13](../C13-Vehicle-Data/C13-Vehicle-Data.md), [C14](../C14-Peds-And-Weapons/C14-Peds-And-Weapons.md)
- **Known bugs / gotchas:** water quad seams if water.dat/water1.dat disagree (C48).
- **Modding:** water.dat + popcycle are edited for map/ambience mods.
- **Performance:** water render is a sibling pass; popcycle sampled periodically.
