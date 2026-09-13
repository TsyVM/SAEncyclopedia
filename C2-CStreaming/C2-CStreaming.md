# Chapter 2 — CStreaming & the Model-ID Space

> **Goal of this chapter:** decode the layer directly above `CdStream` — the flat ID space that unifies
> every streamable asset type, the 20-byte record that tracks each one, and the memory budget that
> governs eviction.

**Subsystem category:** Streaming
**Depends on:** [C1 — Streaming: the IMG Model & the CdStream Layer](../C1-Streaming/C1-Streaming.md)
**Ties:** [C0](../C0-Binary-Identity/C0-Binary-Identity.md), [X1](../X1-SDK-Cross-Reference/X1-SDK-Cross-Reference.md), [C1](../C1-Streaming/C1-Streaming.md), [C3](../C3-Model-Stores/C3-Model-Stores.md), [C4](../C4-Entities-And-Pools/C4-Entities-And-Pools.md)
**RE status:** Documented
**Confidence:** ✅ Verified

---

## Deep-dive pages

- [C2.1 — The flat ID space](01-the-id-space.md): 26,316 IDs, and how an array's capacity gets proved.
- [C2.2 — The streaming-info record](02-streaming-info-record.md): the 20 bytes, field by field, with
  the inferred three kept separate from the verified four.
- [C2.3 — The memory budget & `stream.ini`](03-memory-budget-and-stream-ini.md): the parser, the four
  keys, and the `<< 10`.
- [C2.4 — The request path](04-the-request-path.md): `RequestModel`, the priority guard, and
  touch-on-request.

---

## 2.1 One ID space for everything

`CdStream` moves sectors. `CStreaming` answers "which sectors, and is that thing loaded yet?" — and it
does so through a single flat integer space that covers models, textures, collision, map sections,
animations and scripts alike.

**26,316 IDs.** Verified structurally, not counted from a table: the streaming-info array begins at
`0x008E4CC0` with a stride of 20 bytes, and

```
0x008E4CC0 + 26316 × 20 = 0x009654B0
```

which is exactly where the next referenced global sits (19 xrefs). The array ends where its neighbour
begins — the same technique that pinned both `CdStream` tables in
[C1.2 §2](../C1-Streaming/02-cdstream-layer.md), and the same reason it is ✅ rather than 🟡.

## 2.2 The streaming-info record

```c
struct CStreamingInfo {     // 0x14 = 20 bytes
    uint16_t nextIndex;     // +0x00  intrusive list forward link
    uint16_t prevIndex;     // +0x02  intrusive list back link
    uint16_t nextIndexOnCd; // +0x04  🟡 load-order chain
    uint8_t  flags;         // +0x06  ✅ request flags
    uint8_t  imgId;         // +0x07  🟡 which archive
    uint32_t cdPosn;        // +0x08  🟡 sector offset (packed handle, C1.2 §4)
    uint32_t cdSize;        // +0x0C  ✅ size in sectors
    uint8_t  loadState;     // +0x10  ✅
};
```

Evidence for the ✅ fields:

| Field | Instruction |
|---|---|
| stride 20 | `lea edi,[ebp+ebp*4]; shl edi,2` — index × 5 × 4 |
| `+0x06` flags | `mov byte [esi+6], dl` after `or dl, bl` — request flags are OR-ed in |
| `+0x0C` cdSize | `mov ecx,[edi+0x8E4CCC]; neg ecx; shl ecx,0xB` — × 2048, subtracted from memory used |
| `+0x10` loadState | `mov al,[edi+0x8E4CD0]; cmp al,1` / `cmp al,2` |

⏳ **Open:** `+0x04`, `+0x07` and `+0x08` are marked 🟡 — they are the conventional reading and are
consistent with the 20-byte total, but this pass did not read an instruction that proves each one.
They are stated as inference, not promoted.

### Load states

| Value | Meaning | Evidence |
|---|---|---|
| `0` | not loaded | `test al,al; je` early-out |
| `1` | loaded | `cmp al,1` gates the unload path |
| `2` | requested / in flight | `cmp al,2` gates the request-count decrement |

## 2.3 The memory budget

The streaming budget is **read from `stream.ini` and multiplied by 1024**:

```
005bcd45: call  0x82258e            ; atoi(value)
005bcd4d: shl   eax, 0xa            ; << 10
005bcd50: mov   dword ptr [0x8a5a80], eax
```

✅ *Verified.* With the shipped `stream.ini` (`memory 13500`) the budget is **13,824,000 bytes
(13.18 MiB)**.

The four keys the parser recognises, with their destinations:

| Key | Destination | Note |
|---|---|---|
| `memory` | `0x008A5A80` | value × 1024 |
| `devkit_memory` | `0x008A5A80` | same slot; sets an override flag so `memory` cannot re-clobber it |
| `vehicles` | `0x008A5A84` | raw value (shipped: 12) |
| `dontbuildpaths` | `0x0096F016` (byte) | flag, no value |

✅ *Verified* — parser at `0x005BCCD0`, key strings at `0x0086A8C0`, `0x0086A8B0`, `0x0086A8A4`,
`0x0086A894`.

**Accounting** is in bytes at `0x008E4CB4`, adjusted by `cdSize × 2048` on every load and unload. That
`shl 0xB` is the seam between the two chapters: above it everything is bytes and budgets, below it
everything is sectors ([C1](../C1-Streaming/C1-Streaming.md)).

## 2.4 The request path

`CStreaming::RequestModel` (`0x004087E0`, **in place** on this build):

1. `edi = id × 20` — the record offset, computed once and reused as a base for every field access.
2. If `loadState == 2` (already requested) and the caller passes the **priority** flag `0x10` and the
   record does not already have it: set it and increment the priority counter at `0x008E4BA0`.
3. If `loadState != 2`, strip `0x10` from the incoming flags — priority is meaningless for something
   not already queued.
4. `flags |= requestFlags` (`+0x06`).
5. If already loaded (`loadState == 1`) and the record is linked into a list, unlink it — the classic
   "touch on use" that keeps loaded-but-idle assets from being evicted.

The unlink is a plain doubly-linked-list splice over `0x009654B4`, using `nextIndex`/`prevIndex` and
writing `0xFFFF` into both on removal. ✅ Verified.

Counters:

| Global | Role |
|---|---|
| `0x008E4BA0` | priority request count |
| `0x008E4BB0` | in-use count for the 8-slot ID array at `0x008E4C00 .. 0x008E4C20` |
| `0x008E4CB4` | memory used, bytes |
| `0x008E4CB8` | models requested |
| `0x009654B4` | pointer to the link-list array (stride 20) |

## 2.5 Dependencies

```
CStreaming
  ├── depends on: CdStream            [C1]
  ├── dispatches to: CModelInfo, TXD/COL/IPL/anim/script stores   [C3]
  └── used by:    CWorld, CGame, the script engine
```

⏳ **Open:** the eviction policy itself — which list `nextIndex`/`prevIndex` thread, and the order in
which candidates are chosen when the budget is exceeded — was not decoded in this pass. The
*mechanism* (intrusive list, touch-on-request, byte accounting) is verified; the *policy* is not.

---

### Key takeaways

- **26,316 streaming IDs**, proved by `0x8E4CC0 + 26316×20 = 0x9654B0` landing exactly on the next
  global.
- `CStreamingInfo` is **20 bytes**; `flags +0x06`, `cdSize +0x0C`, `loadState +0x10` are verified,
  three other fields are honestly marked as inference.
- Load states: `0` not loaded, `1` loaded, `2` requested.
- The budget is **`stream.ini memory × 1024`** → `0x008A5A80`; shipped value gives **13,824,000 bytes**.
- The **`shl 0xB`** in the accounting code is the exact seam between the byte world and the sector
  world.
- `RequestModel` ORs flags in, manages a priority bit that is only meaningful for already-queued
  models, and unlinks loaded models from the eviction list on touch.

**Next:** [Chapter 3 — The Model Stores](../C3-Model-Stores/C3-Model-Stores.md)

## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Loaded / consumed by:** [C0](../C0-Binary-Identity/C0-Binary-Identity.md), [C1](../C1-Streaming/C1-Streaming.md), [C3](../C3-Model-Stores/C3-Model-Stores.md), [C4](../C4-Entities-And-Pools/C4-Entities-And-Pools.md), [X1](../X1-SDK-Cross-Reference/X1-SDK-Cross-Reference.md)
- **Known bugs / gotchas:** a mis-sliced streaming-info record poisons the whole 26,316-entry array (why C2 proves it twice).
- **Modding:** the streaming-info flags @0x8E4CC0 gate what is resident; mod tools flip them.
- **Performance:** per-request bookkeeping; the array is flat and cache-friendly.
