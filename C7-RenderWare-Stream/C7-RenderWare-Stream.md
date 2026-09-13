# Chapter 7 — RenderWare: the Stream Format

> **Goal of this chapter:** decode the container that holds 95.7 % of the game's streamed assets — a
> single recursive section format shared by every model and texture file, stamped with a version that
> independently confirms the engine identification from C0.3.

**Subsystem category:** Rendering / asset format
**Depends on:** [C1.1 — The IMG VER2 archive model](../C1-Streaming/01-img-ver2-archive-model.md),
[C3 — The Model Stores](../C3-Model-Stores/C3-Model-Stores.md)
**Ties:** [C0](../C0-Binary-Identity/C0-Binary-Identity.md), [C1](../C1-Streaming/C1-Streaming.md), [C3](../C3-Model-Stores/C3-Model-Stores.md), [C5](../C5-CWorld/C5-CWorld.md), [C6](../C6-Collision/C6-Collision.md), [C9](../C9-Materials-And-Textures/C9-Materials-And-Textures.md), [C10](../C10-2dEffect/C10-2dEffect.md)
**RE status:** Documented
**Confidence:** ✅ Verified

---

## Deep-dive pages

- [C7.1 — The section stream](01-the-section-stream.md): the 12-byte header, recursion, and the eleven
  files with two roots.
- [C7.2 — Version encoding & the RW 3.6 confirmation](02-version-encoding.md): unpacking the library
  ID, and a second independent proof of the engine version.
- [C7.3 — Clumps and texture dictionaries](03-clumps-and-txd.md): what a DFF and a TXD actually
  contain.
- [C7.4 — Modding DFF and TXD files](04-modding-dff-and-txd.md): name-matching, TXD binding, and
  COL requirements for vehicle mods; mip-map chains; plugin preservation table; common mod errors
  and their RW-stream root causes.

---

## 7.1 One format, two extensions

`.dff` and `.txd` are not two formats. They are the same **RenderWare binary stream** with different
root sections:

| Extension | Root section | Count (all archives) |
|---|---|---:|
| `.dff` | `0x10` Clump | 15,325 |
| `.txd` | `0x16` TextureDictionary | 3,974 |
| | | **19,299 total** |

✅ *Verified* across `gta3.img`, `gta_int.img`, `player.img` and `cutscene.img`.

Everything a reader needs — nesting, sizes, versioning — is handled by one recursive section mechanism
([C7.1](01-the-section-stream.md)). The difference between a model and a texture dictionary is which
type ID sits at offset 0.

## 7.2 The version is uniform, and it confirms C0.3

Every RenderWare section carries a **library ID** in its header. Reading it from all 19,299 files:

| Library ID | Files | Decoded |
|---|---:|---|
| `0x1803FFFF` | **19,299 — all of them** | RenderWare **3.6.0.0**, build `0xFFFF` |

✅ *Verified.* Not a single asset in the game deviates.

This matters beyond bookkeeping. [C0.3 §3.1](../C0-Binary-Identity/03-external-analysis-corroboration-and-conflicts.md)
established the engine as **RenderWare 3.6** from 90 `$Id:` source-path strings in `.rdata` — 77 of them
tagged `RW36Active`. That was a fact about the *executable*.

This chapter reaches the same version from the *data*, by a completely unrelated route: a bit-packed
integer in every asset header. **Two independent sources, one answer.** That is the strongest form of
confirmation available without source code, and it retires any doubt about the renderer's identity.

## 7.3 Why this chapter matters most for tooling

[C1.1 §3](../C1-Streaming/01-img-ver2-archive-model.md) observed that `.dff` and `.txd` are 95.7 % of
all IMG entries. This is the format those entries are in, so it is the format every practical GTA SA
tool must read.

It is also the format where [C1.1 §4.1](../C1-Streaming/01-img-ver2-archive-model.md)'s warning bites:
the IMG directory gives *reserved sectors*, not file length. The real length of a RenderWare file is
`12 + rootSection.size`, read from the payload itself.

That check was run against every model and texture in `gta3.img`:

| Result | Count |
|---|---:|
| Root section fits inside its reserved slot | **15,694 / 15,705** |
| Exceptions | 11 |

✅ *Verified* — and the 11 exceptions are not corruption. They are files with **two root sections**
([C7.1 §3](01-the-section-stream.md)), where a UV-animation dictionary precedes the clump. A reader
that assumes one root silently drops the model in all eleven.

## 7.4 Dependencies

```
RenderWare stream                      [C7]
  ├── delivered by:  IMG archive       [C1.1]  -> CdStream -> CStreaming  [C1.2, C2]
  ├── addressed by:  model / TXD ID    [C3.1]
  ├── models bound to: CBaseModelInfo  [C3.3]  via ms_modelInfoPtrs
  └── platform:      D3D9              [C7.3]
```

## 7.5 Scope

⏳ **Open — deliberately.** This chapter decodes the **container**: section headers, nesting, versioning,
root layouts, and the top two levels of clump and dictionary structure. It does **not** decode:

- geometry payloads — vertices, triangles, morph targets, the binary mesh plugin;
- texture payloads — the D3D9 pixel formats, mip chains, DXT blocks;
- the plugin/extension sections in detail (`2dEffect`, `HAnimPLG`, skin data).

Same split as [C1](../C1-Streaming/C1-Streaming.md) and [C6](../C6-Collision/C6-Collision.md): the
container and addressing first, because they are quickly verifiable and are what the SDK needs to
locate anything at all.

---

### Key takeaways

- `.dff` and `.txd` are **one format** — the RenderWare section stream — distinguished only by root
  type (`0x10` Clump vs `0x16` TextureDictionary).
- **19,299 assets, every one stamped `0x1803FFFF` = RenderWare 3.6.0.0.** No deviation.
- This **independently confirms C0.3's RW 3.6 identification**, which came from executable strings —
  two unrelated sources agreeing.
- Real file length is **`12 + rootSection.size`**, never the IMG directory's reserved sectors.
- **11 files have two root sections**; a single-root reader drops them silently.
- Geometry and texture payloads are explicitly **out of scope** for this pass.

**Next:** [C7.1 — The section stream](01-the-section-stream.md)

## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Loaded / consumed by:** [C0](../C0-Binary-Identity/C0-Binary-Identity.md), [C1](../C1-Streaming/C1-Streaming.md), [C3](../C3-Model-Stores/C3-Model-Stores.md), [C5](../C5-CWorld/C5-CWorld.md), [C6](../C6-Collision/C6-Collision.md), [C9](../C9-Materials-And-Textures/C9-Materials-And-Textures.md)
- **Known bugs / gotchas:** truncated RW streams crash the reader; version tags must match RW 3.6.
- **Modding:** DFF/TXD are the RenderWare containers every model mod ships.
- **Performance:** parse-once at load; streaming amortises it.
