# Chapter 10 — 2dEffect & the Map's Lighting

> **Goal of this chapter:** decode the records that make the world more than geometry — lights,
> coronas, cover points, ped attractors — and correct two misidentifications this encyclopedia made in
> C7 and C8 along the way.

**Subsystem category:** World / rendering data
**Depends on:** [C8 — Geometry & the Binary Mesh](../C8-Geometry/C8-Geometry.md),
[C9 — Materials & Texture Payloads](../C9-Materials-And-Textures/C9-Materials-And-Textures.md)
**Ties:** [C0](../C0-Binary-Identity/C0-Binary-Identity.md), [C3](../C3-Model-Stores/C3-Model-Stores.md), [C5](../C5-CWorld/C5-CWorld.md), [C6](../C6-Collision/C6-Collision.md), [C7](../C7-RenderWare-Stream/C7-RenderWare-Stream.md), [C8](../C8-Geometry/C8-Geometry.md), [C9](../C9-Materials-And-Textures/C9-Materials-And-Textures.md)
**RE status:** Verified
**Confidence:** ✅ Verified over the full population
**Closes:** [C8.3 §4](../C8-Geometry/03-identifying-plugins-by-size.md),
[C9 §9.4](../C9-Materials-And-Textures/C9-Materials-And-Textures.md)
**Corrects:** [C7.3 §2](../C7-RenderWare-Stream/03-clumps-and-txd.md)

---

## Deep-dive pages

- [C10.1 — The record, and two corrections](01-the-record-and-corrections.md): what `0x253F2F8` and
  `0x253F2FE` actually are.
- [C10.2 — The effect-type census](02-effect-type-census.md): eight types, one fixed size each, across
  the whole game.
- [C10.3 — The light record](03-the-light-record.md): 80 bytes, and the two texture names every light
  in San Andreas shares.

---

## 10.1 The headline: `0x253F2F8` is `2dEffect`

[C8.3](../C8-Geometry/03-identifying-plugins-by-size.md) left `0x253F2F8` unidentified. C9 predicted it
would regress against material count and [was proven wrong](../C9-Materials-And-Textures/C9-Materials-And-Textures.md).

It is **2dEffect** — a variable-length array of effect records, which is exactly why no fixed-stride
hypothesis ever fit.

Parsed as `{ uint32 count; repeat: float pos[3]; uint32 type; uint32 dataSize; byte data[dataSize] }`
over **every DFF in `gta3.img`**:

| Measurement | Value |
|---|---:|
| DFFs scanned | 12,955 |
| DFFs carrying a `2dEffect` section | **1,681 (13.0 %)** |
| Sections walked | 1,681 |
| **Walks landing exactly on the section end** | **1,681 / 1,681** |
| Effect entries | **17,395** |

✅ Zero mismatches over the full population. The same standard as
[C8.1 §3](../C8-Geometry/01-the-geometry-struct.md).

## 10.2 And `0x253F2FE` is *not* 2dEffect

[C7.3 §2](../C7-RenderWare-Stream/03-clumps-and-txd.md) labelled `0x253F2FE` as `2dEffect` and built a
paragraph on it — "this is where the map's lighting actually comes from."

**That was wrong.** `0x253F2FE` sits in `Clump/FrameList/Extension`, and its payload is a plain ASCII
string: `waterjumpx2`, `pier69_models04`, `lodpdmdocka_las2`. It is the **frame node-name plugin**.

The conclusion happened to be right — the map's lighting *does* hang off models rather than living in a
separate file — but it was attached to the wrong section ID, and the reasoning was an unverified
lookup-table guess. Details and the lesson in [C10.1 §3](01-the-record-and-corrections.md).

## 10.3 Eight types, one size each

| Type | Entries | `dataSize` | Reading |
|---:|---:|---:|---|
| 9 | **14,908** | 12 | cover point |
| 0 | 1,038 | 80 | **light / corona** |
| 3 | 793 | 56 | ped attractor |
| 7 | 489 | 88 | roadsign |
| 6 | 78 | 44 | enter / exit marker |
| 1 | 54 | 24 | particle emitter |
| 8 | 30 | 4 | trigger point |
| 10 | 5 | 40 | escalator |

✅ **Every type has exactly one `dataSize` across all 17,395 entries in the game.** Not a distribution —
a single value each. That uniformity is overwhelming structural confirmation that the type field selects
a fixed record layout, and it is what promotes the type readings from guesswork to a table you can
build a parser on.

🟡 The *names* in the last column are the community reading, consistent with the sizes and with the
models that carry each type. The **type numbers and their sizes** are ✅ verified.

**Type 9 is 85.7 % of every effect in the game.** San Andreas's world is, by record count,
overwhelmingly a mesh of cover points — a fact about the AI's spatial awareness that falls out of an
asset census rather than any code.

## 10.4 Every light shares two textures

All **1,038** light records in `gta3.img` name the same corona texture and the same shadow texture:

```
corona texture : "coronastar"
shadow texture : "shad_exp"
```

✅ Verified on 1,038 of 1,038 — no exceptions. Details and the off-by-one that nearly hid this in
[C10.3](03-the-light-record.md).

## 10.5 Dependencies

```
2dEffect (0x253F2F8)                          [C10]
  ├── lives in: Geometry / Extension          [C8.1]
  ├── positions are model-local               -> placed by IPL   [not yet written]
  ├── type 0 references corona + shadow textures by NAME  [C9.2]
  └── consumed by: the corona renderer, AI cover queries, ped attractors
```

## 10.6 What remains

⏳ **Open:** the per-type payload field layouts for types 1, 3, 6, 7, 8, 9 and 10. This chapter decodes
the container, the type table and the **full** type-0 (light) record; the other seven payloads are
located and sized but not field-mapped.

Type 9 at 12 bytes is the obvious next target — 14,908 instances and only three dwords to account for.

---

### Key takeaways

- **`0x253F2F8` is `2dEffect`** — closing C8.3's open item and explaining why C9's fixed-stride
  prediction failed: the section is a **variable-length record array**.
- Verified over the **full population**: 1,681 sections, **1,681 exact walks**, 17,395 entries, zero
  mismatches.
- ⚠️ **`0x253F2FE` is the frame node-name plugin, not 2dEffect** — C7.3's identification is withdrawn.
- **Eight effect types, each with exactly one `dataSize` game-wide** — the uniformity is what makes the
  type table trustworthy.
- **Type 9 (cover points) is 85.7 %** of all effects in the game.
- **All 1,038 lights share one corona texture and one shadow texture** — `coronastar` and `shad_exp`.

**Next:** [C10.1 — The record, and two corrections](01-the-record-and-corrections.md)

## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Loaded / consumed by:** [C0](../C0-Binary-Identity/C0-Binary-Identity.md), [C3](../C3-Model-Stores/C3-Model-Stores.md), [C5](../C5-CWorld/C5-CWorld.md), [C6](../C6-Collision/C6-Collision.md), [C7](../C7-RenderWare-Stream/C7-RenderWare-Stream.md), [C8](../C8-Geometry/C8-Geometry.md)
- **Known bugs / gotchas:** 2dEffect misplacement (lights/coronas offset from model).
- **Modding:** 2dEffect entries add lights/particles/ped-attractors to models.
- **Performance:** small per-visible-model cost.
