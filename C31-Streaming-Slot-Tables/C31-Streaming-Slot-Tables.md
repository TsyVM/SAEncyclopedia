# Chapter 31 — Streaming Slot Tables: Collision, IPL and Streamed Scripts

> **Goal of this chapter:** the streamer ([C1](../C1-Streaming/C1-Streaming.md)/[C2](../C2-CStreaming/C2-CStreaming.md))
> fans out to a family of per-asset-type *slot tables*, one kind of record per resource type. This chapter
> reads three of them out of `gta_sa.exe`: `CColStore`'s **44-byte** collision slots, `CIplStore`'s
> **52-byte** IPL slots, and `CStreamedScripts`' fixed **82 × 32-byte** table. The first independently
> **re-confirms** [C6.3](../C6-Collision/03-colstore-and-binding.md)'s CColStore finding from a different
> set of methods; the other two are new.

**Subsystem category:** Streaming — per-type slot tables
**Depends on:** [C27](../C27-Function-Catalogue/C27-Function-Catalogue.md) (names + addresses),
[C28](../C28-Class-Catalogue/C28-Class-Catalogue.md) (disassembly method), [C1](../C1-Streaming/C1-Streaming.md)
/ [C3](../C3-Model-Stores/C3-Model-Stores.md) (the streaming architecture and stores)
· cross-checks [C6.3](../C6-Collision/03-colstore-and-binding.md), [C18](../C18-SCM-Script/C18-SCM-Script.md)
**Ties:** [C1](../C1-Streaming/C1-Streaming.md), [C2](../C2-CStreaming/C2-CStreaming.md), [C3](../C3-Model-Stores/C3-Model-Stores.md), [C6](../C6-Collision/C6-Collision.md), [C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md), [C18](../C18-SCM-Script/C18-SCM-Script.md), [C27](../C27-Function-Catalogue/C27-Function-Catalogue.md), [C28](../C28-Class-Catalogue/C28-Class-Catalogue.md)
**RE status:** Documented
**Confidence:** ✅ for every stride / base / offset below (re-checked by `derive_streaming_stores.py`) ·
🔷 for methods listed but not individually disassembled
**Data artifact:** [`RE-Data/data/streaming_stores.json`](../RE-Data/data/streaming_stores.json) — generated
by [`tools/derive_streaming_stores.py`](../tools/derive_streaming_stores.py)

---

## Deep-dive pages

- [C31.1 — CColStore, re-confirmed and located](01-ccolstore.md): the **44-byte** collision slot — C6.3's
  finding reproduced from `RemoveCol`/`RemoveColSlot`/`IncludeModelIndex` — plus the `CPool` object's
  address (`0x965560`) and its array/flag/count layout, and the per-slot status byte.
- [C31.2 — CIplStore and the 52-byte slot](02-ciplstore.md): the IPL store, a sibling `CPool` at
  `0x8E3FB0` with a **52-byte** slot, sized four ways — the streaming table behind [C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md)'s
  map placements.
- [C31.3 — CStreamedScripts and the 82-slot table](03-cstreamedscripts.md): the fixed embedded table of
  **82 × 32-byte** slots (count word at object +0xA44), the streamed/external mission scripts behind
  [C18](../C18-SCM-Script/C18-SCM-Script.md).

---

## 31.0 The result first

| Claim | Evidence |
|---|---|
| **`CColStore` slot = 44 B (`0x2C`)** in a `CPool` at `0x965560` | `imul …,0x2C` in 3 methods — **re-confirms** [C6.3](../C6-Collision/03-colstore-and-binding.md) |
| `CColStoreSlot` fields: bounding rect @+0, filename @+16, model-range @+34/+36, refCount @+38, status bytes @+40–+43 | RE-Data sweep ✅ |
| **`CIplStore` slot = 52 B (`0x34`)** in a `CPool` at `0x8E3FB0` | `imul …,0x34` in 4 methods + ÷52 reciprocal ✅ |
| `CIplStoreSlot` fields: bounding rect @+0, filename @+16, entity ranges @+34–+41, staticIdx @+42, 6 status bytes @+44–+49 | RE-Data sweep ✅ |
| Both stores share the `CPool` layout: **array ptr @+0, flag bytes @+4, count @+8** | `RemoveCol` / `RequestIplAndIgnore` ✅ |
| **`CStreamedScripts` = 82 slots (`0x52`) × 32 B (`0x20`)**, records from +8, count word @ object+0xA44 | `ReInitialise` + `RegisterScript` ✅ |
| **9 / 9** structural facts re-verified against `gta_sa.exe` on every run | `derive_streaming_stores.py` |

---

## Why these three together

[C1 §](../C1-Streaming/C1-Streaming.md) sketches the streamer fanning out to `CModelInfo`, `CTxdStore`,
`CColStore` and `CIplStore` — a family of per-type stores, each a table of same-shaped slots the streamer
indexes by a flat id. C28.3 recovered the streaming-info array that sits at the top of that fan-out; this
chapter recovers three of the leaves.

Two of them (`CColStore`, `CIplStore`) are **`CPool`-based**: the same three-field pool object C30.1's
entry/exit markers used — a pointer to the packed slot array, a parallel array of one status byte per slot,
and a count — instantiated once per resource type with a different slot width (44 vs 52 bytes). Their slot
counts are therefore *dynamic* (whatever the loaded map needs), not fixed arrays. The third,
`CStreamedScripts`, is the contrast: a **fixed 82-slot table embedded directly in its manager object**,
sized at compile time.

`CColStore` is a deliberate re-derivation. [C6.3](../C6-Collision/03-colstore-and-binding.md) already found
its 44-byte `CPool` record while chasing the collision-to-model binding; this chapter reaches the identical
44 from three unrelated methods (`RemoveCol`, `RemoveColSlot`, `IncludeModelIndex`) and adds the pool's
address and slot layout — the same "confirm an existing result from the other end" move C28.3 made against
C2. The two independent routes to 44 are the confirmation.

**Next:** [C31.1 — CColStore, re-confirmed and located](01-ccolstore.md).

## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Key functions:** `CColStore::RemoveCol` (0x410730), `CColStore::RemoveColSlot` (0x411330), `CIplStore::RequestIplAndIgnore` (0x405850)
- **Callers:** **5** `.text` call-sites reach this chapter's functions.
- **Callees:** **15** distinct functions called from within them.
- **Known bugs / gotchas:** CColStore 44-byte slot re-confirms C6.3; slot exhaustion drops loads.
- **Modding:** the CPool template (Col/Ipl/streamed-script) is the streaming-hook base.
- **Performance:** O(1) slot indexing; flat pools.
