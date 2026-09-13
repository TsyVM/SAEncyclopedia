# C13.3 — carcols & carmods

> **The one-sentence version:** a 127-entry palette the file numbers itself, 2,360 colour references of
> which exactly one points nowhere, and a mod table where every single row resolves to a real vehicle.

[← C13.2 — The cars section, repaired](02-cars-section-repaired.md) · [Chapter 13 hub](C13-Vehicle-Data.md)

**Confidence:** ✅ Verified

---

## 1. `carcols.dat` — three sections

| Section | Rows | Content |
|---|---:|---|
| `col` | **127** | RGB triples — the shared palette |
| `car` | **196** | vehicle → colour-pair list |
| `car4` | 3 | ⏳ four-colour variants |

Section keywords open a block and `end` closes it — the same model as `.ide`
([C11.1 §1](../C11-IDE-And-IPL/01-ide-definitions.md)), and a third separator convention after IDE's
commas and `handling.cfg`'s whitespace.

## 2. The palette numbers itself

The `col` section's inline comments carry the index:

```
0,0,0            # 0 black          black
245,245,245      # 1 white          white
42,119,161       # 2 police car blue    blue
...
236,106,174      # 126 pink         light
```

✅ **127 data rows, annotated 0 through 126.** The file states its own indexing, which removes any
question about whether the palette is zero- or one-based — a rare courtesy in game data.

### ⚠️ One malformed entry

```
77.93,96
```

A period where a comma belongs. It parses as **two** fields instead of three, so **126 of 127 rows are
valid RGB triples**.

✅ Verified. 🟡 *Reasoned:* the intent is plainly `77,93,96` — a dark slate grey, consistent with its
neighbours. The typo sits at the index the comment numbering makes index 77, which is a pleasing
coincidence and not evidence of anything.

**For tooling:** a strict RGB parser rejects this row. A rebuilder that "fixes" it changes the shipped
file. Neither is wrong, but a tool should say which it did.

## 3. Colour pairs

A `car` row is a vehicle name followed by **pairs** of palette indices:

```
admiral, 34,34, 35,35, 37,37, 39,39, 41,41, 43,43, 45,45, 47,47
alpha,   58,1,  69,1,  75,77, 18,1,  32,1,  45,45, 13,1,  34,1
```

✅ **193 of 196 rows are a name followed by an even count of numeric indices** — primary and secondary
colour, up to eight schemes per vehicle. The three exceptions are rows with a different field shape,
and `car4` (3 rows) is a separate four-colour section.

The `admiral` row pairs each colour with itself (single-tone); `alpha` pairs most colours with `1`
(white) as a secondary. Both patterns are visible directly in the data.

### ⚠️ One dangling reference

| Measurement | Value |
|---|---:|
| Colour references across `car` + `car4` | **2,360** |
| Distinct indices used | 127 |
| References outside 0–126 | **1** |
| The offender | `moonbeam` → **227** |

```
moonbeam, 119,119, 117,227, 114,114, 108,108, 95,95, 81,81, 61,61, 41,41
```

✅ Verified. One reference in 2,360 points at a palette entry that does not exist. Every other index in
the row is in the 95–119 band, so `227` is 🟡 plausibly a typo for `127` — which would itself be one
past the end — or for `27`.

**This nearly became a much bigger claim.** The maximum referenced index is 227 against a 127-entry
palette, which reads like a hundred broken references. Counting them gives **one**. Maximum is not
distribution — the same trap as [C11.2 §5](../C11-IDE-And-IPL/02-ipl-placements.md)'s top-six interior
table.

⏳ **Open:** what the game does with it. A clamp, a wrap, or an out-of-bounds read are all possible, and
distinguishing them means reading the colour lookup rather than the data.

## 4. `carmods.dat` — three sections

| Section | Rows | Widths |
|---|---:|---|
| `mods` | **77** | 4 – 15 fields |
| `link` | 23 | 2 |
| `wheel` | 3 | 10 – 11 |

```
mods:   admiral, nto_b_l, nto_b_s, nto_b_tw
link:   bntl_b_ov, bntr_b_ov
wheel:  0, wheel_gn1, wheel_gn2, wheel_gn3, ... wheel_or1
```

A `mods` row lists the upgrade part names that fit a vehicle — genuinely variable-length, because
vehicles accept different numbers of upgrades. A `link` row pairs two part names; the samples are
left/right symmetric (`bntl_b_ov` / `bntr_b_ov`), so 🟡 it maps a part to its mirror.

## 5. Every name resolves

| Check | Result |
|---|---|
| `carcols` `car` rows naming a known vehicle | **196 / 196** ✅ |
| `carmods` `mods` rows naming a known vehicle | **77 / 77** ✅ |

Both files key on the **model name** from the `cars` row, and both resolve completely — against the
*repaired* row set from [C13.2](02-cars-section-repaired.md), which matters: two of the three malformed
rows are `emperor` and `wayfarer`, and both appear in `carcols`.

Only **77 of 212 vehicles** accept modifications, and **196 of 212** have colour schemes. Both are
subsets, and the difference between them is the tuning-shop roster.

## 6. Three files, three separator conventions

Worth stating together, because a tool that assumes one gets the others wrong:

| File | Separator | Comment | Structure |
|---|---|---|---|
| `.ide` / `.ipl` | comma | `#` | section keyword + `end` |
| `handling.cfg` | **whitespace** | `;` | leading sigil |
| `carcols.dat` | comma | `#` | section keyword + `end` |
| `carmods.dat` | comma | `#` | section keyword + `end` |

`handling.cfg` is the outlier on every axis — separator, comment character, and dispatch mechanism.
🟡 *Reasoned:* it is the oldest of the four, inherited with the least modification from earlier titles.

---

### Key takeaways

- `carcols.dat` has **three** sections — `col` (127), `car` (196), `car4` (3).
- **The palette numbers itself in comments, 0–126**, removing any base-index ambiguity.
- ⚠️ **One palette row is malformed** — `77.93,96`, a period for a comma — so 126 of 127 parse as RGB.
- Colour rows are **name + pairs**; 193 of 196 are a clean even-length numeric list.
- ⚠️ **Exactly one of 2,360 colour references dangles**: `moonbeam` → index 227, palette is 0–126. The
  *maximum* suggested a hundred breakages; counting gave one.
- `carmods.dat`: **77 modifiable vehicles**, 23 mirror links, 3 wheel sets.
- **Every name in both files resolves** — 196/196 and 77/77 — against the repaired `cars` rows.
- **`handling.cfg` is the outlier**: whitespace-separated, `;` comments, sigil dispatch. The other three
  share IDE conventions.

**Continue:** [Chapter 13 hub](C13-Vehicle-Data.md) · next chapter: `C14 — Peds, Weapons & Stats`
