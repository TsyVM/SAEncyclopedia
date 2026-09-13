# C4.3 — The Entity → Model Link

> **The one-sentence version:** four instructions join the entity world to the streaming world — a
> signed 16-bit model index at `CEntity+0x22`, the pointer array at `0x00A9B0C8`, and two
> dereferences to the RenderWare object.

[← C4.2 — The construction idiom](02-construction-idiom.md) · [Chapter 4 hub](C4-Entities-And-Pools.md) ·
[Next: C4.4 — Pools and the SDK →](04-pools-and-the-sdk.md)

**Confidence:** ✅ Verified (the field and the chain) / ⏳ (the rest of `CEntity`)

---

## 1. The chain

```
0054667c: movsx eax, word ptr [edi + 0x22]         ; entity->m_nModelIndex  (SIGNED)
00546680: mov   ecx, dword ptr [eax*4 + 0xa9b0c8]  ; ms_modelInfoPtrs[index]
0054669b: mov   edx, dword ptr [ecx + 0x14]        ; modelInfo->+0x14
0054669e: mov   eax, dword ptr [edx + 0x24]        ;          ->+0x24
```

✅ *Verified.* This is the join between [C4](C4-Entities-And-Pools.md) and
[C3](../C3-Model-Stores/C3-Model-Stores.md): an entity holds an ID, the ID indexes the model-info
pointer array, and the model info leads to the renderable data.

## 2. `+0x22` is signed, and that matters

`movsx` — **move with sign extension**. The field is a signed 16-bit integer, which means:

- `-1` is a meaningful value: *no model assigned*.
- The valid range is `0 … 19999` ([C3.1](../C3-Model-Stores/01-the-partition.md)), comfortably inside
  positive `int16`.
- Code that reads the field as **unsigned** turns `-1` into `65535`, then computes
  `0x00A9B0C8 + 65535 × 4` and reads **262 KB past the end** of a 20,000-entry array.

That last point is not hypothetical: reading `m_nModelIndex` as `uint16_t` is a natural mistake in any
reimplementation or SDK wrapper, and it produces a garbage pointer that is very likely mapped — so it
dereferences successfully and returns nonsense rather than crashing.

**The SDK's accessor must return a signed type or an optional**, never a raw `uint16_t`.

## 3. The two-step to RenderWare

`modelInfo + 0x14` then `+ 0x24` on the result. 🟡 *Reasoned:* the first step reaches the model's
RenderWare object (clump or atomic) and the second reaches something inside it — plausibly the frame or
the geometry. Neither offset was independently confirmed here; only the *chain* is verified, because it
appears as four consecutive instructions with no branch between them.

⏳ **Open:** what `+0x14` and `+0x24` actually name. Confirming them means reading the model-info
constructor and a RenderWare type definition, which belongs to the rendering chapter rather than this
one.

## 4. The rest of `CEntity`

✅🔷 **Closed** in [X1 §3.6](../X1-SDK-Cross-Reference/X1-SDK-Cross-Reference.md): the full 56-byte
layout — `m_pRwObject +0x18`, 32 bit-flags at `+0x1C`, `m_nRandomSeed +0x20`, `m_pReferences +0x24`,
`m_pStreamingLink +0x28`, `m_nScanCode +0x2C`, `m_nIplIndex +0x2E`, `m_nAreaCode +0x2F`,
`m_nLodIndex +0x30`, and the `m_nType : 3` / `m_nStatus : 5` bitfield at `+0x36`.

The 56-byte total matches this project's independently measured `Buildings`/`Dummys` pool element size
exactly ([C4.5 §4](05-the-cpool-object.md)) — a building is a bare `CEntity`.

*Originally left open in this pass; the paragraph below records why, and the reasoning still holds for
how the gap was framed.*

This page verified **one field** directly. The rest of the layout — the RenderWare object pointer, the
placement matrix, the flags word, the entity type discriminator, the sector list links — was not
recovered here.

That is a deliberate stopping point rather than an oversight. `CEntity` is the base of the whole entity
hierarchy (`CPhysical`, `CObject`, `CPed`, `CVehicle`) and mapping it properly needs the constructor
chain and the vtable layout, which is a chapter of its own. Publishing `+0x22` alone, clearly marked as
alone, is more useful than publishing a speculative full layout.

**What would close it:** the pool constructors from
[C4.2 §2](02-construction-idiom.md) give `sizeof(CPed)`, `sizeof(CVehicle)` and `sizeof(CBuilding)`
directly. Those three sizes constrain the hierarchy hard — they bound where each subclass's fields can
start — and they are four instructions away in seventeen functions already located.

## 5. Why this single field is worth its own page

Because it is the load-bearing one. Almost every interesting question about an entity — what does it
look like, what does it collide with, is it a vehicle — routes through `m_nModelIndex` into the model
info. A wrapper that gets this field right and nothing else can still answer a great deal; a wrapper
that gets it wrong is unsafe everywhere.

It is also the field that makes the streaming chapters pay off. `CStreaming` tracks 26,316 IDs
([C2](../C2-CStreaming/C2-CStreaming.md)); `CEntity+0x22` is where one of those IDs is actually held by
a live object. The bookkeeping and the world meet in a 16-bit field.

---

### Key takeaways

- **`CEntity::m_nModelIndex` is a *signed* 16-bit field at `+0x22`** — `movsx`, so `-1` means no model.
- Reading it as unsigned yields `65535` and a read **262 KB past** `ms_modelInfoPtrs`, which usually
  does not crash — it returns nonsense.
- The chain `entity+0x22 → ms_modelInfoPtrs[i] → +0x14 → +0x24` is verified as a chain; the meanings of
  `+0x14` and `+0x24` are **open**.
- Only this one `CEntity` field is established; the rest of the hierarchy is explicitly **not** covered
  rather than speculated.
- The next step is cheap: the **pool constructors already located** carry `sizeof(CPed)`,
  `sizeof(CVehicle)`, `sizeof(CBuilding)`, which strongly constrain the layout.

**Continue:** [C4.4 — Pools and the SDK](04-pools-and-the-sdk.md)
