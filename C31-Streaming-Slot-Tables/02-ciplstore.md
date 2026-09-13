# C31.2 — CIplStore and the 52-byte Slot

> **The one-sentence version:** the IPL store — the streaming table behind [C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md)'s
> map placements — is a sibling `CPool` at `0x8E3FB0` whose slot is **52 bytes**, the stride read the same
> way from four methods and its pool object laid out identically to `CColStore`'s, differing only in the
> slot width.

**Subsystem category:** Streaming — IPL (map section) store
**Depends on:** [C31 hub](C31-Streaming-Slot-Tables.md), [C31.1](01-ccolstore.md),
[C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md) (IPL placements)
**RE status:** Documented
**Confidence:** ✅ for the stride, pool address and layout · 🔷 for methods not disassembled here

---

## 1. The 52-byte slot

`CIplStore::RequestIplAndIgnore` (`entry_va 0x00405850`) indexes the store exactly as `CColStore` does, only
with a wider slot:

```
015651A0  mov  ecx, dword ptr [0x8E3FB0]   ; the CIplStore CPool object
015651A3  mov  edx, dword ptr [ecx + 4]    ; +4 = per-slot flag-byte array
015651A6  cmp  byte ptr [edx + eax], 0     ; this slot's flag
...
015651AA  mov  edx, dword ptr [ecx]        ; +0 = packed slot array base
015651AC  imul esi, esi, 0x34              ; index * 52
015651B1  add  esi, edx                     ; -> slot address
```

Stride `0x34` = **52 bytes**, pool object at **`0x8E3FB0`**. The same `imul …, 0x34` appears in
`RemoveIplAndIgnore` (`0x00405890`), `IncludeEntity` (`0x00404C90`) and `RemoveIplSlot` (`0x00405B60`), and
`Shutdown` (`0x00405FA0`) walks the array with `add …, 0x34` — four independent witnesses to the 52-byte
slot. `RemoveIplSlot` also carries the reciprocal-multiply that inverts a slot pointer back to an index
(`imul` by the ÷52 magic), the same two-directions proof used throughout C28–C30.

## 2. Same pool object, different width — full slot field map

The two dereferences in the loop above — `[ecx]` (slot array) and `[ecx + 4]` (status-byte array) — are the
first two fields of the identical `CPool` object `CColStore` uses, with the count at +8 as its loop bound
elsewhere. So `CColStore` and `CIplStore` are the **same store template instantiated twice**: one field-for-
field layout, two slot widths (44 vs 52 bytes) and two pool addresses (`0x965560` vs `0x8E3FB0`).

The RE-Data sweep recovered all twelve fields inside the 52-byte slot:

```cpp
struct CIplStoreSlot {              // 52 bytes (0x34)
    float   bb[4];                  // +0   bounding rect (left, bottom, right, top) — streaming trigger area
    char    name[18];               // +16  null-terminated IPL filename, e.g. "LA_SFC" (max 18 chars)
    int16_t firstBuilding;          // +34  first CBuilding entity index in this IPL (SHRT_MAX = unpopulated)
    int16_t lastBuilding;           // +36  last CBuilding entity index (SHRT_MIN = unpopulated)
    int16_t firstDummy;             // +38  first CDummy entity index in this IPL (SHRT_MAX = unpopulated)
    int16_t lastDummy;              // +40  last CDummy entity index (SHRT_MIN = unpopulated)
    int16_t staticIdx;              // +42  entity-array index for this IPL; -1 = none
    bool    isInterior;             // +44  true = interior geometry (alters streaming priority)
    uint8_t loaded;                 // +45  non-zero = IPL data currently in memory
    bool    loadRequested;          // +46  true = load request pending
    bool    disableDynamicStreaming;// +47  true (default) = never dynamically unloaded (always-present cells)
    uint8_t ignoreWhenDeleted;      // +48  suppresses deletion notifications during shutdown
    uint8_t isLarge;                // +49  true = use +350 unit bounding-box expansion (vs default +200)
    uint8_t _tail_pad[2];           // +50  alignment to 52 bytes
};
static_assert(sizeof(CIplStoreSlot) == 0x34);
```

The `firstBuilding`/`lastBuilding` and `firstDummy`/`lastDummy` ranges at +34–+41 are the entity bookkeeping
that `IncludeEntity` (`0x00404C90`) writes when an IPL section is loaded — the entity-index pairs that let
the engine know which `CBuilding`/`CDummy` objects belong to this IPL section and must be freed when it
unloads. `disableDynamicStreaming` (defaulting true) is the reason always-present cells like the LA mainland
never unload mid-session.

`RemoveIplSlot`'s `entry_va` (`0x00405B60`) is also one of the twelve C27.3 disassembled to confirm its
name — there it was read as "an index-bounds-checked slot clear"; here the same body yields the 52-byte
stride, an example of C28's point that a name-verification and a structure-recovery are the same disassembly
read for two different purposes.

## 3. The rest of the class

The remaining methods (🔷) are the IPL request path and its entity bookkeeping: proximity requests
(`AddIplsNeededAtPosn`, `ClearIplsNeededAtPosn`), the per-slot entity-index arrays that record which world
entities an IPL section spawned (`GetNewIplEntityIndexArray`, `GetIplEntityIndexArray`,
`IncludeEntity` — the method with the most callers), and distance-based unload (`RemoveIplWhenFarAway`).
These are the runtime engine behind [C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md)'s static IPL placements:
C11 decodes the placement files; this class is the table that streams their sections in and out and tracks
the entities each produced. Full list in [`streaming_stores.json`](../RE-Data/data/streaming_stores.json).

---

### Key takeaways

- The CIplStore slot is **52 bytes (`0x34`)** — stride seen in four methods plus the ÷52 reciprocal in
  `RemoveIplSlot` — in a `CPool` at **`0x8E3FB0`**.
- `CColStore` and `CIplStore` are the **same `CPool` template** (array +0, status bytes +4, count +8) at two
  addresses with two slot widths — the concrete shape of C1's streaming fan-out.
- Full slot layout: bounding rect @+0, filename @+16, building/dummy entity ranges @+34–+41, staticIdx @+42,
  six status bytes (isInterior, loaded, loadRequested, disableDynamicStreaming, ignoreWhenDeleted, isLarge)
  @+44–+49, 2B tail pad @+50.
- `disableDynamicStreaming` (true by default) is why always-present map cells never unload mid-session.

**Next:** [C31.3 — CStreamedScripts and the 82-slot table](03-cstreamedscripts.md).
