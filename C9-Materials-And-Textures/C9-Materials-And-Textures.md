# Chapter 9 — Materials & Texture Payloads

> **Goal of this chapter:** decode the last unopened layer of the RenderWare asset tree — how a
> geometry names its surfaces, how a surface names its texture, and what a texture actually contains
> once you reach the pixels.

**Subsystem category:** Rendering / asset format
**Depends on:** [C7 — The RenderWare Stream](../C7-RenderWare-Stream/C7-RenderWare-Stream.md),
[C8 — Geometry & the Binary Mesh](../C8-Geometry/C8-Geometry.md)
**Ties:** [C3](../C3-Model-Stores/C3-Model-Stores.md), [C5](../C5-CWorld/C5-CWorld.md), [C6](../C6-Collision/C6-Collision.md), [C7](../C7-RenderWare-Stream/C7-RenderWare-Stream.md), [C8](../C8-Geometry/C8-Geometry.md), [C10](../C10-2dEffect/C10-2dEffect.md)
**RE status:** Documented
**Confidence:** ✅ Verified

---

## Deep-dive pages

- [C9.1 — The material list](01-the-material-list.md): count-prefixed indices, the 28-byte material,
  and the three surface floats.
- [C9.2 — Texture references](02-texture-references.md): how a material names a texture it does not
  contain.
- [C9.3 — TextureNative: the pixel payload](03-texturenative-payload.md): formats, dimensions, mip
  chains — the full census.

---

## 9.1 The chain, end to end

With this chapter the path from a streaming ID to a pixel is complete:

```
model ID  ─►  ms_modelInfoPtrs[id]           [C3.3]
              └─► RwObject (clump)           [C7.3]
                    └─► GeometryList         [C7.3]
                          └─► Geometry       [C8.1]
                                ├─► BinMeshPLG — index buffer   [C8.2]
                                └─► MaterialList                [C9.1]
                                      └─► Material (28 B)       [C9.1]
                                            └─► Texture         [C9.2]
                                                  └─ by NAME ──►  TXD  [C7.3]
                                                        └─► TextureNative  [C9.3]
                                                              └─► DXT blocks
```

The one link that is **not** a pointer is the important one: a material refers to its texture **by
name**, not by reference ([C9.2](02-texture-references.md)). That single design choice is why models
and textures are separate streaming ranges ([C3.1](../C3-Model-Stores/01-the-partition.md)), why one
TXD can serve hundreds of DFFs, and why a missing texture produces white geometry rather than a crash.

## 9.2 Verified counts

Sampled from `gta3.img`: 200 DFFs (258 geometries) and 60 TXDs.

| Measurement | Value |
|---|---:|
| Material lists whose struct is exactly `4 + 4 × n` | **258 / 258** |
| Materials | 1,139 |
| Materials whose struct is exactly 28 bytes | **1,139 / 1,139** |
| Textured materials | 1,124 (98.7 %) |
| Texture sections whose struct is exactly 4 bytes | **1,124 / 1,124** |
| `TextureNative` sections decoded (full population, all 3,974 TXDs) | **32,157** |

✅ Every structural claim in this chapter is an exact-size match on every sample, with no exceptions —
the same standard as [C8.1 §3](../C8-Geometry/01-the-geometry-struct.md).

## 9.3 The texture census

Censused over **every texture in the game** — 32,157 sections across all 3,974 TXDs, no sampling:

| Property | Result |
|---|---|
| **DXT1** | 28,807 (89.6 %) |
| **DXT3** | 2,098 (6.5 %) |
| `X8R8G8B8` uncompressed | 1,015 |
| `A8R8G8B8` uncompressed | 237 |
| 16-bit depth (compressed) | 30,905 |
| 32-bit depth (uncompressed) | 1,252 |
| Single mip level | 25,149 (78.2 %) |
| Multi-level mip chain | 7,008 (21.8 %) |

**96.1 % of San Andreas textures are DXT-compressed**, and **78.2 % ship with no mipmaps at all**. Both
matter for anyone replacing textures: an uncompressed replacement is 4–8× the streaming cost, and adding
a mip chain to a texture that shipped without one changes its footprint by a third.

The census also **closes** the `0x8000` raster-format bit — it correlates with `numLevels > 1` perfectly,
zero exceptions in 32,157 records — and **corrects** C7.3's extrapolated "~35,000 textures" to the true
32,157.

Full detail in [C9.3](03-texturenative-payload.md).

## 9.4 ⚠️ A prediction that failed

[C8.3 §4](../C8-Geometry/03-identifying-plugins-by-size.md) proposed that the unidentified geometry
plugin `0x253F2F8` would "likely regress against the material count rather than the vertex count."

**It does not.** With material counts now available, all 33 samples were tested against
`4 + k × numMaterials` for `k` ∈ {2, 4, 8, 12} and against the plugin's own first dword as a count.
**Zero clean fits.** The section's size divided by its first dword yields 32, 76, 100 and 108 in
different files — integers, but not a *constant* one, so it is not a simple count-prefixed array of
fixed records either.

✅ **Now resolved:** [C10](../C10-2dEffect/C10-2dEffect.md) identifies `0x253F2F8` as **`2dEffect`** — a
**variable-length** record array, which is exactly why no fixed-stride hypothesis could fit. The
disproof below was the step that ruled out the whole fixed-stride family and pointed at the answer.

The hypothesis is **disproven rather than quietly dropped**. Recording a failed prediction is the same discipline as
recording the repeat-sector error in
[C5.6 §4](../C5-CWorld/06-the-sector-arrays-closed.md): the method only stays trustworthy if its misses
are visible alongside its hits.

## 9.5 What remains

⏳ **Open after this chapter:**

- ~~`0x253F2F8`~~ ✅ closed in [C10](../C10-2dEffect/C10-2dEffect.md) — it is `2dEffect`. `0x116` remains open.
- The mip-chain byte layout within a `TextureNative` payload — dimensions and level counts are
  decoded for the whole population; the per-level offsets are not walked here.
- `0x116` — 3 occurrences, very large, non-vendor numbering.
- The five oversized `0x253F2FD` sections ([C8.3 §3](../C8-Geometry/03-identifying-plugins-by-size.md)).

---

### Key takeaways

- The chain **streaming ID → model info → clump → geometry → material → texture → pixels** is now
  complete end to end.
- A material names its texture **by name, not by pointer** — the design choice behind separate
  streaming ranges and shared TXDs.
- Structure verified exactly on every sample: **258/258** material lists, **1,139/1,139** materials at
  28 bytes, **1,124/1,124** texture structs at 4 bytes.
- **96.1 % of textures are DXT** (89.6 % DXT1), and **78.2 % ship with a single mip level** — full
  population, not a sample.
- ⚠️ C8.3's prediction that `0x253F2F8` tracks material count is **disproven** — and the elimination
  led directly to [C10](../C10-2dEffect/C10-2dEffect.md), where it is identified as `2dEffect`.
- `2dEffect` is decoded in [C10](../C10-2dEffect/C10-2dEffect.md).

**Next:** [C9.1 — The material list](01-the-material-list.md)

## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Loaded / consumed by:** [C3](../C3-Model-Stores/C3-Model-Stores.md), [C5](../C5-CWorld/C5-CWorld.md), [C6](../C6-Collision/C6-Collision.md), [C7](../C7-RenderWare-Stream/C7-RenderWare-Stream.md), [C8](../C8-Geometry/C8-Geometry.md), [C10](../C10-2dEffect/C10-2dEffect.md)
- **Known bugs / gotchas:** missing TXD parent (C48 txdcut) leaves untextured white models.
- **Modding:** TXD raster edits are the texture-mod path; DXT formats matter.
- **Performance:** raster upload cost at load; sampler state per draw.
