# C16.2 — popcycle.dat and Its Broken Invariant

> **The one-sentence version:** the file states, in its own comments, that a group of columns must sum
> to 100 — and 41 of its 480 rows do not, which means retail San Andreas ships data that trips its own
> documented warning.

[← C16.1 — water.dat](01-water-dat.md) · [Chapter 16 hub](C16-Popcycle-And-Water.md)

**Confidence:** ✅ Verified over all 480 rows
**Corrects:** [C14 §14.1](../C14-Peds-And-Weapons/C14-Peds-And-Weapons.md)

---

## 1. The file explains itself

Like `timecyc.dat` ([C15.2](../C15-Timecycle/02-self-documenting-columns.md)), `popcycle.dat` opens
with a prose header — and this one is unusually generous:

```
// Fore each type of zone (Business, Countryside etc) we have a that controls the ped densities.
// There is a set of values for each time of the day in 2 hour increments (midnight, 2am, 4 am etc)
// There are 2 sets of values: Weekday & Weekend.
// #Peds is the maximum number of peds you can ever have.
// #Cars is the maximum number of cars you can ever have.
// 4 Values for Dealers Gang Cops Other. These are percentages and are used to take the
// number of Dealers, gang members etc down during certain times of the day.
// These numbers do not add up to anything. If they are all 100% you would get the full #Peds.
// A number of values for the different types of peds that are not Dealers, Gang members or cops.
// This is what is specified as 'Other' in the first 4 values.
// This number should add up to 100%. If it doesn't the game will scale it to 100% and print a warning
```

✅ Reproduced verbatim. Note the two typos in the first line — *"Fore each"* and *"we have a that
controls"* — a missing word and a transposition, consistent with the hand-editing residue found
throughout `data/` ([C14 §14.3](../C14-Peds-And-Weapons/C14-Peds-And-Weapons.md)).

The header is doing three things at once: naming the columns, describing the block structure, **and
stating a validation rule**. The third is rare enough to be worth testing.

## 2. The structure

```
480 rows = 20 zone types × 2 (Weekday / Weekend) × 12 two-hour slots
```

✅ Verified two ways:

- **20 `// Weekday` and 20 `// Weekend` banners** counted directly.
- The header's own *"2 hour increments"* gives 24 ÷ 2 = **12** slots, and 20 × 2 × 12 = **480** — the
  exact data-row count.

Zone banners include `BUSINESS`, `DESERT`, `COUNTRYSIDE`, `RESIDENTIAL_RICH`, `RESIDENTIAL_AVERAGE`,
`RESIDENTIAL_POOR`, `GANGLAND`, `BEACH`, `PARK`, `INDUSTRY`, `AIRPORT`, `GOLF_CLUB`,
`OUT_OF_TOWN_FACTORY`, `AIRPORT_RUNWAY` and a compound `SHOPPING_BUSY  LAS VEGAS`.

🟡 A regex-based banner scan finds 15 of the 20 because several names carry irregular internal spacing.
The `Weekday`/`Weekend` count is the reliable one, and it is exactly 20 each.

## 3. ⚠️ The invariant fails on 8.5 % of rows

The header says the ped-type percentages **should add up to 100 %**. Summing those columns across every
row:

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

| Result | Value |
|---|---:|
| Rows satisfying the documented invariant | **439 / 480 (91.5 %)** |
| **Rows violating it** | **41 / 480 (8.5 %)** |

✅ *Verified.*

**Retail San Andreas ships 41 rows that break a rule its own file documents.** Sums range from 80 to
180 — some well under, some nearly double.

### This is not silent

The header states the consequence too: *"the game will scale it to 100 % and print a warning."* So the
engine handles it, and the failure mode is defined rather than corrupting. But it does mean:

- the shipped game **triggers its own diagnostic** during load, 41 times;
- **relative** proportions within a violating row are preserved by the rescale, so the gameplay effect
  is nil — a row summing to 150 produces the same distribution as one summing to 100 with the same
  ratios;
- what is actually lost is the **absolute** reading. A row summing to 150 does not mean "150 % of
  peds" — after rescaling it means exactly what its normalised proportions say.

🟡 *Reasoned:* the violations are therefore cosmetic rather than behavioural, which is presumably why
they survived to release. The rescale makes the invariant advisory in practice.

## 4. Why this is a different class of defect

Five defects were catalogued through C15 — missing commas, a period for a comma, a dangling index, two
missing values. All are **malformed syntax**: the row cannot be parsed as intended.

These 41 are **well-formed rows that violate a documented semantic rule.** Every one parses cleanly to
24 numeric fields. No structural check finds them:

| Check | Catches the 41? |
|---|---|
| Field count == 24 | ❌ no |
| All fields numeric | ❌ no |
| Values in plausible range | ❌ no |
| **Tail sums to 100** | ✅ **yes** |

This completes a progression the encyclopedia has been building:

| Defect class | Detector | Example |
|---|---|---|
| Missing separator | field count | [C13.2](../C13-Vehicle-Data/02-cars-section-repaired.md), [C14.1 §4](../C14-Peds-And-Weapons/01-the-ped-table.md) |
| Missing value | field count | [C15.3](../C15-Timecycle/03-the-fifth-defect.md) |
| Out-of-range reference | range check | [C13.3 §3](../C13-Vehicle-Data/03-carcols-and-carmods.md) |
| **Broken semantic invariant** | **domain rule** | **C16.2** |

**Each class needs its own detector, and the earlier ones do not find the later ones.** A validator that
only counts fields would pass this file completely.

The reason these 41 were findable at all is that **the file wrote its own rule down**. Without that
comment there would be no basis for calling them defects rather than intentional values — which is a
concrete argument for self-documenting data formats, and the second time in two chapters that a header
comment has been the decisive evidence ([C15.2](../C15-Timecycle/02-self-documenting-columns.md) being
the first).

## 5. ⚠️ Correcting C14's inventory

[C14 §14.1](../C14-Peds-And-Weapons/C14-Peds-And-Weapons.md) recorded `popcycle.dat` as **691 rows, 480
at 26 fields**. Both figures are wrong:

| | C14 | Actual |
|---|---|---|
| Rows | 691 | **480** |
| Fields | 26 | **24** |

One cause: C14's census stripped `#` comments but not `//`.

- **691** = 480 data rows + 211 `//` comment lines counted as data.
- **26** = 24 real fields + **an inline `//` comment present on every single row**, contributing two
  extra whitespace tokens.

✅ Corrected: **480 rows, 24 fields, and 480 of 480 rows carry an inline comment** — the time-of-day
label for each slot.

**The general lesson:** `popcycle.dat` and `timecyc.dat` use `//`; most of `data/` uses `#`. A census
that assumes one convention mis-measures the other, in both directions at once — inflating the row
count *and* the field count. **Comment style is a per-file property.**

---

### Key takeaways

- The header **names the columns, describes the block structure, and states a validation rule** — and
  contains two typos.
- Structure: **20 zones × 2 day-types × 12 two-hour slots = 480**, verified by banner count and by the
  header's own "2 hour increments".
- ⚠️ **41 of 480 rows (8.5 %) violate the file's own documented 100 % invariant**, with sums from 80 to
  180.
- The engine **rescales and warns**, so retail San Andreas trips its own diagnostic 41 times; the effect
  is cosmetic because relative proportions survive.
- This is a **new defect class** — well-formed rows breaking a semantic rule. No structural check finds
  them; **each defect class needs its own detector**.
- They are findable **only because the file documented its own rule** — the second chapter running where
  a header comment is the decisive evidence.
- ⚠️ **C14's inventory was wrong twice over** (691→480 rows, 26→24 fields) from stripping `#` but not
  `//`. **Comment style is per-file.**

**Continue:** [Chapter 16 hub](C16-Popcycle-And-Water.md)
