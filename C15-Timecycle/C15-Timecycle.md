# Chapter 15 — The Timecycle

> **Goal of this chapter:** decode the file that gives San Andreas its look — 23 weathers × 8 hours ×
> 51 values — and note that it is the one data file in the game that **documents its own columns**,
> which lets this chapter name them without borrowing from anyone.

**Subsystem category:** Rendering / environment data
**Depends on:** [C14 — Peds, Weapons & Stats](../C14-Peds-And-Weapons/C14-Peds-And-Weapons.md)
**Ties:** [X1](../X1-SDK-Cross-Reference/X1-SDK-Cross-Reference.md), [C10](../C10-2dEffect/C10-2dEffect.md), [C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md), [C13](../C13-Vehicle-Data/C13-Vehicle-Data.md), [C14](../C14-Peds-And-Weapons/C14-Peds-And-Weapons.md)
**RE status:** Verified
**Confidence:** ✅ Verified over the full file

---

## Deep-dive pages

- [C15.1 — Twenty-three weathers, eight hours](01-weathers-and-hours.md): the block structure, exact
  and uniform.
- [C15.2 — The columns the file names itself](02-self-documenting-columns.md): a header comment that
  makes borrowed tables unnecessary.
- [C15.3 — The fifth defect](03-the-fifth-defect.md): two missing values in `RAINY_COUNTRYSIDE`, and
  why the obvious sanity check misses it.

---

## 15.1 The shape

```
timecyc.dat — 437 lines
├── 253 comment lines  (//)
└── 184 data rows
      = 23 weather blocks × 8 time-of-day rows
```

✅ *Verified.* Every one of the 23 blocks has **exactly 8 rows**, and all 23 carry the **identical**
time-label sequence:

```
Midnight · 5AM · 6AM · 7AM · Midday · 7PM · 8PM · 10PM
```

Eight samples per day, unevenly spaced — dense around dawn and dusk, sparse across midday and the small
hours. The engine interpolates between them.

## 15.2 The 23 weathers

| Region | Blocks | Names |
|---|---:|---|
| Los Santos | 5 | `EXTRASUNNY_LA`, `SUNNY_LA`, `EXTRASUNNY_SMOG_LA`, `SUNNY_SMOG_LA`, `CLOUDY_LA` |
| San Fierro | 5 | `SUNNY_SF`, `EXTRASUNNY_SF`, `CLOUDY_SF`, `RAINY_SF`, `FOGGY_SF` |
| Countryside | 4 | `EXTRASUNNY_`, `SUNNY_`, `CLOUDY_`, `RAINY_COUNTRYSIDE` |
| Las Venturas | 3 | `SUNNY_`, `EXTRASUNNY_`, `CLOUDY_VEGAS` |
| Desert | 3 | `EXTRASUNNY_`, `SUNNY_`, `SANDSTORM_DESERT` |
| Special | 3 | `UNDERWATER`, `EXTRACOLOURS_1`, `EXTRACOLOURS_2` |

✅ *Verified.*

The regional split is the design in one table. **Los Santos gets smog variants nobody else has; San
Fierro gets fog and rain; the desert gets a sandstorm.** Weather is not a global system with regional
tinting — each region has its own authored set, and the count differs by region.

## 15.3 A file that documents itself

`timecyc.dat` opens each block with a column-header comment:

```
//Amb  Amb_Obj  Dir  Sky top  Sky bot  SunCore  SunCorona  SunSz  SprSz  SprBght
  Shdw  LightShd  PoleShd  FarClp  FogSt  LightOnGround  LowCloudsRGB
  BottomCloudRGB  WaterRGBA  Alpha1  RGB1  Alpha2  RGB2  CloudAlpha
```

**This is the only data file in the encyclopedia that names its own columns.** Everywhere else —
`handling.cfg`'s 36 fields ([C13.1 §5](../C13-Vehicle-Data/01-handling-cfg.md)), `pedstats.dat`'s 11,
`popcycle.dat`'s 26 — column meanings were left ⏳ open rather than adopted from community tables.

Here they are **derived from the file itself**, which is a different evidence class entirely. Details
and the mapping in [C15.2](02-self-documenting-columns.md).

## 15.4 The fifth defect

| Measurement | Value |
|---|---:|
| Data rows | 184 |
| Rows at 51 fields | **183** |
| Rows at 49 fields | **1** |

The odd row is **`RAINY_COUNTRYSIDE`, 8PM** — it is missing **two values from the leading ambient
triple**, so every column after shifts left by two.

⚠️ And here is the part worth remembering: a plausible sanity check — *"the first three fields should be
a 0–255 triple"* — **passes on the defective row**, because the values that land there (`255 167 198`)
happen to be in range. **Only the field count catches it.** Full analysis in
[C15.3](03-the-fifth-defect.md).

This is the fifth defect found in retail `data/`:

| File | Defect | Count |
|---|---|---:|
| `vehicles.ide` `cars` | missing comma | 3 |
| `peds.ide` `peds` | missing comma | 1 |
| `carcols.dat` `col` | period for comma | 1 |
| `carcols.dat` `car` | dangling colour index | 1 |
| **`timecyc.dat`** | **two missing values** | **1** |

## 15.5 What remains

⏳ **Open:** the exact field-group boundaries. The header names 24 logical groups against 51 numeric
fields, so groups span multiple values (`Amb` is an RGB triple, `WaterRGBA` is four). The mapping is
reconstructable and [C15.2 §3](02-self-documenting-columns.md) proposes one, but the group→field-count
assignment was not verified against every row.

Also open: `popcycle.dat` (691 rows, 480 at 26 fields) and `water.dat` (308 rows, 301 at 29) — both
sit alongside the timecycle in the environment data and neither is decoded.

---

### Key takeaways

- **23 weather blocks × 8 time-of-day rows = 184 data rows**, and every block has exactly 8 with an
  identical label sequence.
- Time samples are **unevenly spaced** — dense at dawn and dusk, sparse at midday and midnight.
- Weather is **per-region, not global**: LA has smog variants, SF has fog and rain, the desert has a
  sandstorm, and the block counts differ by region.
- **The file names its own columns in a header comment** — the only data file in this encyclopedia that
  does, so C15.2 names them from derived evidence rather than borrowed tables.
- ⚠️ **A fifth retail defect**: `RAINY_COUNTRYSIDE` 8PM is missing two values (49 fields, not 51).
- The obvious range check **passes** on the defective row — only the field count catches it.

**Next:** [C15.1 — Twenty-three weathers, eight hours](01-weathers-and-hours.md)

## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Loaded / consumed by:** [C10](../C10-2dEffect/C10-2dEffect.md), [C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md), [C13](../C13-Vehicle-Data/C13-Vehicle-Data.md), [C14](../C14-Peds-And-Weapons/C14-Peds-And-Weapons.md), [X1](../X1-SDK-Cross-Reference/X1-SDK-Cross-Reference.md)
- **Known bugs / gotchas:** timecyc column drift between timecyc/timecycp (C48) if edited inconsistently.
- **Modding:** timecyc.dat is the lighting/atmosphere mod file; sky colours feed C38.
- **Performance:** per-frame colour interpolation; cheap.
