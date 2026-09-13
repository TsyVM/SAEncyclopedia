# Chapter 6 — Collision & the COL Model

> **Goal of this chapter:** decode the collision asset pipeline end to end — the versioned container,
> the header that binds a collision model to a game model, and the 255-slot store the streaming
> partition reserved for it.

**Subsystem category:** World / physics data
**Depends on:** [C3 — The Model Stores](../C3-Model-Stores/C3-Model-Stores.md),
[C4 — Entities & Pools](../C4-Entities-And-Pools/C4-Entities-And-Pools.md)
**Ties:** [X1](../X1-SDK-Cross-Reference/X1-SDK-Cross-Reference.md), [C1](../C1-Streaming/C1-Streaming.md), [C3](../C3-Model-Stores/C3-Model-Stores.md), [C4](../C4-Entities-And-Pools/C4-Entities-And-Pools.md), [C31](../C31-Streaming-Slot-Tables/C31-Streaming-Slot-Tables.md), [C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md), [C45](../C45-Damage/C45-Damage.md), [C47](../C47-Vehicle-Dynamics/C47-Vehicle-Dynamics.md)
**RE status:** Documented
**Confidence:** ✅ Verified

---

## Deep-dive pages

- [C6.1 — The COL container & version dispatch](01-col-container-and-versions.md): four versions, the
  loader's own comparison chain, and the one that ships unused.
- [C6.2 — The header, the bounds, and a broken file](02-header-and-bounds.md): both bounds layouts
  verified by arithmetic, plus a genuine off-by-one in retail data.
- [C6.3 — CColStore & the model binding](03-colstore-and-binding.md): 251 archives in 255 slots, and
  the name-hash check that guards the binding.
- [C6.4 — Collision detection and physics integration](04-collision-detection-and-physics-integration.md):
  broad-phase sector query to narrow-phase triangle tests; how collision contacts become impulses in
  `CPhysical`; the COL–surface link to [C24](../C24-Surfaces/C24-Surfaces.md); damage from collision.

---

## 6.1 Two levels, not one

Collision has a structure the other stores do not: a **streamed archive** contains **many collision
models**.

```
levelmap_1.col                  ← one streaming slot (one COL-store ID)
 ├── lib_street09  (COL2)       ← one collision model, bound to model 3890
 ├── lib_street08  (COL2)       ← bound to model 3891
 └── … 38 more on average
```

So the 255 COL IDs from [C3.1](../C3-Model-Stores/01-the-partition.md) are not 255 collision *models* —
they are 255 **archives**, and the retail game ships **10,155 collision models** inside 251 of them.

That is the chapter's organising fact, and it explains why the COL range is the smallest of the large
ranges: it is counting containers, not contents.

## 6.2 The retail inventory

| Source | Archives | Collision models |
|---|---:|---:|
| `models/gta3.img` | 216 | 8,255 |
| `models/gta_int.img` | 35 | 1,900 |
| **Total streamed** | **251** | **10,155** |
| `models/coll/*.col` (loose, not streamed) | 3 | 36 |

✅ *Verified* by walking every archive. **251 archives against 255 slots — four spare.** The partition
in [C3.1](../C3-Model-Stores/01-the-partition.md) was sized against the shipped content with almost no
headroom, which is both a nice independent confirmation of that boundary and a warning to anyone adding
map content.

By version:

| Version | Models | Where |
|---|---:|---|
| `COLL` (v1) | 36 | only the three loose files |
| `COL2` (v2) | 885 | streamed |
| `COL3` (v3) | 9,270 | streamed |
| `COL4` (v4) | **0** | — supported by the loader, unused by the data |

> ⚠️ **The loose-file count was wrong in the first draft of this chapter**, which said 14. That number
> came from a strict `off += 8 + fileSize` walk — the exact reader bug documented in
> [C6.2 §4](02-header-and-bounds.md), hit while writing the page that documents it. `peds.col`
> desynchronises after its eighth chunk, so a strict walker reports 8 of 30 models and raises no error.
> Corrected by magic scan: **30 + 1 + 5 = 36**. The streamed figures were unaffected — all 251 archives
> walk cleanly.

## 6.3 The binding

A collision model names the game model it belongs to, twice — by **ID** and by **name** — and the
loader checks them against each other:

```
005384f4: cmp  eax, 0x4e20                     ; modelId < 20000?
005384ff: mov  ebx, dword ptr [eax*4 + 0xa9b0c8] ; ms_modelInfoPtrs[modelId]
0053850a: mov  esi, dword ptr [ebx + 4]        ; modelInfo->nameHash
00538512: call 0x53cf30                        ; hash(the COL header's name)
0053851a: cmp  esi, eax
0053851c: je   0x538569                        ; match -> bind
```

✅ *Verified.* The ID is the fast path and the name hash is the guard — if the ID has shifted (because
`.ide` files were reordered, which mods do constantly) the hash fails and the loader falls back to a
search rather than silently attaching collision to the wrong object.

This is also the third consumer of `ms_modelInfoPtrs` at `0x00A9B0C8` documented so far, after
[C3.3](../C3-Model-Stores/03-model-info-polymorphism.md) and
[C4.3](../C4-Entities-And-Pools/03-entity-model-link.md).

## 6.4 Dependencies

```
COL archive (streamed)                       [C1, C2]
  └── COL model
        ├── binds to CBaseModelInfo via ms_modelInfoPtrs   [C3.3]
        ├── allocated from the ColModel pool (10,150 × 48 B) [C4.5]
        └── consumed by CWorld sector queries               [C5]
```

The `ColModel` pool capacity is **10,150** ([C4.5](../C4-Entities-And-Pools/05-the-cpool-object.md))
against **10,155** collision models in the shipped data — a five-slot deficit that is only survivable
because not every archive is resident at once. That near-equality is a strong hint the pool was sized
directly against the content census, and it is the tightest limit relationship found anywhere in this
encyclopedia so far.

## 6.5 What this chapter does not cover

⏳ **Open:** the collision *geometry* — spheres, boxes, face groups, the triangle soup, and the shadow
mesh that COL3 adds. This chapter decodes the container, the header, the version dispatch and the
binding; the payload past the bounds is the next pass.

That split is deliberate and matches [C1](../C1-Streaming/C1-Streaming.md)'s: the container and the
addressing are verifiable quickly and are what the SDK needs first; the geometry is a larger job with
its own verification demands.

---

### Key takeaways

- Collision is **two-level**: 255 streaming slots hold **archives**, and the archives hold **10,155
  collision models** (plus 36 in three non-streamed loose files).
- Retail ships **251 archives — four short of the limit**, independently confirming C3.1's boundary.
- Version census: **COL3 9,270, COL2 885, COLL 36, COL4 zero** — the loader supports a version the data
  never uses.
- The COLL figure was **initially wrong (14)** because a strict `fileSize` walk silently truncates
  `peds.col` — this chapter's own documented bug, hit while documenting it.
- Binding is **by model ID with a name-hash guard**, so reordered `.ide` files degrade to a search
  rather than mis-attaching collision.
- The `ColModel` pool (**10,150**) is five slots smaller than the shipped model count — the tightest
  limit relationship in the encyclopedia, and only viable because archives stream.
- Geometry decoding is **explicitly deferred**; this chapter is container, header, dispatch and binding.

**Next:** [C6.1 — The COL container & version dispatch](01-col-container-and-versions.md)

## See also (forward links)

Collision contacts become physics impulses in [C42 — Vehicle Physics](../C42-Vehicle-Physics/C42-Vehicle-Physics.md) and [C47 — Vehicle Dynamics](../C47-Vehicle-Dynamics/C47-Vehicle-Dynamics.md), and the impact magnitude feeds [C45 — Damage](../C45-Damage/C45-Damage.md). The CColStore slot table is re-derived in [C31.1](../C31-Streaming-Slot-Tables/01-ccolstore.md).

## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Key functions:** `CColStore::RemoveCol` (0x410730), `CIplStore::RequestIplAndIgnore` (0x405850)
- **Callers:** **4** `.text` call-sites reach this chapter's functions.
- **Callees:** **6** distinct functions called from within them.
- **Known bugs / gotchas:** bad COL bounds cause fall-through-world; the CColStore 44-byte slot must match.
- **Modding:** COL files are heavily modded; the CColStore pool @0x965560 is the hook.
- **Performance:** broadphase + narrowphase per contact; bounding volumes cheap.
