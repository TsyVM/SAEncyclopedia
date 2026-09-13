# Chapter 1 — Streaming: the IMG Model & the CdStream Layer

> **Goal of this chapter:** decode how bytes get from a `.img` archive into memory — the archive
> container, the asynchronous request layer that reads it, and the boundary where raw sectors become
> the streaming system's problem instead of the file system's.

**Subsystem category:** Streaming
**Depends on:** [C0.1](../C0-Binary-Identity/01-the-hoodlum-layer.md), [C0.2](../C0-Binary-Identity/02-build-fingerprint-and-address-resolver.md)
**Ties:** [C0](../C0-Binary-Identity/C0-Binary-Identity.md), [C2](../C2-CStreaming/C2-CStreaming.md), [C3](../C3-Model-Stores/C3-Model-Stores.md), [C31](../C31-Streaming-Slot-Tables/C31-Streaming-Slot-Tables.md), [C40](../C40-Render-Pipeline/C40-Render-Pipeline.md)
**RE status:** Documented
**Confidence:** ✅ Verified

---

## Deep-dive pages

- [C1.1 — The IMG VER2 archive model](01-img-ver2-archive-model.md): the 8-byte header, the 32-byte
  directory entry, sector arithmetic, and validation across all six retail archives.
- [C1.2 — The CdStream layer](02-cdstream-layer.md): the 48-byte request struct, the 32-slot image
  table, the packed sector handle, and the six-function API.
- [C1.3 — The streaming request lifecycle and channel model](03-streaming-cache-and-request-lifecycle.md): the four phases from demand registration through model construction; one-read-per-image channel constraint; the 32-image slot limit and its modding consequences; what the sector handle's 24-bit range enables.
- [C1.4 — Modding the streaming pipeline and IMG archives](04-modding-streaming-and-img.md): the IMG editor workflow for model replacement; patching the streaming buffer for high-poly mods; adding new `.img` archives; CdStream error states and silent failure modes; the hook-point hierarchy (CdStreamRead vs CModelInfo::AddModel).

---

## 1.1 Why streaming is the first engine chapter

The project brief asks which subsystem should be documented first. Streaming is the answer, for a
reason that is structural rather than sentimental: **everything else in GTA San Andreas is downstream
of it.** Models, textures, collision, animation, scripts and the world itself all arrive through the
same pipe. `CModelInfo`, `CTxdStore`, `CColStore` and `CWorld` are all, from one angle, consumers of
whatever the streamer produced. Document the pipe and every later chapter has a floor to stand on;
document a consumer first and you are describing a function of an input you have not defined.

It is also the subsystem with the highest ratio of *verifiable* to *inferable* content. The archives
are on disk and can be parsed exhaustively; the request layer is small, self-contained, and touches a
tight cluster of globals. That makes it the right place to establish the encyclopedia's evidence
standard before tackling anything that requires judgement.

## 1.2 The two layers

The streaming system splits cleanly at the sector boundary:

```
   .img archive on disk                       [C1.1]  format, offsets, sizes
        │
        │  a request is (imageIndex, sectorOffset, sectorCount, buffer)
        ▼
   CdStream                                   [C1.2]  async queue, 48-byte requests
        │
        │  a buffer of raw bytes, and a status code
        ▼
   CStreaming ──► CModelInfo / CTxdStore / CColStore / CIplStore
        │
        ▼
   CWorld
```

Below the line, nothing knows what a model is. `CdStream` moves sectors. Above the line, nothing knows
what a sector is. The interface between them is a packed 32-bit value and a byte count — §1.4.

## 1.3 The archive layer in one paragraph

An IMG archive is a directory of fixed 32-byte records followed by 2048-byte-aligned payloads. The
header is `'VER2'` plus a `u32` entry count; each record is a sector offset, a sector count, and a
24-byte name. There is no compression, no per-file header, and no index beyond the directory — the
format exists to make "seek here, read this many sectors" a single arithmetic step. Verified across
all six retail archives: **20,168 entries, zero malformed records, zero trailing slack**
([C1.1](01-img-ver2-archive-model.md)).

## 1.4 The packed sector handle — the chapter's keystone

The single design decision that ties both layers together is that an archive and a position inside it
are carried in **one 32-bit word**:

```
 31           24 23                              0
┌───────────────┬─────────────────────────────────┐
│  image index  │        sector offset            │
└───────────────┴─────────────────────────────────┘
        8 bits              24 bits
```

`CdStreamOpen` returns `index << 24` when it opens an archive; every caller then ORs a sector offset
into the low 24 bits and hands the result back to `CdStreamRead`, which splits it again with
`shr edx, 0x18` / `and esi, 0xFFFFFF`.

✅ *Verified* from both sides of the round trip ([C1.2](02-cdstream-layer.md)). The consequences are
exact and worth stating as engine limits rather than folklore:

| Limit | Value | Origin |
|---|---|---|
| Concurrent archives | **32** | the handle-table scan bound, not the 8 bits |
| Addressable offset per archive | 2²⁴ sectors = **32 GiB** | the 24-bit field |
| Largest single file | 2¹⁶ sectors = **128 MiB** | the `u16` size field in the directory |

Note the asymmetry: the *field* allows 256 archives but the *table* holds 32, so the binding
constraint is the table. This is exactly the kind of claim that gets repeated wrongly, and both halves
are verified in [C1.2 §4](02-cdstream-layer.md).

## 1.5 Synchronous and asynchronous modes

`CdStream` runs in one of two modes, selected by two globals (`0x008E3FE4`, `0x008E3FE8`) that are
tested at the head of every entry point:

- **Async** — requests are pushed onto a ring buffer and a worker is released via a semaphore; the
  archive is opened with `FILE_FLAG_OVERLAPPED` and completion is collected with
  `GetOverlappedResult`.
- **Sync** — the read happens inline and completion is polled by waiting on the file handle itself
  with a zero timeout.

Both paths write the same 48-byte request struct and return the same status codes, so consumers above
the line cannot tell which is running. ✅ Verified — [C1.2 §2–3](02-cdstream-layer.md).

## 1.6 Dependencies

```
CdStream
  ├── depends on: KERNEL32 (CreateFileA, ReadFile, GetOverlappedResult,
  │               WaitForSingleObject[Ex], SetFilePointer, SetLastError, CloseHandle)
  └── used by:    CStreaming, CFileMgr

CStreaming                              [C2, not yet written]
  ├── depends on: CdStream, CModelInfo, CTxdStore, CColStore
  └── used by:    CWorld, CGame
```

⏳ **Open:** `CStreaming` itself — the model-ID space, the request queue above `CdStream`, and the
priority/eviction policy — is the next chapter and is not covered here. This chapter deliberately
stops at the sector boundary.

## 1.7 A note on where the code lives

Two of this chapter's six functions — `CdStreamRead` and `CdStreamSync` — are among the 492 relocated
into `.HOODLUM` on this build ([C0.1](../C0-Binary-Identity/01-the-hoodlum-layer.md)). Their documented
addresses hold a 5-byte `jmp`; their bodies are 17 MB away. Everything in
[C1.2](02-cdstream-layer.md) was recovered by following the relocation map, and none of it appears in
the external analysis dump's function catalogue
([C0.3 §5](../C0-Binary-Identity/03-external-analysis-corroboration-and-conflicts.md)).

This is the first chapter where C0 stops being throat-clearing and starts paying: without the
relocation map, the two most important functions in the streaming layer are invisible.

---

### Key takeaways

- Streaming is documented first because **every other subsystem is downstream of it**, and because it
  is the most verifiable subsystem in the engine.
- The system splits at the **sector boundary**: `CdStream` moves sectors and knows nothing about
  models; `CStreaming` and above know nothing about sectors.
- The **packed sector handle** (`imageIndex << 24 | sectorOffset`) is the interface between the two
  layers, verified from both the producing and consuming side.
- Engine limits that fall out of it: **32 archives**, **32 GiB** addressable per archive, **128 MiB**
  per file — each traced to the specific field or bound that imposes it.
- Two modes (async ring buffer + overlapped I/O, or inline + polled) share one request struct, so
  consumers cannot distinguish them.
- **`CdStreamRead` and `CdStreamSync` are relocated functions** — this chapter is the first
  demonstration that C0's relocation map is load-bearing, not bookkeeping.

**Next:** [C1.1 — The IMG VER2 archive model](01-img-ver2-archive-model.md)

## See also (forward links)

The streamer feeds [C2 — CStreaming](../C2-CStreaming/C2-CStreaming.md) (the request path), [C3 — Model Stores](../C3-Model-Stores/C3-Model-Stores.md), and the per-type [C31 — Streaming Slot Tables](../C31-Streaming-Slot-Tables/C31-Streaming-Slot-Tables.md). The streamed atomics are drawn by [C40 — Render Pipeline](../C40-Render-Pipeline/C40-Render-Pipeline.md)'s four visible-entity lists.

## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Loaded / consumed by:** [C0](../C0-Binary-Identity/C0-Binary-Identity.md), [C2](../C2-CStreaming/C2-CStreaming.md), [C3](../C3-Model-Stores/C3-Model-Stores.md), [C31](../C31-Streaming-Slot-Tables/C31-Streaming-Slot-Tables.md), [C40](../C40-Render-Pipeline/C40-Render-Pipeline.md)
- **Known bugs / gotchas:** streaming races can pop entities in/out at chunk edges; the request queue is bounded.
- **Modding:** streaming memory budget + the IMG archive are the classic mod-content injection points.
- **Performance:** the dominant background cost; async disk + eviction each frame.
