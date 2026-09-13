# Chapter 3 — The Model Stores & the ID Partition

> **Goal of this chapter:** decode how one flat ID space fans out into eight independent asset stores —
> the exact partition boundaries, the store each range dispatches to, and the one range that has no
> handler at all.

**Subsystem category:** Streaming
**Depends on:** [C2 — CStreaming & the Model-ID Space](../C2-CStreaming/C2-CStreaming.md)
**Ties:** [C0](../C0-Binary-Identity/C0-Binary-Identity.md), [C2](../C2-CStreaming/C2-CStreaming.md), [C4](../C4-Entities-And-Pools/C4-Entities-And-Pools.md), [C31](../C31-Streaming-Slot-Tables/C31-Streaming-Slot-Tables.md), [C40](../C40-Render-Pipeline/C40-Render-Pipeline.md)
**RE status:** Documented
**Confidence:** ✅ Verified (boundaries) / 🟡 (some handler identities)

---

## Deep-dive pages

- [C3.1 — The partition](01-the-partition.md): the eight ranges, and why the sum is the proof.
- [C3.2 — The unhandled band](02-the-unhandled-band.md): 475 IDs with no store call.
- [C3.3 — Model-info polymorphism](03-model-info-polymorphism.md): `ms_modelInfoPtrs`, the type tag,
  and the 8-slot side table.
- [C3.4 — Typed IDs for the SDK](04-typed-ids-for-the-sdk.md): turning the partition into a type.

---

## 3.1 The partition

`CStreaming` keeps one array of 26,316 records ([C2 §2.1](../C2-CStreaming/C2-CStreaming.md)). The
asset *type* is not stored in the record — it is implied by where the ID falls. The dispatcher at
`0x004089A0` is the authoritative statement of that partition, and it reads as a descending chain of
`cmp`/`jge` against literal boundaries.

| Range | Count | Asset | Rebased index | Handler |
|---|---:|---|---|---|
| `0 – 19999` | 20,000 | models (DFF) | — | `ms_modelInfoPtrs[id]`, virtual call |
| `20000 – 24999` | 5,000 | textures (TXD) | `id − 0x4E20` | `0x00731E90` |
| `25000 – 25254` | 255 | collision (COL) | `id − 0x61A8` | `0x00410730` |
| `25255 – 25510` | 256 | map sections (IPL) | `id − 0x62A7` | ⏳ not captured |
| `25511 – 25574` | 64 | path/node data | `id − 0x63A7` | `0x0044D0F0`, `this = 0x0096F050` |
| `25575 – 25754` | 180 | animations (IFP) | `id − 0x63E7` | `0x004D3F40` |
| `25755 – 26229` | **475** | vehicle recordings | — | **none — falls through** |
| `26230 – 26315` | 86 | streamed scripts | `id − 0x6676` | `0x004708E0`, `this = 0x00A47B60` |

`20000 + 5000 + 255 + 256 + 64 + 180 + 475 + 86 = 26,316` ✅ — the partition accounts for the array
size exactly, which is the check that makes the table trustworthy rather than a list of guesses.

🟡 *Reasoned:* the asset-type labels are the long-standing community reading. What is ✅ verified is the
**boundary values**, the **rebasing arithmetic**, and the **handler address each range calls**. The
names are a convention; the SDK binds to ranges, not to labels.

### The boundaries as they appear

```
004089c6: cmp  esi, 0x4e20       ; 20000
00408a28: cmp  esi, 0x61a8       ; 25000
00408a44: cmp  esi, 0x62a7       ; 25255
00408a5d: cmp  esi, 0x63a7       ; 25511
00408a76: cmp  esi, 0x63e7       ; 25575
00408a91: cmp  esi, 0x649b       ; 25755
00408aaa: cmp  esi, 0x6676       ; 26230
```

## 3.2 The gap at 25755–26229

The chain does something unusual at the seventh boundary:

```
00408a91: cmp  esi, 0x649b       ; 25755
00408a97: jge  0x408aaa          ; -> skip to the 26230 test
...
00408aaa: cmp  esi, 0x6676       ; 26230
00408ab0: jl   0x408ac3          ; -> fall through, NO handler
```

**475 IDs have no case in this dispatcher.** An ID in that band reaches the shared tail at `0x408AC3`
and is accounted for in the memory total, but no store is told to release anything.

🟡 *Reasoned:* the band is vehicle recordings (`.rrr` playback data), which are allocated and freed by
their own subsystem rather than through the store interface — so there is nothing for this path to
call. That is a plausible reading of an intentional gap, not a bug.

⏳ **Open:** confirming what actually owns that band, and whether any code path *does* release it.
Recorded as a bounded question rather than smoothed over.

## 3.3 The model range is the odd one out

Every range from 20000 up rebases to a store-local index and calls a free function or a `thiscall` on a
fixed object. The 0–19999 range does neither:

```
004089cf: mov  ebx, dword ptr [esi*4 + 0xa9b0c8]   ; ms_modelInfoPtrs[id]
004089d6: mov  eax, dword ptr [ebx]                ; vtable
004089da: call dword ptr [eax + 0x20]              ; virtual
004089e1: call dword ptr [edx + 0x10]              ; virtual -> returns a type tag in al
004089e4: cmp  al, 7
...
00408a11: cmp  al, 6
```

**`CModelInfo::ms_modelInfoPtrs` is at `0x00A9B0C8`**, a flat array of 20,000 pointers indexed
directly by model ID — no rebasing, because model IDs *are* the store's index. ✅ Verified.

The second virtual call returns a small integer type tag; the dispatcher compares it against `7` and
`6` and takes different cleanup paths. So model info is **polymorphic** where every other store is
flat — the model range covers buildings, vehicles, peds, weapons and more, each with its own subclass,
while a TXD is just a TXD.

🟡 *Reasoned:* tags `6` and `7` are two of the model-info subclasses; which ones was not established
here. The *existence* of a vtable-dispatched type tag at vtable offset `+0x10` is ✅.

### The 8-slot side table

Within the type-7 path the dispatcher linearly scans a small array:

```
004089ee: mov  eax, 0x8e4c00
004089f3: cmp  dword ptr [eax], esi
004089f7: mov  dword ptr [eax], ebp        ; ebp = -1
004089fa: add  eax, 4
004089fd: cmp  eax, 0x8e4c20
```

`0x008E4C00 .. 0x008E4C20` — **8 entries of 4 bytes**, holding model IDs, with a count at
`0x008E4BB0` decremented on removal. ✅ Verified. A small fixed cache of "special" models that must be
forgotten when one is unloaded.

## 3.4 Why the partition matters to the SDK

The partition is the reason `SA::Streaming` cannot expose a single `Load(id)` and be honest:

- An ID alone does not tell a caller what it will get back.
- The rebasing is asymmetric — model IDs are absolute, everything else is relative to its range base.
- One band (§3.2) is not owned by this path at all.

So the SDK should expose **typed handles** (`ModelId`, `TxdId`, `ColId`, …) that carry their range, and
convert to the flat ID only at the boundary. The conversion is the table in §3.1, generated from the
verified boundaries rather than hand-written — same rule as the address database
([C0.2 §4](../C0-Binary-Identity/02-build-fingerprint-and-address-resolver.md)).

---

### Key takeaways

- The flat ID space is partitioned into **eight ranges**, and their counts sum to **exactly 26,316** —
  the check that validates the whole table.
- Boundaries, rebasing arithmetic and handler addresses are ✅ verified; the asset-type *names* are
  convention.
- **475 IDs (25755–26229) have no handler** — an intentional-looking gap, flagged as open rather than
  explained away.
- **`ms_modelInfoPtrs` is at `0x00A9B0C8`**, indexed by raw model ID; the model range is polymorphic
  (vtable type tag at `+0x10`) where every other store is flat.
- An **8-entry side table** at `0x008E4C00` tracks special models, with its count at `0x008E4BB0`.
- The SDK should carry **typed IDs**, not raw integers, and generate the conversion from these
  boundaries.

**Next:** [Chapter 4 — Entities & the Pool Allocator](../C4-Entities-And-Pools/C4-Entities-And-Pools.md)

## See also (forward links)

Model stores feed [C4 — Entities & Pools](../C4-Entities-And-Pools/C4-Entities-And-Pools.md), the per-type [C31 — Streaming Slot Tables](../C31-Streaming-Slot-Tables/C31-Streaming-Slot-Tables.md) (Col/Ipl/streamed-script stores), and the render lists of [C40](../C40-Render-Pipeline/C40-Render-Pipeline.md).

## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Loaded / consumed by:** [C0](../C0-Binary-Identity/C0-Binary-Identity.md), [C2](../C2-CStreaming/C2-CStreaming.md), [C4](../C4-Entities-And-Pools/C4-Entities-And-Pools.md), [C31](../C31-Streaming-Slot-Tables/C31-Streaming-Slot-Tables.md), [C40](../C40-Render-Pipeline/C40-Render-Pipeline.md)
- **Known bugs / gotchas:** model-store slot exhaustion silently drops loads.
- **Modding:** TXD/DFF/COL stores are where replacement assets register.
- **Performance:** fixed-size stores; O(1) slot lookup.
