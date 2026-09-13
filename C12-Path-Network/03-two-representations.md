# C12.3 — Two Representations of One Network

> **The one-sentence version:** the text divided by 16 and the binary divided by 8 land on the same
> world extents to within 0.21 units — the exact quantisation error of storing those positions as
> `int16` — which proves they are one network in two encodings.

[← C12.2 — The text path source](02-text-path-source.md) · [Chapter 12 hub](C12-Path-Network.md)

**Confidence:** ✅ Verified

---

## 1. The measurement

Both forms were decoded independently — the text as ⅟₁₆-unit fixed point
([C12.2 §4](02-text-path-source.md)), the binary as `int16 ÷ 8`
([C12.1 §3](01-nodes-dat.md)) — and then their world extents compared:

| Extreme | Text ÷ 16 | Binary ÷ 8 | Δ |
|---|---:|---:|---:|
| X min | −2992.07 | −2992.00 | **0.069** |
| X max | 2946.30 | 2946.25 | **0.050** |
| Y min | −2932.66 | −2932.50 | **0.156** |
| Y max | 2853.84 | 2853.62 | **0.213** |

✅ *Verified.* Four extremes, four agreements, all under a quarter of a unit.

## 2. Why the deltas are exactly the right size

This is the part that makes it proof rather than coincidence.

The binary stores positions as `int16` at ⅛-unit precision. Rounding a continuous value to the nearest
⅛ produces a maximum error of **0.0625 units**, and the observed deltas cluster right around that:
0.069, 0.050, 0.156, 0.213.

The two larger deltas exceed one quantisation step, which is also expected — the *extreme* of a
rounded set is not the rounded *extreme*. If the binary's outermost node is not the same node as the
text's outermost, the gap grows by whatever separates them.

**A wrong scale factor would not do this.** Divide the text by 8 and the extents come out at ±5984 —
twice the world. Divide by 32 and they come out at ±1496 — half. Only 16 lands on the binary's range,
and it lands within the precision the binary is capable of representing.

So: the text is ⅟₁₆ fixed point, the binary is ⅛, **the binary carries half the precision of its
source**, and they describe the same node network.

## 3. The third confirmation of the sector grid

[C5.2](../C5-CWorld/02-sector-index-arithmetic.md) derived the world extent from three floating-point
instructions. Two later chapters tested it against real content:

| Chapter | Evidence | Records | Outside the grid |
|---|---|---:|---:|
| [C5.2](../C5-CWorld/02-sector-index-arithmetic.md) | `× 0.02 + 60.0` in compiled code | — | — |
| [C11.3](../C11-IDE-And-IPL/03-what-placements-prove.md) | object placements | 36,569 | **0** |
| **C12** | **navigation nodes** | **68,237** | **0** |

✅ **104,806 records from two unrelated data pipelines, zero outside a boundary read out of the
executable.**

The three sources share no mechanism. The constants are compiler output; the placements are a map
editor's export; the nodes are a path-tool's export. Their agreement is what promotes
"−3000 … +3000 is the world" from a reading of two instructions to a fact about San Andreas.

### Z tells a different story

The X/Y agreement is tight; Z is not gridded at all, and the two data sets disagree usefully:

| Source | Z max |
|---|---:|
| Object placements ([C11.3](../C11-IDE-And-IPL/03-what-placements-prove.md)) | 1,382 |
| **Navigation nodes** | **2,023** |

The path network reaches **641 units above the highest placed object**. 🟡 *Reasoned:* those are flight
paths — nodes for aircraft, which need a network where there is deliberately no geometry.

## 4. Which is authoritative

| | Text (`paths*.ipl`) | Binary (`NODES*.DAT`) |
|---|---|---|
| Precision | ⅟₁₆ unit | ⅛ unit |
| Live nodes | 100,089 | 68,237 |
| Links | — | 143,622 |
| Area partition | none | 8 × 8 × 750 units |
| Read at runtime | 🟡 incidentally | ✅ yes |

The binary is **lower precision but richer**: it adds the link graph, the area partition, and the
vehicle/ped split, none of which the text carries. It is a compiled artefact, and the text is closer to
source.

**For tooling this matters.** Editing the text and expecting the game to change will not work — the
runtime reads the binary. Editing the binary means regenerating link indices and area assignments,
because `linkId` at `+0x08` and `areaId` at `+0x0A` are positional
([C12.1 §3](01-nodes-dat.md)). Neither file is a convenient edit target, which is why path editing has
always been the hardest part of GTA SA map modding.

## 5. A pattern worth naming

This chapter's proof follows the same shape as three earlier ones:

| Chapter | Two independent readings that had to agree |
|---|---|
| [C6.2 §2](../C6-Collision/02-header-and-bounds.md) | `center` must be the midpoint of `min`/`max` |
| [C7.2 §4](../C7-RenderWare-Stream/02-version-encoding.md) | RW 3.6 from exe strings *and* from asset headers |
| [C9.3 §3](../C9-Materials-And-Textures/03-texturenative-payload.md) | `depth` cross-checking `d3dFormat` |
| **C12.3** | **text ÷ 16 == binary ÷ 8** |

In each case a quantity is recoverable two ways, and their agreement — to a precision the format
explains — is the verification. It is stronger than any single reading, and it costs nothing but
noticing that the redundancy exists.

The counter-example remains [C5.6 §4](../C5-CWorld/06-the-sector-arrays-closed.md), where two
derivations *appeared* independent and shared a false premise. The test is whether the two paths could
have disagreed. Here they could have: a wrong scale factor gives ±5984 or ±1496, and neither is the
world.

---

### Key takeaways

- **Text ÷ 16 matches binary ÷ 8 to within 0.05–0.21 units** at all four X/Y extremes.
- The deltas are **the exact size ⅛-unit quantisation produces** — which is what makes it proof rather
  than coincidence. A wrong scale gives ±5984 or ±1496.
- The binary carries **half the precision of its source** and adds links, areas and the vehicle/ped
  split.
- ✅ **104,806 records across two unrelated pipelines, zero outside the −3000 … +3000 grid** — the third
  confirmation of [C5.2](../C5-CWorld/02-sector-index-arithmetic.md).
- **Nodes reach Z = 2,023 against geometry's 1,382** — 🟡 flight paths, where there is deliberately no
  world.
- The runtime reads the **binary**; editing the text changes nothing, and editing the binary means
  regenerating positional `linkId` and `areaId`.
- The recurring method: **find a quantity recoverable two ways and check they agree to a precision the
  format explains.**

**Continue:** [Chapter 12 hub](C12-Path-Network.md) · next chapter: `C13 — Handling & Vehicle Data`
