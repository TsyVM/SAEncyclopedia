# C24.2 — `surfinfo.dat`: the 37-Field Physics Record

> **The one-sentence version:** every surface's physics is one fixed 37-column record — adhesion group,
> tyre grip, wet grip, skidmark, friction effect, thirty behaviour flags and the name repeated — uniform
> across all 179 rows, with every enumerated field inside the domain the file's own header documents.

[← C24.1 — The namespace](01-the-surface-namespace.md) · [Chapter 24 hub](C24-Surfaces.md) · next:
[C24.3 — friction matrix & audio](03-friction-matrix-and-audio.md)

**Confidence:** ✅ Verified (the fixed column count, the field order from the header, the enumerated
domains, the self-labelling) / ⏳ (the numeric semantics of a few scalar and small-integer fields)

---

## 1. A fixed-width record in a free-form file

`surfinfo.dat` looks free-form — whitespace-separated columns with generous tab alignment — but it is
rigidly fixed underneath. Every one of the 179 records has exactly **37** whitespace-separated tokens:

```
surfinfo column count over all records  →  {37: 179}
```

Not 36, not 38, for any row. A fixed column count across the whole population is the text-format equivalent
of a residue-free record width: it means the reader steps field-by-field with no optional columns, and a
row that lost or gained a token would stand out immediately. None does.

✅ *Verified:* `surfinfo.dat` is a uniform **37-column** record, 179 rows.

## 2. The field order, read off the header

Unlike most binary records, this one documents itself: the file's header block names every column. Taken in
order, the 37 columns are:

| # | Field | Kind |
|---:|---|---|
| 1 | `name` | surface name |
| 2 | `adhesion_group` | enum: `RUBBER HARD ROAD LOOSE SAND WET` |
| 3 | `tyre_grip` | scalar (grip multiplier) |
| 4 | `wet_grip` | scalar (wet modifier) |
| 5 | `skidmark` | enum: `DEFAULT SANDY MUDDY` |
| 6 | `friction_effect` | enum: `NONE SPARKS` |
| 7–36 | 30 behaviour flags | booleans and small-integer scales (see §3) |
| 37 | `name` (repeated) | self-label |

The thirty behaviour flags, in column order, are: `softland`, `see_thro`, `shoot_t`, `sand`, `water`,
`s_water`, `beach`, `steep_sl`, `glass`, `stairs`, `skateable`, `pavement`, `roughness`, `flame`, `sparks`,
`sprint`, `footsteps`, `footdust`, `cardirt`, `carclean`, `w_grass`, `w_gravel`, `w_mud`, `w_dust`,
`w_sand`, `w_spray`, `proc_plant`, `proc_obj`, `climbable`, `bullet_fx`. Most are booleans (can you climb it,
does it leave footsteps, does it dirty the car); a few are small enumerations — `roughness` (0–3), `flame`
(0–2) — and the last, `bullet_fx`, is the impact-effect enum.

Because the executable holds no scalar field names (the columns are read positionally), field order is
load-bearing — the same lesson [C21](../C21-Particles/C21-Particles.md) recorded for `effects.fxp`. The
header is the authority for that order, and §3 shows the data never contradicts it.

## 3. The enumerated fields never leave their documented domains

The header does not just name the enumerated fields; it lists their legal values. Reading the actual data
back, every one stays inside its documented set across all 179 rows:

| Field | Header's documented domain | Values actually used |
|---|---|---|
| `skidmark` | `DEFAULT SANDY MUDDY` | `DEFAULT MUDDY SANDY` ✅ |
| `friction_effect` | `NONE SPARKS` | `NONE SPARKS` ✅ |
| `bullet_fx` | `NONE SPARKS SAND WOOD DUST` | `DUST SAND SPARKS WOOD` ✅ (all ⊆; `NONE` simply unused) |
| `adhesion_group` | `RUBBER HARD ROAD LOOSE SAND WET` | all six, and only those ✅ |

✅ *Verified:* every enumerated column is a subset of the domain the file's own header declares. A typo or a
mis-counted column would almost certainly have produced an out-of-domain token; none appears.

This is a self-consistency check of the kind the project values: the file describes its own grammar in
comments, and the body obeys that grammar exactly, which confirms both the column count and the field
assignment at once.

## 4. The name is repeated as the last column

The 37th column is not a physics field — it is the surface name again. All **179 / 179** records have
`column 1 == column 37`. This is the same self-labelling convention [C22](../C22-Map-Zones/01-the-zone-tables.md)
found on the zone record, where the display key sat in the last field rather than the first. Here the repeat
is pure redundancy — a human-readable end-of-row marker that lets an author scan the wide, tab-aligned table
and confirm each long row still belongs to the surface it started with. That it holds in every record is a
free integrity check on the parse: a dropped or extra column mid-row would break the equality.

## 5. What is left open

⏳ The **numeric semantics** of `tyre_grip` and `wet_grip` are not derived here. The header calls them a
tyre-grip override and a wet multiplier, and the values are consistent with that, but their exact role in
the handling model belongs to [C13](../C13-Vehicle-Data/C13-Vehicle-Data.md). Likewise the small-integer
scales `roughness` (documented 0–3) and `flame` (0–2) are read and their ranges noted, but the engine's use
of them — pad-vibration strength, fire spread — is not independently confirmed.

The **positions and domains** of all 37 fields are ✅; the physical *interpretation* of the continuous
scalars is the part left to the handling chapter and to future work.

---

### Key takeaways

- ✅ `surfinfo.dat` is a uniform **37-column** record over all **179** surfaces — a fixed-width record in
  free-form clothing.
- ✅ The field order is read from the file's **own header**: name, adhesion group, tyre/wet grip, skidmark,
  friction effect, **30 behaviour flags**, and the name repeated.
- ✅ Every **enumerated field** (`skidmark`, `friction_effect`, `bullet_fx`, `adhesion_group`) stays inside
  the domain the header documents — a self-consistency check that confirms the column assignment.
- ✅ The name is **repeated as the 37th column** in `179 / 179` rows — the same self-labelling as C22's zone
  record, and a free integrity check on the parse.
- ⏳ The numeric meaning of the grip scalars and the small-integer scales is left to
  [C13](../C13-Vehicle-Data/C13-Vehicle-Data.md) and future work.

**Continue:** [C24.3 — `surface.dat` and `surfaud.dat`](03-friction-matrix-and-audio.md)
