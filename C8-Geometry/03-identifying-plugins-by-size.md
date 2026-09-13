# C8.3 — Identifying Plugins by Size

> **The one-sentence version:** two unknown section IDs closed without opening a disassembler — because
> a plugin's payload size, regressed against the counts its parent already declares, constrains its
> structure to one answer.

[← C8.2 — BinMeshPLG](02-binmesh-plugin.md) · [Chapter 8 hub](C8-Geometry.md)

**Confidence:** ✅ Verified (structure) / 🟡 (interpretation)

---

## 1. The problem

Five extension IDs appear inside geometry sections. One is documented (`0x50E`,
[C8.2](02-binmesh-plugin.md)). The other four are vendor plugins in RenderWare's `0x253F2xx` range plus
one stray, and nothing in the stream says what they are:

| ID | Occurrences (of 258 geometries) |
|---|---:|
| `0x253F2FD` | 258 |
| `0x253F2F9` | 188 |
| `0x253F2F8` | 33 |
| `0x116` | 3 |

The obvious route is to find the registration code in `gta_sa.exe`. The cheaper route is to notice that
**the parent geometry already tells us `numVertices`, `numTriangles`, `numUVSets` and `numMorphTargets`
— so if a plugin's size is a function of any of those, the function identifies its record layout.**

## 2. `0x253F2F9` — one dword per vertex

Testing candidate relationships against all 188 occurrences:

| Hypothesis | Matches |
|---|---:|
| `size == 4 × numVertices` | 0 |
| **`size == 4 + 4 × numVertices`** | **188 / 188** |

✅ *Verified.* Exact, on every occurrence, across geometries whose vertex counts vary by two orders of
magnitude. A 4-byte header followed by **one dword per vertex**.

🟡 *Reasoned:* a per-vertex dword on a geometry that already has prelit colours
([C8.1 §2](01-the-geometry-struct.md)) is a **second vertex-colour set** — San Andreas's night/dusk
colours, blended against the day set by time of day. Supporting evidence from the census: it appears on
188 of 258 geometries, i.e. most but not all world geometry, which matches "buildings that light up at
night" rather than a universal property.

What is ✅ is the shape: `{ uint32 header; uint32 value[numVertices]; }`. What is 🟡 is that the dwords
are BGRA colours. Confirming that means sampling the values and checking they look like colours rather
than indices — a cheap follow-up not done here.

## 3. `0x253F2FD` — a four-byte zero

| Measurement | Value |
|---|---|
| Occurrences | 258 (every geometry) |
| Size | **4 bytes** on 253 of 258 |
| Payload value | **`0x00000000` on all 253** |

✅ *Verified.*

A section present on every geometry, four bytes long, always zero. 🟡 *Reasoned:* a reserved or
flag-bearing plugin that the shipped exporter always writes empty — the RenderWare plugin mechanism
requires a section to exist for a registered plugin even when it has nothing to say.

⏳ **Open:** the five geometries where it is *not* 4 bytes (one is 1,480). Those are the interesting
cases and were not examined. A plugin that is empty 253 times and populated 5 times is worth chasing
precisely because the exceptions are where the meaning lives.

## 4. The two still open

**`0x253F2F8`** — ✅ **CLOSED** in [C10](../C10-2dEffect/C10-2dEffect.md): it is **`2dEffect`**, a
**variable-length** array of effect records.

The suggestion below — regress against material count — was **tested in
[C9 §9.4](../C9-Materials-And-Textures/C9-Materials-And-Textures.md) and disproved**. Both this page and
C9 were searching for a fixed stride that does not exist. The failure was still useful: eliminating every
fixed-stride hypothesis is what pointed at a variable-length structure.

*Original note, kept:* 33 occurrences, small payloads (68, 104 bytes observed). No clean relationship to
vertex or triangle counts. The size range suggests a fixed-size record repeated a few times — possibly
per-material rather than per-vertex.

**`0x116`** — 3 occurrences, very large (21,035 and 24,232 bytes). ⏳ Open. Three occurrences is too few
to regress against anything. Non-`0x253F2xx` numbering suggests it is not a vendor plugin at all.

## 5. The method, stated generally

This is worth extracting because it applies well beyond geometry.

> **When a container declares counts, and a child section's size is a function of those counts, solving
> for the function determines the child's record layout — without any code.**

The procedure:

1. Collect `(sectionSize, parentCounts…)` over as many samples as possible.
2. Test candidate forms: `a + b × count`, for each count and small integer `b`.
3. A hypothesis that holds on *every* sample, across a wide range of count values, is the layout.
4. State the **shape** as verified and the **meaning** as reasoned — they are different claims.

The wide range in step 3 is what does the work. `4 + 4 × numVertices` matching 188 geometries whose
vertex counts span tens to thousands cannot be coincidence; the same match on three similar samples
could be.

This is the third time this encyclopedia has resolved a layout from data alone rather than code — after
the `center == midpoint(min, max)` test in [C6.2 §2](../C6-Collision/02-header-and-bounds.md) and the
exact-size assertion in [C8.1 §3](01-the-geometry-struct.md). It is worth treating as a standard first
move: **try to make the data prove it before opening the disassembler.**

And the standing caveat, from [C5.6 §4](../C5-CWorld/06-the-sector-arrays-closed.md): a fit is not a
meaning. `4 + 4 × numVertices` is a fact about bytes. "Night vertex colours" is an interpretation, and
it is tagged as one.

---

### Key takeaways

- **`0x253F2F9` = `4 + 4 × numVertices`, exact on 188/188** — a 4-byte header plus one dword per vertex.
  🟡 Very likely night vertex colours.
- **`0x253F2FD` = 4 bytes containing zero**, on 253 of 258 — a registered-but-empty plugin. ⏳ The five
  oversized exceptions were not examined and are the interesting ones.
- ✅ `0x253F2F8` is **`2dEffect`** ([C10](../C10-2dEffect/C10-2dEffect.md)) — variable-length, which is
  why no fixed-stride fit existed. `0x116` (3 occurrences) remains open.
- **Method:** solve a child section's size as a function of the parent's declared counts. A fit that
  holds across a wide range of count values *is* the layout.
- Report **shape as verified, meaning as reasoned** — they are separate claims.
- Third resolution-from-data-alone in this project; worth trying **before** the disassembler.

**Continue:** [Chapter 8 hub](C8-Geometry.md) · next chapter: `C9 — Materials & Texture Payloads`
