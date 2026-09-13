# C7.3 — Clumps and Texture Dictionaries

> **The one-sentence version:** a DFF is a clump of frames, geometries and atomics; a TXD is a count, a
> device tag and a run of native textures — and every texture in the game is tagged **D3D9**.

[← C7.2 — Version encoding](02-version-encoding.md) · [Chapter 7 hub](C7-RenderWare-Stream.md)

**Confidence:** ✅ Verified (structure, platform) / ⏳ (payloads)

---

## 1. The clump

Root type `0x10`. Its children, in the order they appear:

| Child | Type | Role |
|---|---|---|
| Struct | `0x01` | counts — atomics, lights, cameras |
| FrameList | `0x0E` | the transform hierarchy |
| GeometryList | `0x1A` | the meshes |
| Atomic × N | `0x14` | binds a frame to a geometry |
| Extension | `0x03` | plugin data |

✅ *Verified* over 60 sampled DFFs — all 60 have exactly this set, in this order.

The separation is the point: a **frame** is a transform, a **geometry** is a mesh, and an **atomic** is
the pairing of one with the other. A model with three moving parts has three frames, up to three
geometries, and three atomics.

Observed: **61 atomics across 60 clumps.** Almost every San Andreas model is a single atomic; multi-part
models (vehicles, doors, animated props) are the exception in the world set, though they are the rule in
`player.img` and vehicle models.

## 2. What lives in the extensions

The `Extension` sections are where San Andreas keeps everything RenderWare itself did not define.
Sampled at depth 3:

| Section | Occurrences in 60 files |
|---|---:|
| ~~`2dEffect`~~ **node-name plugin** (`0x253F2FE`) | **92** |
| `0x11E` | 34 |
| `HAnimPLG` (`0x1F`) | 1 |

> ⚠️ **Correction — see [C10.1 §3](../C10-2dEffect/01-the-record-and-corrections.md).** `0x253F2FE` was
> labelled `2dEffect` here. **It is the frame node-name plugin** — it lives in `FrameList/Extension` and
> its payload is plain ASCII (`waterjumpx2`, `pier69_models04`). The real `2dEffect` is **`0x253F2F8`**,
> in `Geometry/Extension` ([C10](../C10-2dEffect/C10-2dEffect.md)).
>
> The paragraph below drew the right conclusion — the map's lighting *does* hang off models — from the
> wrong section. It is kept, struck, because a right answer from wrong reasoning is a coincidence, not a
> finding.

~~**`2dEffect` is the important one.** 92 occurrences in 60 models means world geometry routinely carries
attached effect markers — lights, coronas, particle emitters, ped-attractor points. This is where the
map's lighting actually comes from: not a separate lighting file, but sections hanging off the models.~~

🟡 *Reasoned:* the low `HAnimPLG` count (1 in 60) is expected — hierarchical animation is for skinned
characters, and the sample was drawn from `gta3.img`, which is overwhelmingly static world geometry. A
sample from `player.img` would invert that ratio.

⏳ **Open:** `0x11E`, at 34 occurrences, is common enough to matter and was not identified.

## 3. The texture dictionary

Root type `0x16`. Structure:

```c
// TextureDictionary (0x16)
//   Struct (0x01):
struct TxdStruct {
    uint16_t textureCount;
    uint16_t deviceId;
};
//   then textureCount × TextureNative (0x15)
//   then Extension (0x03)
```

✅ *Verified.* Sampling 120 TXDs:

| Measurement | Value |
|---|---|
| Textures found | **1,070** |
| Average per dictionary | **8.9** |
| `deviceId` | **2** — on all 120 |
| `TextureNative` platform ID | **9** — on all 1,070 |

## 4. Everything is D3D9

Two independent platform tags, both consistent:

- **`deviceId = 2`** in the dictionary struct.
- **Platform ID `9`** in every `TextureNative` — RenderWare's `PLATFORM_D3D9`.

✅ *Verified* across 1,070 textures.

This is a third confirmation of the D3D9 pipeline, after the `world/pipe/p2/d3d9/wrldpipe.c` source
path in [C0.3 §3.1](../C0-Binary-Identity/03-external-analysis-corroboration-and-conflicts.md) and the
`g_D3DDevice` machinery the renderer is built on. The textures are not platform-neutral: they ship
pre-swizzled for D3D9, which is why console asset sets are not interchangeable with the PC ones.

**Practical consequence:** a texture inserted with a platform ID other than 9 will load and then fail at
bind time, not at parse time — a class of "my mod is invisible" bug that a parse-time check would catch.

## 5. Sizing

At 8.9 textures per dictionary across 3,974 dictionaries, the game holds roughly **35,000 textures**.

⚠️ **Corrected.** A full-population census in
[C9.3 §1](../C9-Materials-And-Textures/03-texturenative-payload.md) — every TXD, no sampling — gives
**32,157 textures**, so this extrapolation was about **10 % high**. That is the error a 🟡 estimate
deserves, and it is why the later chapter counted all of them rather than scaling a sample.

Against 15,325 models this is about 2.3 textures per model, which is consistent with San Andreas's
approach of heavy texture reuse across world geometry — the same wall and road textures appearing in
hundreds of models, held once in a shared dictionary.

That reuse is exactly why textures are a **separate streaming range** with its own IDs
([C3.1](../C3-Model-Stores/01-the-partition.md)) rather than being embedded per model: one TXD serves
many DFFs, so they must be requested and evicted independently.

## 6. What is not decoded

⏳ **Open, and substantial:**

- **Geometry payload** — the `Geometry` struct's vertex/triangle format, morph targets, the binary mesh
  plugin that supplies the actual index buffers.
- **Material list** — texture references, colours, the surface properties that feed the renderer.
- **`TextureNative` payload** — pixel formats, DXT compression, mip chains, the raster flags.
- **Frame list contents** — the matrices and the parent indices that form the hierarchy.
- ~~**`2dEffect` records**~~ — now decoded in [C10](../C10-2dEffect/C10-2dEffect.md).

This chapter establishes that these exist, where they sit in the tree, and how to reach them. Decoding
them is a chapter of its own — realistically two, since geometry and texture payloads have little in
common beyond their container.

---

### Key takeaways

- A **clump** is `Struct`, `FrameList`, `GeometryList`, `Atomic`×N, `Extension` — verified in all 60
  sampled models, in that order.
- **Frame = transform, Geometry = mesh, Atomic = the pairing.** 61 atomics across 60 clumps: world
  models are overwhelmingly single-atomic.
- ⚠️ **`0x253F2FE` is the node-name plugin, not `2dEffect`** — corrected in
  [C10.1 §3](../C10-2dEffect/01-the-record-and-corrections.md). The real `2dEffect` is `0x253F2F8`.
- A **TXD** is `{u16 count, u16 deviceId}` then `count` × `TextureNative`; the true game-wide total is
  **32,157** ([C9.3 §1](../C9-Materials-And-Textures/03-texturenative-payload.md)) — this page's
  sample-extrapolated "~35,000" was 10 % high.
- **`deviceId = 2` and platform ID `9` — D3D9 — on every sample**, a third independent confirmation of
  the D3D9 pipeline.
- Texture reuse across models is why TXDs occupy their **own streaming range** rather than being
  embedded.
- Geometry, material, pixel and `2dEffect` payloads are **all still open** — located, not decoded.

**Continue:** [Chapter 7 hub](C7-RenderWare-Stream.md) · next chapter: `C8 — Geometry & the Binary Mesh`
