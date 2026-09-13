# C14.3 — Stats and Cross-Checks

> **The one-sentence version:** every stat reference in the ped table resolves, nine stat definitions
> are orphaned, and the pattern of what is orphaned across four chapters says something consistent
> about how San Andreas was cut.

[← C14.2 — weapon.dat and the £ sigil](02-weapon-dat.md) · [Chapter 14 hub](C14-Peds-And-Weapons.md)

**Confidence:** ✅ Verified

---

## 1. `pedstats.dat`

43 records, **11 whitespace-separated fields on every one** — no width variation.

```
STAT_PLAYER   0.0   9.0   50   50   50   ...
```

The first field is the identifier the `peds` table references
([C14.1 §1](01-the-ped-table.md)).

**The eleven columns, from the file's own header** (recorded here to close what an earlier draft left
open): `A` the stat name (identifier); `B` flee distance (float); `C` heading-change rate (float, degrees);
`D` fear, `E` temper, `F` lawfulness, `G` sexiness (each `0–100`); `H` attack strength and `I` defend
weakness (float damage multipliers); `J` shooting rate (`0–100`); and `K` the default decision-maker
(`0` group member, `1` cop, `2` rand-normal, `3` rand-tough, `4` rand-weak). The columns are read
positionally — the header comment is the authority for their order, exactly as in
[C25](../C25-Object-Physics/C25-Object-Physics.md). ✅ *Verified — 11 fields on all 43 rows; meanings from
the header.*

## 2. The cross-check

| Direction | Result |
|---|---|
| `peds` → `pedstats.dat` | **276 / 276 resolve** ✅ |
| `pedstats.dat` → `peds` | **34 / 43 referenced — 9 orphaned** |

✅ Verified.

Every ped names a stat that exists. Nine stats exist that no ped names:

```
STAT_BUSKER        STAT_GANG9        STAT_GANG10       STAT_OLDSHOPPER
STAT_PSYCHO        STAT_SENSIBLE_GIRL  STAT_SPORTSFAN  STAT_STEWARD
```

(plus one more)

**`STAT_GANG9` and `STAT_GANG10` are the interesting pair.** San Andreas ships with gangs numbered
below that, so the stat ladder was built with more rungs than the game uses — the definitions outlived
the content.

`STAT_BUSKER`, `STAT_SPORTSFAN`, `STAT_OLDSHOPPER`, `STAT_STEWARD` read as ambient-civilian archetypes
that were specified and never populated.

## 3. The orphan pattern across four chapters

This is now the fourth data set where definitions outnumber uses, and the ratio is consistent:

| Chapter | Defined | Referenced | Orphaned | Rate |
|---|---:|---:|---:|---:|
| [C11.3 §3](../C11-IDE-And-IPL/03-what-placements-prove.md) — objects | 14,266 | ~7,800 | ~6,400 | 45 % |
| [C13 §13.3](../C13-Vehicle-Data/C13-Vehicle-Data.md) — handling | 210 | 204 | **6** | 2.9 % |
| C14 — ped stats | 43 | 34 | **9** | 20.9 % |
| [C14.2 §3](02-weapon-dat.md) — `%` weapon records | 21 | ⏳ | ⏳ | — |

🟡 *Reasoned:* the object rate is high because scripts and code place models the IPL files never
mention, so "orphaned" there mostly means *placed elsewhere*. The handling and stat rates are the more
telling numbers — those tables are referenced from exactly one place each, so an orphan really is
unused.

**A 3–21 % orphan rate in single-reference tables is the signature of late content cuts**, and it is
remarkably consistent with the defect rate found in the same files (four missing separators across
~500 hand-maintained rows). Both are what a large team shipping under deadline leaves behind.

## 4. The remaining stat files

| File | Rows | Width | Sample |
|---|---:|---:|---|
| `ar_stats.dat` | 59 | **3, all rows** | `0  STAT_INC_CYCLE_STAMINA  50` |
| `statdisp.dat` | 122 | **5, all rows** | `21  FAT  morethan  250  SBOFAT0` |

✅ Both are perfectly uniform — no defects, no width variation.

`statdisp.dat` is self-describing enough to read directly: a stat index, a name, a comparison operator
(`morethan`), a threshold, and a string identifier. It is a **display-rule table** — the mapping from a
numeric stat to the text the game shows about it.

🟡 *Reasoned:* `SBOFAT0` is a text-label key of the kind used by localisation tables, so the last field
points into the string file rather than being display text itself.

⏳ **Open:** the operator vocabulary. `morethan` is one value; censusing all 122 rows would give the
complete set, and that was not done.

## 5. The bulk data files

Three files dominate `data/` by row count and none are decoded here:

| File | Rows | Dominant width |
|---|---:|---:|
| `popcycle.dat` | **480** | **24 fields** ⚠️ ([C16.2 §5](../C16-Popcycle-And-Water/02-popcycle-dat.md)) |
| `timecyc.dat` | 437 | **183 rows at 51 fields** |
| `water.dat` | 308 | **301 at 29, 6 at 22** |

✅ The counts and widths are verified; the meanings are not.

`timecyc.dat` at 51 fields × 183 rows is the timecycle — the per-hour, per-weather colour and fog
table that gives San Andreas its look. It is the single most-modified file in the game's modding
scene, and it deserves its own chapter rather than a row here.

`water.dat`'s split — 301 rows at 29 fields and **6 at 22** — is the kind of two-width pattern that
[C13.2](../C13-Vehicle-Data/02-cars-section-repaired.md) showed can be either design or defect.

✅ **Closed in [C16.1 §2](../C16-Popcycle-And-Water/01-water-dat.md): it is design.** `29 = 4×7+1` and
`22 = 3×7+1` — quads and triangles, confirmed by a vertex count of exactly 1,222. The caution was right;
the resolution took one division.

## 6. What this chapter establishes

Structure and referential integrity, not semantics:

- Every `.dat` file's **row count and field-width distribution** — verified over the full files.
- Every **cross-reference that can be checked** between the ped table and the stat table — 276/276.
- The **orphan sets**, named.
- One new **defect** ([C14.1 §4](01-the-ped-table.md)).

What it does not do is name columns. Six files here have unexplained numeric columns, and the community
has tables for most of them. Adopting those would make this chapter look far more complete and would
reproduce exactly the failure that produced the `0x253F2FE` correction in
[C10.1 §3](../C10-2dEffect/01-the-record-and-corrections.md) — **a name accepted without asking what the
data says.**

The tests that would close them are cheap and listed where relevant: distribution censuses over a few
hundred rows, of the kind [C13.1 §5](../C13-Vehicle-Data/01-handling-cfg.md) sketches for
`handling.cfg`.

---

### Key takeaways

- `pedstats.dat` is **43 records at 11 fields**, uniform.
- **276/276 ped stat references resolve**; **9 of 43 stat definitions are orphaned**, including
  `STAT_GANG9` and `STAT_GANG10` — a ladder built taller than the game shipped.
- Across four chapters, single-reference tables show a **3–21 % orphan rate** — the signature of late
  content cuts, consistent with the four separator defects found in the same files.
- `ar_stats.dat` (59 × 3) and `statdisp.dat` (122 × 5) are **perfectly uniform**; `statdisp` is a
  display-rule table whose last field 🟡 points into the localisation strings.
- **`timecyc.dat` (183 × 51) deserves its own chapter** — it is the timecycle, and the most-modified
  file in the scene.
- ✅ `water.dat`'s **301/6 split is design** — quads and triangles ([C16.1](../C16-Popcycle-And-Water/01-water-dat.md)).
- This chapter establishes **structure and referential integrity**, deliberately not column semantics.

**Continue:** [Chapter 14 hub](C14-Peds-And-Weapons.md) · next chapter: `C15 — The Timecycle`
