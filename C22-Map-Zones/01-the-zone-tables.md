# C22.1 — The Zone Tables

> **The one-sentence version:** `info.zon` and `map.zon` are the same ten-field text record — a name, a
> type, an axis-aligned box, an island and a text key — and the tenth field is proved to be a GXT key
> because **all 169 distinct values hash onto keys stored in `american.gxt`**, with no exceptions to name.

[← Chapter 22 hub](C22-Map-Zones.md) · [Next: C22.2 — The radar grid →](02-the-radar-grid.md)

**Confidence:** ✅ Verified (grammar, field order, world extent, box validity, GXT resolution) / 🟡 (the
island field identifies the three cities) / ⏳ (the `type` enumeration)

---

## 1. Two files, one record

Both files are plain CRLF text wrapping a single section:

```
zone
SUNMA, 0, -2353.17, 2275.79, 0.0, -2153.17, 2475.79, 200.0, 1, SUNMA
SUNNN, 0, -2741.07, 2175.15, 0.0, -2353.17, 2722.79, 200.0, 1, SUNNN
BATTP, 0, -2741.07, 1268.41, -4.57764e-005, -2533.04, 1490.47, 200.0, 1, BATTP
  ⋮
end
```

| File | Bytes | Records | Fields per record |
|---|---:|---:|---:|
| `info.zon` | 28,139 | **378** | 10 in **378 / 378** |
| `map.zon` | 436 | **6** | 10 in **6 / 6** |

Every non-blank line is either a section marker (`zone`, `end` — exactly those two in both files) or a
ten-field record. There is nothing else in either file. ✅

## 2. The executable names the fields

Field *order* is the part a text table will not tell you: ten comma-separated values could be grouped many
ways. The executable settles it. At VA `0x868D04` sits a `scanf` format string with exactly ten
conversions, referenced once, from `0x5B4AE9`:

```
"%s %d %f %f %f %f %f %f %d %s"
  │   │   └────── six floats ──────┘  │   │
  │   └─ int                          │   └─ string
  └─ string                           └─ int
```

Six consecutive floats between two `%d`s, bracketed by a leading and a trailing string. Matched against
the data, that fixes the record:

| # | Type | Field | Evidence it is what it looks like |
|---:|---|---|---|
| 1 | `%s` | name | 377 distinct across 378 records (§4) |
| 2 | `%d` | type | `0` on all info zones, `3` on all map zones ⏳ |
| 3–5 | `%f` | box minimum, X Y Z | see §3 |
| 6–8 | `%f` | box maximum, X Y Z | see §3 |
| 9 | `%d` | island | `1` everywhere in `info.zon`; `1`/`2`/`3` in `map.zon` (§5) |
| 10 | `%s` | text key | **169 / 169** resolve as GXT keys (§6) |

✅ *Verified:* the record is `{name, type, min.xyz, max.xyz, island, textKey}`.

## 3. The boxes are well-formed, and they bound the world

If fields 3–8 are a minimum corner followed by a maximum corner, then `min < max` must hold on every axis
of every record. It does — **384 of 384**, counting both files, with no exceptions. Read in any other
grouping (say, a centre plus a size, or X-pair/Y-pair/Z-pair) the inequality would not hold uniformly.

The extents are the more interesting measurement:

| | X | Y | Z |
|---|---|---|---|
| `map.zon` | **−3000.0 … 3000.0** | **−3000.0 … 3000.0** | −500.0 … 500.0 |
| `info.zon` | −2997.47 … 2997.06 | −2892.97 … 2993.87 | −242.99 … 900.0 |

✅ `map.zon`'s six regions between them span **exactly** `−3000` to `+3000` on both horizontal axes — not
approximately, but on the round number, because two of the six records carry `±3000.0` as a literal. **The
world is a 6,000-unit square**, and every grid in [C22.2](02-the-radar-grid.md) and
[C22.3](03-gridref-and-the-build-tree.md) divides that number.

✅ Every `info.zon` box lies **inside** that square — the tightest zone corner is 2.53 units from the
boundary. A zone table that overran the world square would indicate a misread field; none does.

The vertical axis is looser and more revealing of authoring practice: **341 of 378** info zones have a Z
span of exactly **200.0**, with the rest at 100 (12 zones), 300 (4), 624 (4), 1143 (9) and a scattering of
others. 🟡 The obvious reading is that 200 is the template height an artist accepted unless the zone needed
to be taller — the ones that differ are the interiors and the tall terrain. The heights themselves are
data, not inference; the explanation is 🟡.

## 4. Names, and the one duplicate

`info.zon` has **377 distinct names across 378 records**. The exception is named rather than rounded away:

> **`MONINT`** appears **twice**, as two separate boxes. Every other name is unique.

The names are terse five-to-six character identifiers (`SUNMA`, `BATTP`, `PARA`, `MONINT`) and they are
*not* the same namespace as the text keys — only **110 of 378** records have `name == textKey`. Of the 377
distinct names, only 116 happen to resolve as GXT keys, which is what one expects of internal identifiers
that were never meant to be displayed.

⚠️ The trap here is assuming the first field is the label because it sometimes equals the last. It is the
**tenth** field that is the label, and §6 is what proves it.

## 5. `map.zon`: six regions, three islands

The whole file:

| Name | Type | Island | Box (X, Y) |
|---|---:|---:|---|
| `Vegas` | 3 | **3** | 685.0 … 3000.0, 476.09 … 3000.0 |
| `SF01` | 3 | **2** | −3000.0 … −1270.53, −742.31 … 1530.24 |
| `SF02` | 3 | **2** | −1270.53 … −1038.45, −402.48 … 832.50 |
| `SF03` | 3 | **2** | −1038.45 … −897.55, −145.54 … 376.63 |
| `LA01` | 3 | **1** | 480.0 … 3000.0, −3000.0 … −850.0 |
| `LA02` | 3 | **1** | 80.0 … 1075.0, −2101.61 … −1239.61 |

🟡 The island field identifies the three cities. The correspondence with the names is perfect — every
`LA*` record carries 1, every `SF*` carries 2, `Vegas` carries 3 — and the names are themselves data in
the file rather than an outside interpretation. It is not promoted to ✅ because a three-value
correspondence, however clean, is a small sample on which to declare an enumeration.

Note also what `map.zon` is *not*: the six boxes do not tile the world square. They are three named
clusters with gaps between them, which is consistent with a lookup that returns "no island" for the
countryside between the cities. ⏳ Not derived; recorded as an observation.

`gta.dat` loads both files through the **IPL** path — the same loader
[C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md) documents — and the ordering constraint is written in the
manifest as a comment:

```
# have to load map.zon before any of the IPLs
IPL DATA\MAP.ZON
# IPL DATA\NAVIG.ZON
IPL DATA\INFO.ZON
```

⚠️ The middle line is **commented out**, and `data/navig.zon` does not exist in the retail tree. A third
zone file was planned and cut, and the manifest still carries its ghost. Named, since the encyclopedia
records absences as well as presences.

## 6. The cross-check: 169 for 169

[C19](../C19-GXT-Text/C19-GXT-Text.md) derived the GXT key hash as **CRC-32 with the final complement
omitted, applied to the uppercased key**, and confirmed it against the text keys named literally in
[C18](../C18-SCM-Script/C18-SCM-Script.md)'s mission scripts — 1,180 of 1,186.

The zone table supplies a second population of candidate keys that C19 never saw. Walk `american.gxt`,
collect all **16,588** stored key hashes across its 127 tables, then hash the **169** distinct values of
`info.zon`'s tenth field:

```
resolved: 169 / 169          misses: none
```

✅ Every one. Three separate things are confirmed by that single number: the hash function is right, the
container walk that produced the 16,588 hashes is right, and the tenth field of the zone record is a GXT
key rather than a second internal name. A wrong hash resolves ~0 %; a wrong field resolves a scattering.

The reuse pattern is itself informative — 378 records share only 169 labels, with `MUL` used 13 times,
`ROD` 12, `RIH` 10, `LDT` 9 and `SFAIR` 8. Several boxes make up one named neighbourhood, which is exactly
why the label is a separate field from the name.

### 6.1 The key that correctly fails

All six `map.zon` records carry the literal `UNUSED` in field ten, and `UNUSED` hashes to **no** stored
key. This is recorded as a confirmation rather than an exception: an island region has no on-screen label,
so the field holds a sentinel, and a sentinel that resolved to a real GXT string would be the surprising
result.

---

### Key takeaways

- ✅ Both files are a single `zone` … `end` section of **ten-field** comma-separated records —
  `378 / 378` and `6 / 6`, with nothing else in either file.
- ✅ Field order is fixed by the executable's own `scanf` format `"%s %d %f %f %f %f %f %f %d %s"` at VA
  `0x868D04`, referenced once from `0x5B4AE9`.
- ✅ **384 / 384** boxes satisfy `min < max` on all three axes; `map.zon` spans **exactly −3000 … +3000**
  on X and Y, and every `info.zon` box lies inside it.
- ✅ **169 / 169** distinct text keys resolve against C19's GXT — confirming the hash, the container walk
  and the field's identity at once. `map.zon`'s `UNUSED` correctly resolves to nothing.
- ⚠️ Named exceptions: **`MONINT`** is the one duplicated zone name; **`NAVIG.ZON`** is commented out in
  `gta.dat` and absent from the tree.
- 🧭 Name and label are **different namespaces** — only 110 of 378 records have `name == textKey`, and 378
  records share just 169 labels.
- 🟡 341 of 378 zones are exactly 200 units tall (an authoring default); the island field identifies
  LA / SF / Vegas. ⏳ The `type` enumeration is not derived.

**Continue:** [C22.2 — The radar grid](02-the-radar-grid.md)
