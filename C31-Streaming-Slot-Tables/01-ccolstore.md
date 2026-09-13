# C31.1 — CColStore, Re-confirmed and Located

> **The one-sentence version:** [C6.3](../C6-Collision/03-colstore-and-binding.md) found `CColStore` to be
> a `CPool` of 44-byte records while chasing the collision-to-model binding; three different `CColStore`
> methods reach the identical 44-byte stride here, and add what C6.3 did not have — the pool object's
> address (`0x965560`) and the three fields inside it, plus the per-slot status byte at +0x28.

**Subsystem category:** Streaming — collision store
**Depends on:** [C31 hub](C31-Streaming-Slot-Tables.md), [C6.3](../C6-Collision/03-colstore-and-binding.md)
(the store's 44-byte record and the model binding)
**RE status:** Documented
**Confidence:** ✅ for the stride (now two independent derivations), the pool address and the layout ·
🔷 for methods not disassembled here

---

## 1. The 44-byte slot, reached a second way

[C6.3 §](../C6-Collision/03-colstore-and-binding.md) established the CColStore record as 44 bytes from the
binding path. `CColStore::RemoveCol` (`entry_va 0x00410730`) reaches it independently:

```
01564E9D  mov  eax, dword ptr [0x965560]   ; the CColStore CPool object
01564EA0  mov  ecx, dword ptr [eax + 4]    ; +4 = per-slot flag-byte array
01564EA6  cmp  byte ptr [ecx + ebx], 0     ; this slot's flag
...
01564EAA  mov  ecx, dword ptr [eax]        ; +0 = packed slot array base
01564EAD  imul esi, esi, 0x2C              ; index * 44
01564EB1  add  esi, ecx                     ; -> slot address
```

Stride `0x2C` = **44 bytes**, the same value C6.3 read from the binding code — reached here from a wholly
different method. `RemoveColSlot` (`0x00411330`) and `IncludeModelIndex` (`0x00410820`) use the same
`imul …, 0x2C`, so three `CColStore` methods plus C6.3's binding path now agree on the record width. What
C6.3 could leave at "a `CPool` of 44-byte records" is now also *located*: the pool object is the global at
**`0x965560`**.

## 2. The CPool layout, and the full slot field map

`RemoveCol` above already shows the pool object's shape — the two dereferences `[eax]` and `[eax + 4]` are
its first two fields:

- **+0** — pointer to the packed array of 44-byte slots.
- **+4** — pointer to a parallel array of one status byte per slot (`RemoveCol` tests it with `jns`, so the
  high bit marks a free/invalid slot — the standard `CPool` free-flag convention, same as C30.1's entry/exit
  pool).
- **+8** — the slot count (read as the loop bound elsewhere in the class).

This is the identical three-field `CPool` object C30.1 recovered for `CEntryExitManager`, confirming it as
the engine's shared pool template rather than a one-off. The RE-Data sweep recovered all nine fields inside
the 44-byte slot:

```cpp
struct CColStoreSlot {          // 44 bytes (0x2C)
    float    m_Area[4];         // +0   bounding rect (left, bottom, right, top); read by HasCollisionLoaded
    char     name[18];          // +16  null-terminated collision filename, e.g. "generic" (max 18 chars)
    int16_t  m_nModelIdStart;   // +34  first model index covered by this .col file
    int16_t  m_nModelIdEnd;     // +36  last model index covered by this .col file
    uint16_t m_nRefCount;       // +38  reference count; 0 = eligible for unload
    uint8_t  m_bActive;         // +40  non-zero = slot is active and data is loaded
    bool     m_bCollisionIsRequired; // +41 must not be unloaded (SetCollisionRequired)
    bool     m_bProcedural;     // +42  procedurally generated collision, not from a .col file
    bool     m_bInterior;       // +43  interior collision (separate streaming priority)
};
static_assert(sizeof(CColStoreSlot) == 0x2C);
```

`RemoveColSlot` reads `m_bActive` at +40 (`+0x28` from slot base) to decide whether the collision is
currently loaded before freeing the slot:

```
0156723D  imul eax, eax, 0x2C
01567243  mov  dl, byte ptr [eax + 0x28]   ; m_bActive
01567246  test dl, dl                       ; loaded? -> unload it first
```

The `m_nModelIdStart`/`m_nModelIdEnd` pair (offsets +34/+36) directly feeds `IncludeModelIndex`
(`0x00410820`) — that method receives a model index and walks the slot table looking for a slot whose range
contains it, closing the loop C6.3 opened when it found the collision-to-model binding.

## 3. The rest of the class

The remaining methods (🔷) are the collision-streaming request path: proximity requests
(`AddCollisionNeededAtPosn`, `SetCollisionRequired`), reference counting (`AddRef`, `RemoveRef`), the load
itself (`LoadCol[2]`), bounds (`GetBoundingBox`), and slot allocation (`AddColSlot`). These sit directly on
[C6](../C6-Collision/C6-Collision.md)'s collision container work — C6 decodes the COL data a slot points at;
this class is the slot table that owns it. Full list in
[`streaming_stores.json`](../RE-Data/data/streaming_stores.json).

---

### Key takeaways

- The CColStore slot is **44 bytes (`0x2C`)** — now proven twice, by [C6.3](../C6-Collision/03-colstore-and-binding.md)'s
  binding path and by three `CColStore` methods here.
- The store is a **`CPool` at `0x965560`** with the engine's shared three-field layout (slot array +0,
  status-byte array +4, count +8) — the same template as C30.1's entry/exit pool.
- The full slot layout: bounding rect @+0, filename @+16, model-index range @+34/+36, ref-count @+38,
  four status bytes (active, required, procedural, interior) @+40–+43.

**Next:** [C31.2 — CIplStore and the 52-byte slot](02-ciplstore.md).
