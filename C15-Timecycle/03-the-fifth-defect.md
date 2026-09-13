# C15.3 — The Fifth Defect

> **The one-sentence version:** one row in the timecycle is missing two values, and the obvious sanity
> check for it — "the first three fields should be a colour" — passes anyway, because the wrong values
> that land there happen to look right.

[← C15.2 — The columns the file names itself](02-self-documenting-columns.md) ·
[Chapter 15 hub](C15-Timecycle.md)

**Confidence:** ✅ Verified

---

## 1. The row

| Measurement | Value |
|---|---:|
| Data rows | 184 |
| Rows at **51** fields | **183** |
| Rows at **49** fields | **1** |

The outlier is **`RAINY_COUNTRYSIDE`, row index 6 — the 8PM sample**.

Placed beside its neighbours in the same block:

```
[5] 7PM       51 fields   21  21  21  | 135 173 193 | 255 255 255 | ...
[6] 8PM       49 fields   255         | 167 198 223 | 255 255 255 | ...
[7] 10PM      51 fields   31  31  39  | 167 198 223 | 255 255 255 | ...
```

✅ *Verified.*

Every well-formed row opens with a **three-value ambient colour**. Row 6 opens with a **single value**,
then continues with `167 198 223` — which is the *second* group (`Amb_Obj`) in a normal row, and is
byte-identical to what the 10PM row carries in that position.

**Two values are missing from the leading triple**, and every subsequent field shifts left by two.
51 − 2 = 49.

## 2. ⚠️ Why the obvious check does not catch it

A natural validator for this file:

> *"The first three fields of every row should be integers in 0–255 — an RGB colour."*

Running it over all 184 rows:

| Check | Result |
|---|---|
| Rows whose first three fields are a 0–255 triple | **184 / 184** ✅ |

**It passes on the defective row.** The three values that land in the ambient slot are `255`, `167`,
`198` — all integers, all in range, all a perfectly plausible colour.

The row is only detectable by **counting fields**:

| Check | Catches it? |
|---|---|
| First three fields are a 0–255 triple | ❌ no |
| All fields numeric | ❌ no |
| **Field count == 51** | ✅ **yes** |

That is the finding worth carrying: **a semantic plausibility check on a shifted row will usually pass,
because a shift moves valid data into the wrong slot rather than producing garbage.** The only reliable
detector for a missing separator is arity.

It is the same lesson as [C14.1 §2](../C14-Peds-And-Weapons/01-the-ped-table.md)'s column total, from
the opposite direction — there, a count *revealed* the defect; here, a value check *conceals* it.

## 3. What it does in the game

🟡 *Reasoned:* a strict positional parser reading this row assigns:

- `255` → the red component of ambient, with green and blue taken from the next group
- every later value → two slots early, so cloud colours land in water fields, and so on
- the last two named groups → **absent**, since the row runs out early

The visible result would be a wrong colour grade for rainy countryside at 8PM specifically — one weather,
one hour. That is an obscure enough combination that it could easily ship unnoticed, which is presumably
what happened.

⏳ **Open:** what the engine's own parser does. If it reads by token count it will misalign as above; if
it reads greedily per named group it may recover. Answering that means reading the loader, and it is the
difference between "this is a visible bug in retail San Andreas" and "this is a latent defect the engine
tolerates." **This page does not claim to know which.**

## 4. The running tally

Five defects across the shipped `data/` directory, all found by structural checks rather than by
inspection:

| File | Defect | Count | Detected by |
|---|---|---:|---|
| `vehicles.ide` `cars` | missing comma | 3 | field count |
| `peds.ide` `peds` | missing comma | 1 | field count |
| `carcols.dat` `col` | period for comma | 1 | field count |
| `carcols.dat` `car` | dangling index | 1 | range check |
| **`timecyc.dat`** | **two missing values** | **1** | **field count** |

**Four of five were caught by counting fields.** One — the dangling colour index — needed a range
check, and it is the only one that a field count could not have found because the row is well-formed.

Against roughly 1,000 hand-maintained rows across these files, that is a defect rate near 0.6 %: low,
and exactly the kind of residue a large team leaves under deadline. The point is not that Rockstar was
careless. It is that **any tool consuming this data will meet all five**, and a tool that treats
malformed rows as fatal cannot read the shipped game.

## 5. What a reader should do

```python
EXPECTED = 51

def timecyc_rows(path):
    for block, label, fields in raw_rows(path):
        if len(fields) != EXPECTED:
            warn(f'{block} {label}: {len(fields)} fields, expected {EXPECTED}')
            continue                      # skip, do not abort, do not guess
        yield block, label, fields
```

Three properties matter:

- **Warn, do not throw.** The shipped file contains this row; rejecting it rejects San Andreas.
- **Skip, do not repair.** Unlike the `cars` and `peds` defects
  ([C13.2 §5](../C13-Vehicle-Data/02-cars-section-repaired.md),
  [C14.1 §5](../C14-Peds-And-Weapons/01-the-ped-table.md)), this one has **no unambiguous repair** —
  two values are simply gone, and nothing in the file says what they were. Interpolating from the 7PM
  and 10PM neighbours would be inventing data.
- **Name the row.** A warning that says `RAINY_COUNTRYSIDE 8PM` is actionable; one that says
  `line 231` is not.

That asymmetry between the defect classes is the practical takeaway: **a shifted separator is
repairable because the data survives; a missing value is not.**

---

### Key takeaways

- **183 of 184 rows have 51 fields.** `RAINY_COUNTRYSIDE` 8PM has **49** — two values missing from the
  leading ambient triple, shifting every later field left by two.
- ⚠️ **A 0–255 range check on the first three fields passes on the defective row** — the shifted values
  are all plausible colours. **Only the field count catches it.**
- General principle: **semantic plausibility checks miss shifted rows**, because a shift relocates valid
  data rather than corrupting it. Arity is the reliable detector.
- 🟡 The likely visible effect is a wrong colour grade for one weather at one hour; ⏳ whether the
  engine's own parser misaligns is **not claimed** — that needs the loader.
- **Four of the five retail defects found so far were caught by counting fields.**
- **This defect is not repairable.** Unlike the comma defects, the data is gone — warn, name the row,
  and skip. Do not interpolate.

**Continue:** [Chapter 15 hub](C15-Timecycle.md) · next chapter: `C16 — popcycle & water`
