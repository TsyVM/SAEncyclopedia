# C24.3 — `surface.dat` and `surfaud.dat`

> **The one-sentence version:** `surface.dat` is a 6 × 6 lower-triangular adhesion/friction matrix whose six
> rows are exactly the adhesion groups `surfinfo` uses, and `surfaud.dat` is a 10-column table giving each
> of the 179 surfaces nine boolean audio-material flags.

[← C24.2 — the physics record](02-surfinfo-the-physics-record.md) · [Chapter 24 hub](C24-Surfaces.md)

**Confidence:** ✅ Verified (the triangular shape, the group labels, the boolean audio flags) / ⏳ (the
physical unit of the coefficients)

---

## 1. `surface.dat` is a triangular matrix

`surface.dat` is tiny — 576 bytes — and its shape is the whole point. Six labelled rows, and the *n*-th row
carries *n* numbers:

```
Rubber   6.0
Hard     3.6  2.0
Road     4.5  3.0  6.0
Loose    3.2  3.5  2.0  1.0
Sand     3.0  4.0  2.0  1.0  1.0
Wet      2.8  2.0  1.0  1.0  1.0  0.5
```

The *i*-th row (1-indexed) carries *i* values, so the rows run `1, 2, 3, 4, 5, 6` — **21** in total. That is
the lower triangle (including the diagonal) of a symmetric 6 × 6 matrix: `6 × 7 / 2 = 21`. A symmetric matrix
need only store its lower triangle because cell (i, j) equals cell (j, i) — the adhesion between Rubber and
Road is the same as between Road and Rubber — so the file stores each unordered pair once.

✅ *Verified:* `surface.dat` is the **lower triangle of a 6 × 6 symmetric adhesion matrix** — `1 + 2 + 3 +
4 + 5 + 6 = 21` values, no more and no fewer.

The **diagonal** is each material against itself: `Rubber×Rubber = 6.0`, `Hard×Hard = 2.0`,
`Road×Road = 6.0`, `Loose×Loose = 1.0`, `Sand×Sand = 1.0`, `Wet×Wet = 0.5`. The off-diagonal cells are the
adhesion between two different groups. Higher is grippier: rubber-on-rubber and rubber-on-road are the
stickiest pairings at 6.0; anything against wet bottoms out at 0.5–1.0. That ordering is physically sensible
and is worth stating, though this page does not claim a unit for the numbers (§3).

## 2. The rows *are* the adhesion groups

The matrix does not stand alone — it is the lookup table behind `surfinfo`'s `adhesion_group` column. The
six row labels of `surface.dat` are exactly the six values that column takes across all 179 surfaces:

```
surface.dat rows      :  RUBBER  HARD  ROAD  LOOSE  SAND  WET
surfinfo adhesion set :  RUBBER  HARD  ROAD  LOOSE  SAND  WET      (identical set)
```

So the friction between any two *surfaces* is computed in two steps the data makes explicit: each surface
names an adhesion group ([C24.2](02-surfinfo-the-physics-record.md)), and the group pair indexes this
matrix. 179 surfaces collapse onto 6 adhesion groups, and the 6 × 6 matrix gives the coefficient for each
group pair. The two files are a classic normalisation — the per-surface property that would repeat endlessly
is factored out into a small shared table.

✅ *Verified:* `surface.dat`'s six rows are exactly the adhesion groups `surfinfo.dat` references — the
matrix is the friction lookup for `surfinfo`'s group column.

## 3. What the coefficients are not (yet)

⏳ The matrix's **shape and role are proven**; the physical **unit** of its cells is not. Whether `6.0` is a
Coulomb friction coefficient, a scaled adhesion force, or an engine-internal analogue is not derivable from
the file, and this page does not promote a reading. What is certain is the structure — a symmetric
group-pair table — and the qualitative ordering, which tracks intuition (dry rubber grips, wet everything
slips). The quantitative interpretation is [C13](../C13-Vehicle-Data/C13-Vehicle-Data.md)'s handling model,
where `surfinfo`'s `tyre_grip` also feeds in.

## 4. `surfaud.dat`: nine boolean audio categories

The audio file is the simplest of the four. Each of the 179 surfaces gets one 10-column record: the surface
name, then **nine boolean flags** naming the acoustic material it behaves as —

```
CON  GRS  SND  GRV  WOD  WTR  MTL  LGS  TIL
concrete grass sand gravel wood water metal long-grass tile
```

Every value in those nine columns, across all 179 rows, is `0` or `1`:

```
surfaud category-column value domain  →  {0, 1}
```

✅ *Verified:* `surfaud.dat` is a **10-column** record — name plus **9 boolean** audio-material categories,
strictly `{0, 1}`.

The flags select which footstep, impact and scrape sample set a surface uses. They are not mutually
exclusive in principle — the format is nine independent booleans, not a single enum — which lets a surface
read as, say, both concrete and tile if an author wanted, though in practice most rows set exactly one. And
because `surfaud`'s rows are in the **same order** as `surfinfo`'s ([C24.1](01-the-surface-namespace.md)),
the engine indexes a surface's audio categories and its physics from the one surface index, with no
cross-file name lookup.

## 5. The system in one picture

```
                 surface name  ─────────────┐
                                            │  (name → index, via the exe's string table)
   surfinfo.dat  [179 × 37]  physics + adhesion_group ─┐
   surfaud.dat   [179 × 10]  9 audio-material booleans │  paired by row index
                                                       │
   adhesion_group ──► surface.dat [6 × 6 triangle] ──► friction coefficient for a group pair
```

One namespace, resolved to an index by the executable; two 179-row tables paired by that index for physics
and audio; and a small 6 × 6 matrix that turns adhesion-group pairs into friction. Every arrow in that
picture is a check this chapter closes.

---

### Key takeaways

- ✅ `surface.dat` is the **lower triangle of a 6 × 6 symmetric adhesion matrix** — rows of `1…6`, **21**
  values — with the material-against-itself coefficients on the diagonal.
- ✅ Its six rows are **exactly** the adhesion groups `surfinfo.dat` uses; the matrix is the friction lookup
  for that column, normalising 179 surfaces onto 6 groups.
- ✅ `surfaud.dat` is a **10-column** record — name plus **9 boolean** audio-material categories, all
  `∈ {0, 1}` — in the **same row order** as `surfinfo`, so physics and audio share one index.
- ⏳ The **physical unit** of the friction coefficients is not derived; the structure and ordering are.

**Continue:** [Chapter 24 hub](C24-Surfaces.md)
