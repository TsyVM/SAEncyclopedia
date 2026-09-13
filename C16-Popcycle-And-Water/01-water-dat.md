# C16.1 — water.dat: Quads and Triangles

> **The one-sentence version:** a two-width file where the widths are polygon arity, and a water
> surface whose vertices land exactly on the world boundary the executable's constants define.

[← Chapter 16 hub](C16-Popcycle-And-Water.md) · [Next: C16.2 — popcycle.dat →](02-popcycle-dat.md)

**Confidence:** ✅ Verified over the full file
**Closes:** [C14.3 §5](../C14-Peds-And-Weapons/03-stats-and-crosschecks.md)

---

## 1. The file

```
processed                                    <- single-word header line
-1584.0 -1826.0 0.00000 0.00000 0.00000 0.05100 0.10200    -1360.0 -1826.0 ...
-3000.0   354.0 0.00000 0.00000 0.00000 1.00000 0.00000    -2832.0   354.0 ...
```

| Measurement | Value |
|---|---:|
| Lines | 308 |
| Header (`processed`) | 1 |
| Data rows | **307** |
| Rows at 29 fields | **301** |
| Rows at 22 fields | **6** |

The header is the literal word `processed` — 🟡 *reasoned:* a marker written by the export tool, not a
count or a version.

## 2. Resolving the width split

[C14.3 §5](../C14-Peds-And-Weapons/03-stats-and-crosschecks.md) flagged the 301/6 split as "either
design or defect." One line of arithmetic settles it:

```
29 = 4 × 7 + 1        four vertices of seven values, plus one trailing flag
22 = 3 × 7 + 1        three vertices of seven values, plus one trailing flag
```

**Quads and triangles.** ✅ Verified by the vertex count:

```
301 quads × 4  +  6 triangles × 3  =  1,222 vertices
```

and extracting on that assumption yields **exactly 1,222**.

That is the whole proof, and it is worth noting how cheap it was: the widths differ by 7, which is the
per-vertex stride, so the difference *is* one vertex. **When two record widths differ by a clean
divisor of both, the difference is usually a repeated group, not an error.**

C14.3's caution was right to withhold judgement — but the resolution took one division.

## 3. The vertex

Seven values per vertex:

```
x   y   z   ?   ?   u   v
```

| Field | Observation |
|---|---|
| 0–2 | position — spans the world, §4 |
| 3–4 | **`0.00000, 0.00000` on all 1,222 vertices** ✅ |
| 5–6 | small floats, 0.0–1.0 range |

🟡 *Reasoned:* fields 5–6 varying in 0–1 across a flat surface read as **texture coordinates or flow
parameters**; fields 3–4 being universally zero suggests reserved slots, or a normal that is implied
for a horizontal plane and never authored.

⏳ **Open:** confirming either. Fields 3–4 are the more interesting: a column that is constant across an
entire shipped data set is either vestigial or waiting for content that never arrived — the same shape
as `0x253F2FD`'s four zero bytes in
[C8.3 §3](../C8-Geometry/03-identifying-plugins-by-size.md).

## 4. The plane touches the boundary exactly

| Axis | Minimum | Maximum |
|---|---:|---:|
| X | **−3000.0** | **3000.0** |
| Y | **−3000.0** | **3000.0** |
| Z | −5.0 | 1082.7 |

| Check | Result |
|---|---|
| Vertices outside −3000 … +3000 | **0 of 1,222** |

✅ *Verified.*

This is the **fourth** confirmation of the world extent that
[C5.2](../C5-CWorld/02-sector-index-arithmetic.md) derived from `× 0.02 + 60.0` in compiled code — and
the first that reaches the boundary **precisely**:

| Source | Records | Closest approach to ±3000 |
|---|---:|---|
| Object placements ([C11.3](../C11-IDE-And-IPL/03-what-placements-prove.md)) | 36,569 | 6.2 units short |
| Path nodes ([C12](../C12-Path-Network/C12-Path-Network.md)) | 68,237 | 8.0 units short |
| **Water vertices** | **1,222** | **0.0 — exact** |

**Objects and paths were placed inside the world; the water was built to be the world.** A designer
positioning a building stops short of the edge; a tool generating a water plane writes the edge
coordinate itself.

That distinction turns ±3000 from "a bound the content respects" into "a value the content is
constructed from" — which is a stronger statement about the constant than any of the previous three
confirmations could make alone.

### Z reaching 1082.7

Water above 1,000 units is not sea level. 🟡 *Reasoned:* those are elevated water bodies — reservoirs,
pools, the tops of dams. Compare the ceilings established elsewhere:

| Data set | Z max |
|---|---:|
| Object placements | 1,382 |
| **Water** | **1,082.7** |
| Path nodes | 2,023 |

Water sits below the tallest geometry and well below the flight paths, which is the expected ordering
and a small consistency check across three chapters.

## 5. The trailing flag

| Value | Rows |
|---:|---:|
| 1 | 284 |
| 3 | 21 |
| 0 | 2 |

🟡 *Reasoned:* a small enumerated type — plausibly water kind (sea / pool / shallow) or a behaviour
flag. The 2 rows carrying `0` are the interesting minority, on the principle from
[C8.3 §3](../C8-Geometry/03-identifying-plugins-by-size.md) that exceptions are where meaning lives.

⏳ **Open.** Three values across 307 rows is a small enough space to settle by correlating each against
its polygon's position — a cheap follow-up not run here.

## 6. Reading it

```python
def water_polys(path):
    lines = [l.strip() for l in open(path, encoding='latin-1') if l.strip()]
    assert lines[0] == 'processed'
    for line in lines[1:]:
        f = line.split()
        n, rem = divmod(len(f) - 1, 7)
        assert rem == 0 and n in (3, 4)        # holds on all 307 retail rows
        verts = [tuple(map(float, f[i*7:(i+1)*7])) for i in range(n)]
        yield verts, int(f[-1])
```

The `divmod` assertion is the whole format check: **subtract the flag, divide by the vertex stride, and
the remainder must be zero.** It accepts both widths without special-casing either, and it would catch a
genuinely malformed row — which is the property C15.3 showed a range check cannot provide.

---

### Key takeaways

- ✅ **The 301/6 split is polygon arity** — 29 fields = 4 vertices × 7 + flag, 22 = 3 × 7 + 1. Closes
  C14.3's open question.
- **1,222 vertices**, matching `301×4 + 6×3` exactly.
- Method: **when two widths differ by a clean divisor of both, the difference is a repeated group** —
  here, one vertex.
- **The plane spans exactly −3000.0 … 3000.0 with 0 vertices outside** — the fourth confirmation of the
  world grid and the **first to land on the boundary precisely**.
- Objects and paths stop *short* of the edge; **water is built from it** — which upgrades ±3000 from a
  respected bound to a constructive constant.
- Per-vertex fields 3–4 are **`0.00000` on all 1,222** — vestigial or unauthored; ⏳ open.
- Trailing flag takes three values (1 / 3 / 0); the two `0` rows are the ones worth chasing.
- Validate with `divmod(len(fields) - 1, 7)` — accepts both widths, rejects malformed rows.

**Continue:** [C16.2 — popcycle.dat and its broken invariant](02-popcycle-dat.md)
