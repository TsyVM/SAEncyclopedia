# Chapter 14 — Peds, Weapons & Stats

> **Goal of this chapter:** decode the character and weapon side of the data model — and find the same
> comma defect that broke three vehicle rows in C13 sitting in the ped table too, plus a data file that
> dispatches on a **pound sign**.

**Subsystem category:** Gameplay / data model
**Depends on:** [C11 — IDE & IPL](../C11-IDE-And-IPL/C11-IDE-And-IPL.md),
[C13 — Handling & Vehicle Data](../C13-Vehicle-Data/C13-Vehicle-Data.md)
**Ties:** [C3](../C3-Model-Stores/C3-Model-Stores.md), [C6](../C6-Collision/C6-Collision.md), [C7](../C7-RenderWare-Stream/C7-RenderWare-Stream.md), [C10](../C10-2dEffect/C10-2dEffect.md), [C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md), [C13](../C13-Vehicle-Data/C13-Vehicle-Data.md), [C16](../C16-Popcycle-And-Water/C16-Popcycle-And-Water.md), [C25](../C25-Object-Physics/C25-Object-Physics.md)
**RE status:** Verified
**Confidence:** ✅ Verified over the full data set

---

## Deep-dive pages

- [C14.1 — The ped table](01-the-ped-table.md): 276 rows, six ped types, 270 voices, and one more
  missing comma.
- [C14.2 — weapon.dat and the £ sigil](02-weapon-dat.md): a fifth separator convention, and the only
  non-ASCII byte in the data set.
- [C14.3 — Stats and cross-checks](03-stats-and-crosschecks.md): what resolves, what is orphaned, and
  what the orphans mean.
- [C14.4 — The `$` gun-record columns](04-the-gun-record-columns.md): the column legend the data
  actually obeys, a corrected field-width count, the `PISTOL` skill-tier exception, two shapes of
  disabled weapon, and a `gta_sa.exe` cross-check that independently confirms which weapons were cut.

---

## 14.1 The inventory

| Source | Records | Structure |
|---|---:|---|
| `peds` IDE section | **276** | 14 fields (one at 13 — §14.3) |
| `weap` IDE section | **50** | 7 fields, all rows |
| `pedstats.dat` | 43 | 11 fields, all rows |
| `weapon.dat` | 91 | **sigil-dispatched** — `$` 53 (25/29-token, [C14.4](04-the-gun-record-columns.md)), `%` 21, `£` 17 |
| `melee.dat` | 153 | name/data pairs |
| `ar_stats.dat` | 59 | 3 fields, all rows |
| `statdisp.dat` | 122 | 5 fields, all rows |
| `popcycle.dat` | **480** | 24 fields, all rows — ⚠️ corrected, see below |
| `timecyc.dat` | 437 | 183 rows at 51 fields |
| `water.dat` | 307 | 301 quads at 29, 6 triangles at 22 ([C16.1](../C16-Popcycle-And-Water/01-water-dat.md)) |

✅ All counts verified over the full files.

> ⚠️ **Corrected by [C16.2 §5](../C16-Popcycle-And-Water/02-popcycle-dat.md).** This table originally
> recorded `popcycle.dat` as **691 rows at 26 fields**. Both were wrong: the census stripped `#`
> comments but not `//`, so 211 comment lines were counted as data and an inline `//` comment present on
> every row added two phantom fields. The true figures are **480 rows, 24 fields**.
> **Comment style is a per-file property** — `popcycle.dat` and `timecyc.dat` use `//`, most of `data/`
> uses `#`.

**Ped IDs run 0–299; weapon IDs run 321–373.** Both sit inside the 0–19999 model range that
[C3.1](../C3-Model-Stores/01-the-partition.md) derived from a `cmp` instruction and
[C11.1 §3](../C11-IDE-And-IPL/01-ide-definitions.md) confirmed from the object side — so peds and
weapons share the model ID space with world objects, occupying the low band that
[C11.1 §3](../C11-IDE-And-IPL/01-ide-definitions.md) noted as reserved.

## 14.2 The cross-checks

| Check | Result |
|---|---|
| `peds` `STAT_*` references resolving in `pedstats.dat` | **276 / 276** ✅ |
| `pedstats.dat` entries referenced by some ped | 34 / 43 — **9 orphaned** |
| Distinct `PED_TYPE_*` values | **6** |
| Distinct `VOICE_*` identifiers | **270** across 276 peds |

Nine `pedstats.dat` entries are defined and never used: `STAT_BUSKER`, `STAT_GANG9`, `STAT_GANG10`,
`STAT_OLDSHOPPER`, `STAT_PSYCHO`, `STAT_SENSIBLE_GIRL`, `STAT_SPORTSFAN`, `STAT_STEWARD` and one more.

🟡 *Reasoned:* the same residue as the six unused handling entries in
[C13 §13.3](../C13-Vehicle-Data/C13-Vehicle-Data.md) and the unplaced object definitions in
[C11.3 §3](../C11-IDE-And-IPL/03-what-placements-prove.md) — content cut late, definitions left behind.
`STAT_GANG9` and `STAT_GANG10` are particularly telling: the gang ladder was built with more rungs than
shipped.

**270 distinct voices for 276 peds** is near-total uniqueness — almost every character has its own
voice identifier, which is a striking amount of authored audio metadata for a 2004 title.

## 14.3 ⚠️ The comma defect, again

[C13.2](../C13-Vehicle-Data/02-cars-section-repaired.md) found three `cars` rows missing a comma. The
`peds` section has **one**, and it is the same failure:

```
normal:  12, BFYRI, BFYRI, CIVFEMALE, STAT_COWARD, sexywoman, 120C,1, null,7,9,PED_TYPE_GEN,VOICE_GEN_BFYRI ,VOICE_GEN_BFYRI
row 51:  51, BMYMOUN, BMYMOUN, CIVMALE, STAT_SENSIBLE_GUY, man,0800,1, man,2,0 PED_TYPE_GEN,VOICE_GEN_BMYMOUN ,VOICE_GEN_BMYMOUN
                                                                            ^^^ space, not comma
```

✅ Verified. **275 of 276 rows have 14 fields; row 51 has 13**, because a space replaced the comma
between `0` and `PED_TYPE_GEN`.

Four such defects now, across two files, all of the same shape — a separator lost during hand editing.
Running total in retail `data/`:

| File | Defect | Count |
|---|---|---:|
| `vehicles.ide` `cars` | missing comma | 3 |
| `peds.ide` `peds` | missing comma | **1** |
| `carcols.dat` `col` | period for comma | 1 |
| `carcols.dat` `car` | dangling colour index | 1 |

## 14.4 A fifth separator convention

[C13.3 §6](../C13-Vehicle-Data/03-carcols-and-carmods.md) tabulated three conventions across the data
files. `weapon.dat` adds a fourth mechanism and the only non-ASCII byte found anywhere in `data/`:

| File | Separator | Dispatch |
|---|---|---|
| `.ide` / `.ipl` / `carcols` / `carmods` | comma | section keyword + `end` |
| `handling.cfg` | whitespace | leading sigil `% ! $ ^` |
| **`weapon.dat`** | **whitespace** | **leading sigil `$ % £`** |

**`£` — byte `0xA3` — is the only byte above 127 in `weapon.dat`**, and it selects the melee/unarmed
record type. It occurs 19 times: 17 live sigils and 2 in comments, one of them a commented-out
`SKATEBOARD` weapon. Full analysis in [C14.2](02-weapon-dat.md).

## 14.5 What remains

⏳ **Open:** the column meanings in `weapon.dat` (26/30/10/12 by sigil),
`popcycle.dat` (26), `timecyc.dat` (51) and `water.dat` (29). This chapter verifies **structure and
cross-references**; naming columns means either correlating against code or adopting community tables,
and per [C10.1 §4](../C10-2dEffect/01-the-record-and-corrections.md) this encyclopedia does not do the
latter.

`melee.dat`'s block structure (`START_LEVELS`-style keywords with 2- and 8-field rows) is also not
decoded.

---

### Key takeaways

- **276 peds and 50 weapons**, IDs 0–299 and 321–373 — both inside the model ID range C3.1 derived from
  code.
- Cross-checks: **276/276 `STAT_*` references resolve**; **9 of 43 `pedstats` entries are orphaned**,
  including `STAT_GANG9` and `STAT_GANG10`.
- **270 distinct voice identifiers for 276 peds** — near-total uniqueness.
- ⚠️ **A fourth comma defect**: `peds.ide` row 51 (`BMYMOUN`) has a space where a comma belongs, giving
  13 fields instead of 14.
- **`weapon.dat` dispatches on `$`, `%` and `£`** — `£` (`0xA3`) is the only non-ASCII byte in the file,
  and one of its 19 occurrences marks a commented-out `SKATEBOARD` weapon.
- ⏳ Column meanings across six `.dat` files are deliberately **not** adopted from community tables.

**Next:** [C14.1 — The ped table](01-the-ped-table.md)

## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Loaded / consumed by:** [C3](../C3-Model-Stores/C3-Model-Stores.md), [C6](../C6-Collision/C6-Collision.md), [C7](../C7-RenderWare-Stream/C7-RenderWare-Stream.md), [C10](../C10-2dEffect/C10-2dEffect.md), [C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md), [C13](../C13-Vehicle-Data/C13-Vehicle-Data.md)
- **Known bugs / gotchas:** weapon.dat column miscount (C14.4 fixed 26/30->25/29); melee 52/52 open.
- **Modding:** weapon.dat is the weapon-balance file; the $ gun record is the target.
- **Performance:** data only.
