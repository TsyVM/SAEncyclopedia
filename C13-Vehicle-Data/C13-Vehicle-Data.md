# Chapter 13 — Handling & Vehicle Data

> **Goal of this chapter:** decode the four text files that turn a vehicle model into a driveable
> vehicle — and cross-check them against each other, which turns up three malformed rows, one bad
> palette entry, and one dangling reference in the shipped game.

**Subsystem category:** Gameplay / data model
**Depends on:** [C11 — IDE & IPL](../C11-IDE-And-IPL/C11-IDE-And-IPL.md)
**Ties:** [C10](../C10-2dEffect/C10-2dEffect.md), [C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md), [C12](../C12-Path-Network/C12-Path-Network.md), [C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md), [C47](../C47-Vehicle-Dynamics/C47-Vehicle-Dynamics.md)
**RE status:** Verified
**Confidence:** ✅ Verified over the full data set
**Closes:** [C11.1 §2](../C11-IDE-And-IPL/01-ide-definitions.md) (variable `cars` row width)

---

## Deep-dive pages

- [C13.1 — handling.cfg](01-handling-cfg.md): five record types, 210 vehicles, 36 fields each.
- [C13.2 — The cars section, repaired](02-cars-section-repaired.md): three retail rows with a missing
  comma, and the false anomaly they produced.
- [C13.3 — carcols & carmods](03-carcols-and-carmods.md): the palette, the colour pairs, and the one
  index that points nowhere.
- [C13.4 — Handling runtime and modding](04-handling-runtime-and-modding.md): `CHandlingData` shared
  record; `handling.cfg` field-to-physics mapping; safe vs. breaking changes; new-vehicle mod workflow;
  frame-rate sensitivity of drag and gear-shift timing.

---

## 13.1 Four files, one vehicle

```
vehicles.ide  `cars` row  ──►  model ID, model name, TXD, TYPE, HandlingId, game name, …
                                                            │
                                     handling.cfg  ─────────┘   physics: mass, drag, gears, grip
                                     carcols.dat   ─────────►   colour pairs from a shared palette
                                     carmods.dat   ─────────►   which upgrade parts fit
```

Every vehicle in San Andreas is assembled from four text files keyed by two different identifiers —
the **model name** (`carcols`, `carmods`) and the **HandlingId** (`handling.cfg`). Those are not the
same string, and the split is what makes the cross-checks in this chapter possible.

## 13.2 The inventory

| File | Records | Structure |
|---|---:|---|
| `vehicles.ide` `cars` | **212** | 15 fields (11 for boats) |
| `handling.cfg` | **210** vehicles + 79 sub-records | 36 fields per vehicle |
| `carcols.dat` | 127 palette + 199 vehicle rows | RGB triples + colour pairs |
| `carmods.dat` | 77 mods + 23 links + 3 wheel sets | part-name lists |

`handling.cfg` also carries four auxiliary record types selected by a leading sigil
([C13.1](01-handling-cfg.md)):

| Sigil | Records | Fields |
|---|---:|---:|
| `%` boat | 12 | 16 |
| `!` bike | 13 | 17 |
| `$` flying | 24 | 23 |
| `^` (unclassified) | 30 | 36 |

## 13.3 The cross-checks

Because the same vehicles are described in four places, each file validates the others.

| Check | Result |
|---|---|
| `cars` HandlingIds resolving to a `handling.cfg` entry | **212 / 212** ✅ |
| `carcols` vehicle rows naming a known vehicle | **196 / 196** ✅ |
| `carmods` mods rows naming a known vehicle | **77 / 77** ✅ |
| `handling.cfg` entries referenced by some `cars` row | 204 / 210 — **6 unused** |
| `carcols` colour references inside the palette | **2,359 / 2,360** — **1 dangling** |

Six handling entries are defined and never used: `AIRTRAIN`, `BLOODRB`, `FLOAT`, `RANGER`, `RIO`,
`ROLLER`. 🟡 *Reasoned:* cut or code-spawned vehicles — the same residue as the unused object
definitions in [C11.3 §3](../C11-IDE-And-IPL/03-what-placements-prove.md).

## 13.4 Three defects in retail data

**Three `cars` rows are missing a comma.** In `vehicles.ide`, IDs `585` (emperor), `586` (wayfarer) and
`593` (dodo) run the model name and TXD name together, separated by a tab instead:

```
585,	emperor		emperor, 	car, 	EMPEROR, ...      <- 14 fields, not 15
```

✅ Verified. Every other one of the 212 rows parses cleanly.
Full analysis in [C13.2](02-cars-section-repaired.md).

**One palette entry is malformed.** `carcols.dat` line for colour index 77 reads `77.93,96` — a period
where a comma belongs, so it parses as two fields instead of three.

**One colour reference is dangling.** The `moonbeam` row references colour index **227**; the palette
defines 0–126. It is the only out-of-range reference in 2,360.

## 13.5 A near-miss worth recording

The comma defect in §13.4 produced a **false anomaly**. Parsing naively, row 586's fields shift by one,
so the HandlingId column reads `WAYFARE` — and `handling.cfg` contains `WAYFARER`, not `WAYFARE`. The
first pass therefore reported *"one vehicle references a non-existent handling entry."*

There is no such bug. Repairing the comma defect first gives **212 / 212** HandlingIds resolving.

The same thing nearly happened with the colour palette: the highest referenced index is 227 against 127
palette entries, which reads like a hundred broken references. Counting them gives **exactly one**.

Both are the [C11.2 §5](../C11-IDE-And-IPL/02-ipl-placements.md) lesson again — **a maximum is not a
distribution, and a parse artefact is not a finding.** Every anomaly in §13.4 survived a second pass;
the two that did not are recorded here instead of in the results.

## 13.6 What remains

⏳ **Open:** the field *meanings* in `handling.cfg`. This chapter verifies that every vehicle line has
exactly 36 fields and that the sigil records have 16/17/23/36 — the structure. Naming each column
(mass, drag coefficient, gear count, traction) means correlating against the physics code, and per
[C10.1 §4](../C10-2dEffect/01-the-record-and-corrections.md) this chapter does not adopt community
column names it has not derived.

Also open: the `^` sigil's 30 records, and the `car4` section of `carcols.dat` (3 rows).

---

### Key takeaways

- A vehicle is assembled from **four text files** keyed by **two different identifiers** — model name
  and HandlingId — which is what makes them mutually checkable.
- **212 cars, 210 handling entries**, and after repairing three malformed rows, **212/212 HandlingIds
  resolve**.
- **6 handling entries are defined and never referenced**; 🟡 cut or code-spawned vehicles.
- ⚠️ **Three retail `cars` rows are missing a comma** (emperor, wayfarer, dodo) — model and TXD name run
  together.
- ⚠️ **One palette row is malformed** (`77.93,96`) and **one colour reference dangles** (`moonbeam` → 227,
  palette is 0–126) — exactly one of 2,360.
- **The comma defect produced a false anomaly** that the first pass reported as a game bug; it was a
  parse artefact. A maximum is not a distribution, and a parse artefact is not a finding.
- ⏳ `handling.cfg` **column meanings** are deliberately not named from community sources.

**Next:** [C13.1 — handling.cfg](01-handling-cfg.md)

## See also (forward links)

handling.cfg's 210 rows become the runtime array in [C42 — Vehicle Physics](../C42-Vehicle-Physics/C42-Vehicle-Physics.md); the wheel/suspension/damage state is [C47 — Vehicle Dynamics](../C47-Vehicle-Dynamics/C47-Vehicle-Dynamics.md).

## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Loaded / consumed by:** [C10](../C10-2dEffect/C10-2dEffect.md), [C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md), [C12](../C12-Path-Network/C12-Path-Network.md), [C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md), [C47](../C47-Vehicle-Dynamics/C47-Vehicle-Dynamics.md)
- **Known bugs / gotchas:** handling reversed values NaN the sim; mass cap ignored (C42).
- **Modding:** handling.cfg + carcols/carmods are the vehicle-tuning trinity.
- **Performance:** data only; runtime cost is C42/C47.
