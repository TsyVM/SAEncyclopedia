# Chapter 11 — IDE & IPL: the World's Data Model

> **Goal of this chapter:** decode the two text-and-binary formats that turn 15,325 anonymous models
> into a world — what each object *is*, and where each copy of it *stands* — and confirm the sector
> grid from C5 against 36,569 real placements.

**Subsystem category:** World / data model
**Depends on:** [C3 — The Model Stores](../C3-Model-Stores/C3-Model-Stores.md),
[C5 — CWorld & Spatial Partitioning](../C5-CWorld/C5-CWorld.md),
[C10 — 2dEffect](../C10-2dEffect/C10-2dEffect.md)
**Ties:** [X1](../X1-SDK-Cross-Reference/X1-SDK-Cross-Reference.md), [C1](../C1-Streaming/C1-Streaming.md), [C2](../C2-CStreaming/C2-CStreaming.md), [C3](../C3-Model-Stores/C3-Model-Stores.md), [C5](../C5-CWorld/C5-CWorld.md), [C6](../C6-Collision/C6-Collision.md), [C9](../C9-Materials-And-Textures/C9-Materials-And-Textures.md), [C10](../C10-2dEffect/C10-2dEffect.md)
**RE status:** Verified
**Confidence:** ✅ Verified over the full data set

---

## Deep-dive pages

- [C11.1 — IDE: the definition side](01-ide-definitions.md): nine section types, the ID space, and
  seven duplicate IDs in retail data.
- [C11.2 — IPL: the placement side](02-ipl-placements.md): text and binary, 45,884 placements, and the
  LOD link.
- [C11.3 — What the placements prove](03-what-placements-prove.md): the world extent, and a
  correction to C5.4's pool arithmetic.
- [C11.4 — Loader and runtime integration](04-loader-and-runtime-integration.md): `CFileLoader::LoadLevel`
  orchestration; `LoadObjectTypes` into `CModelInfo`; `LoadMapData` into `CWorld`; binary IPL fast path;
  LOD chain setup; full map-mod workflow (gta.dat, default.dat, archive slot).

---

## 11.1 Two files, two questions

```
.ide  — DEFINITION    "model 1337 is called `sw_barn02`, uses TXD `sw_farm`,
                       draws to 299 units, flags 0"
   │
   │  model ID
   ▼
.ipl  — PLACEMENT     "put model 1337 at (612.5, -1102.3, 78.4), rotation q,
                       interior 0, LOD -1"
```

An IDE row *creates* a model ID; an IPL row *uses* one. Everything the earlier chapters decoded —
geometry, materials, textures, collision, `2dEffect` lights — is reached through the ID that an IDE row
assigns, and positioned by the IPL rows that reference it.

This is also where [C10.3 §5](../C10-2dEffect/03-the-light-record.md)'s loose end ties off: a light's
position is model-local, and it is the IPL placement that turns it into a world position. One 80-byte
light record on a model placed twenty times produces twenty lighting rigs.

## 11.2 The full inventory

**Definitions** — 59 `.ide` files:

| Section | Rows |
|---|---:|
| `objs` | 14,052 |
| `peds` | 276 |
| `cars` | 212 |
| `tobj` | 160 |
| `2dfx` | 97 |
| `anim` | 54 |
| `weap` | 50 |
| `txdp` | 38 |
| `hier` | 35 |

**Placements** — 53 text `.ipl` files plus 164 binary IPLs streamed from `gta3.img`:

| Source | `inst` placements |
|---|---:|
| Text `.ipl` | 9,315 |
| **Binary IPL (streamed)** | **36,569** |
| **Total** | **45,884** |

✅ *Verified.* The binary IPLs carry **80 %** of the world. The text files are the parts that must exist
before streaming starts; the streamed sections are the world proper.

Other IPL sections, text only: `path` 165,152 · `cull` 1,266 · `occl` 1,012 · `enex` 376 · `auzo` 155 ·
`grge` 52 · `tcyc` 8 · `pick` 5.

`path` at 165,152 rows — **93.1 % of all text IPL rows** — dwarfs everything. The AI path network is by
far the largest hand-authored data set in the game, an order of magnitude bigger than the object
placements it runs through.

## 11.3 The headline result: the world fits the grid exactly

Every one of the 36,569 binary placements was checked against the sector grid derived in
[C5.2](../C5-CWorld/02-sector-index-arithmetic.md) — 120 × 120 cells of 50 units spanning −3000 … +3000:

| Measurement | Value |
|---|---|
| X extent | **−2993.8 … 2955.5** |
| Y extent | **−2937.5 … 2956.4** |
| Z extent | −76.1 … 1382.2 |
| **Placements outside the grid** | **0 (0.00 %)** |

✅ Not one placement falls outside the bounds that the `× 0.02 + 60` arithmetic implies — and the
extremes come within **7 units** of the boundary on one axis.

That is the strongest possible confirmation of C5.2. The grid constants were read from three floating
point instructions; the content was authored by a different team through a different tool chain; and
they agree to within 0.2 % of the world's width. **The constants define the world, and the world was
built to fill them.**

## 11.4 Two anomalies in retail data

**Seven duplicate object IDs.** IDs `16700`–`16708` are each defined twice — once in `countn2.ide` as
LOD rocks (`lod_rockgp1_12`), once in `leveldes.ide` as level-design props (`androm_des_obj`).
✅ Verified. Details in [C11.1 §4](01-ide-definitions.md).

**Twenty-one placed-but-undefined IDs.** Text IPLs place 21 model IDs that no `objs`/`tobj`/`anim` row
defines. ✅ Verified; the likely explanation is in [C11.1 §5](01-ide-definitions.md).

Both are the kind of thing a data model tolerates and a naive tool does not.

## 11.5 Dependencies

```
.ide  ──assigns──►  model ID  ──►  CBaseModelInfo    [C3.3]
                                     ├── DFF / TXD names   [C7]
                                     ├── collision by ID + name hash  [C6.3]
                                     └── 2dEffect lights   [C10]
.ipl  ──places──►   CEntity (m_nModelIndex +0x22, m_nIplIndex +0x2E)   [C4.3, X1 §3.6]
                       └── linked into CWorld sectors   [C5]
```

`CEntity::m_nIplIndex` at `+0x2E` ([X1 §3.6](../X1-SDK-Cross-Reference/X1-SDK-Cross-Reference.md)) is
the back-link: every placed entity remembers which IPL created it, which is how a streamed map section
can remove exactly its own objects.

## 11.6 What remains

⏳ **Open:** the `path` section's 165,152 rows — the AI node network. It is the largest data set in the
game and deserves its own chapter rather than a paragraph here. Also not decoded: `cull`, `occl`,
`enex`, `auzo`, `grge`, `tcyc` record layouts, and the binary IPL header fields beyond `numInst` and
`instOffset`.

---

### Key takeaways

- **IDE defines, IPL places.** Everything decoded in C6–C10 hangs off the model ID an IDE row assigns.
- **45,884 placements** total — **36,569 (80 %) in the 164 streamed binary IPLs**, 9,315 in text.
- **`path` at 165,152 rows (93.1 % of text IPL)** is the largest authored data set in the game.
- ✅ **Zero of 36,569 placements fall outside the −3000 … +3000 sector grid**, with extremes within 7
  units of the boundary — independent confirmation of [C5.2](../C5-CWorld/02-sector-index-arithmetic.md)
  from data authored by a different pipeline.
- Two retail data anomalies: **7 duplicate object IDs** and **21 placed-but-undefined IDs**.
- `m_nIplIndex` is the back-link that lets a streamed section remove exactly its own entities.

**Next:** [C11.1 — IDE: the definition side](01-ide-definitions.md)

## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Loaded / consumed by:** [C1](../C1-Streaming/C1-Streaming.md), [C2](../C2-CStreaming/C2-CStreaming.md), [C3](../C3-Model-Stores/C3-Model-Stores.md), [C5](../C5-CWorld/C5-CWorld.md), [C6](../C6-Collision/C6-Collision.md), [C9](../C9-Materials-And-Textures/C9-Materials-And-Textures.md)
- **Known bugs / gotchas:** malformed IDE/IPL lines are skipped silently (a real modding footgun).
- **Modding:** IDE/IPL are THE map-mod files; default.dat/gta.dat drive load order (C48).
- **Performance:** parsed at load; builds the world db.
