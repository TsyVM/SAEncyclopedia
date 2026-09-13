# C13.1 — handling.cfg

> **The one-sentence version:** one file, five record types selected by a leading sigil, and a field
> count that never varies within a type — 210 vehicle lines of exactly 36 fields, with no exceptions in
> the shipped game.

[← Chapter 13 hub](C13-Vehicle-Data.md) · [Next: C13.2 — The cars section, repaired →](02-cars-section-repaired.md)

**Confidence:** ✅ Verified (structure) / ⏳ (column meanings)

---

## 1. Five record types

Unlike `.ide` and `.ipl` ([C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md)), `handling.cfg` has no section
keywords. Record type is selected by the **first character of the line**:

| Sigil | Records | Fields | Subject |
|---|---:|---:|---|
| *(none)* | **210** | **36** | vehicle handling |
| `%` | 12 | 16 | boat handling |
| `!` | 13 | 17 | bike handling |
| `$` | 24 | 23 | flying handling |
| `^` | 30 | 36 | ⏳ unclassified |

✅ *Verified.* **Every record of a given type has exactly the same field count** — 210 vehicle lines all
at 36, 12 boat lines all at 16, and so on. No variation anywhere in the file.

That uniformity is what makes the file safe to parse by sigil dispatch, and it contrasts sharply with
the `cars` section in `vehicles.ide`, whose width genuinely does vary
([C13.2](02-cars-section-repaired.md)).

Fields are **whitespace-separated**, not comma-separated — a different convention from every other data
file in this encyclopedia. Comments begin with `;`.

```
LANDSTAL  1700.0  5008.3  2.5  0.0 0.0 -0.3  85  0.75 0.85 0.5  5 160.0 25.0 20.0 4 D  ...
%  PREDATOR   0.79  0.5   0.6   7.0   0.60  -1.9  4.0  0.8  0.998 0.998  0.85 0.98 0.97 4.0
!  BIKE       0.35  0.15  0.34  0.10  45.0  38.0  0.93 0.70 0.5   0.1    35.0 -40.0 -0.009 0.7 0.6
$  SEAPLANE   0.5   0.40  -0.00006  0.002  0.10  0.002  -0.002  ...
```

## 2. The sub-records extend, they do not replace

A boat has both a vehicle line *and* a `%` line; a bike has a vehicle line *and* a `!` line. The counts
confirm it:

| Type | `cars` rows ([C13.2](02-cars-section-repaired.md)) | Sub-records |
|---|---:|---:|
| boat | 10 | 12 (`%`) |
| bike + bmx | 13 | 13 (`!`) |
| plane + heli | 24 | 24 (`$`) |

🟡 *Reasoned:* `!` at 13 matches bike (10) + bmx (3) exactly, and `$` at 24 matches plane (13) + heli
(11) exactly. Those are not coincidences — the sub-record sets are sized to their vehicle classes.

The boat count is off by two in the other direction (12 sub-records, 10 boats), which fits the six
unreferenced handling entries in [C13 §13.3](C13-Vehicle-Data.md) — two of them (`FLOAT`, `RIO`) are
plausibly boats.

**Consequence:** a parser cannot treat the file as a flat list. A bike's physics is the *union* of its
36-field vehicle line and its 17-field `!` line, joined by the identifier in field 1.

## 3. The `^` record

30 records, 36 fields — the same width as a vehicle line, but sigil-prefixed and numerically distinct:

```
^  0  0  0  0  0  0  0  0  0  0  0  0  0  0  0  0  0  0  0  0  0
   0.5  0.0  -0.5  -0.3  0.3  0.41  0.8  0.3  0.45  0.06  0.43  0.20  0.43  0
```

The pattern is striking: **the first ~21 fields are zero and the last ~14 are small signed decimals**
clustering around ±0.5.

🟡 *Reasoned:* values in that range, in a group of roughly a dozen, read like **offsets or positions** —
plausibly seat, exhaust or component anchor points. The leading zeros would then be an unused prefix
kept for field alignment with the vehicle record.

⏳ **Open.** 30 records is a small enough population to characterise properly — census each column's
value distribution and see which are ever non-zero — and that was not done here. The identifier field
is `0` in the sampled record, so these are not keyed by vehicle name the way the other sigils are,
which is itself a clue.

## 4. Identifiers, not model names

The first field of a vehicle line is a **HandlingId**, and it is *not* the model name:

| Model name | HandlingId | Game name |
|---|---|---|
| `landstal` | `LANDSTAL` | `LANDSTK` |
| `sentinel` | `SENTINEL` | `SENTINL` |
| `firetruk` | `FIRETRUK` | `FIRETRK` |

Three different strings for one vehicle, in three different columns of the `cars` row
([C13.2](02-cars-section-repaired.md)). They are often similar and occasionally identical, which is
exactly the condition under which a tool that assumes they are the same will work for months and then
fail on `landstal` / `LANDSTK`.

**204 distinct HandlingIds are referenced** by the 212 `cars` rows, so a handful of vehicles share
handling — the natural consequence of variants that drive identically.

## 5. ⏳ Column meanings

This page verifies the file's **structure**: five record types, fixed widths per type, whitespace
separation, sigil dispatch, and the identifier relationships. It does **not** name the 36 columns.

The community has a well-known mapping (mass, turn mass, drag, centre of mass, gear count, max
velocity, traction multipliers, suspension, damage multiplier, monetary value, model flags, handling
flags, front/rear lights, animation group). Adopting it here would repeat exactly the error that
produced the `0x253F2FE` correction in
[C10.1 §3](../C10-2dEffect/01-the-record-and-corrections.md): **a name accepted without asking what the
bytes say.**

What would close it, cheaply and without a disassembler:

- **Column 2** is 1700.0 for a Landstalker and should be mass — check it against a known-heavy vehicle
  (a truck) and a known-light one (a bike). A column that orders vehicles by real-world weight is mass.
- **Column 12** is an integer 4–6 across the file and should be gear count — check its range.
- The single letter (`D` in the Landstalker line) is a small enumerated set — census it.

Each is a distribution test over 210 rows, and each either confirms or refutes a column in one pass.
That is the work this page defers rather than assumes.

---

### Key takeaways

- Record type is chosen by a **leading sigil**, not a section keyword: none / `%` / `!` / `$` / `^`.
- **210 vehicle lines of exactly 36 fields**, and every sub-record type is likewise fixed-width —
  16 / 17 / 23 / 36. No exceptions in the shipped file.
- Fields are **whitespace-separated** — unlike every other data file in this encyclopedia.
- Sub-records **extend** the vehicle line rather than replacing it; `!` (13) matches bike+bmx exactly and
  `$` (24) matches plane+heli exactly.
- The first field is a **HandlingId**, distinct from both the model name and the game name — three
  strings per vehicle, often similar, occasionally not.
- **204 distinct HandlingIds for 212 vehicles** — some share handling.
- ⏳ The `^` record (30 × 36, mostly zeros then small signed decimals) is unclassified.
- ⏳ **Column meanings are deliberately not adopted** from community sources; §5 lists three
  distribution tests that would settle the main ones without any disassembly.

**Continue:** [C13.2 — The cars section, repaired](02-cars-section-repaired.md)
