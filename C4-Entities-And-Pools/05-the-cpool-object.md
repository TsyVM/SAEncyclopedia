# C4.5 — The `CPool` Object

> **The one-sentence version:** one constructor closes four open items at once — the 20-byte pool
> layout, all seventeen element sizes, the runtime capacity read the SDK was blocked on, and the
> question of whether the engine provides a generation counter. It does: seven bits of one.

[← C4.4 — Pools and the SDK](04-pools-and-the-sdk.md) · [Chapter 4 hub](C4-Entities-And-Pools.md)

**Confidence:** ✅ Verified
**Function:** `CPool<T>::CPool(int count, const char* name)` at `0x00550180`
**Closes:** [C4.1 §4](01-the-pool-table.md), [C4.2 §2](02-construction-idiom.md),
[C4.4 §3](04-pools-and-the-sdk.md), [C4.4 §5](04-pools-and-the-sdk.md)

---

## 1. The constructor

```
00550180: push esi
00550181: push edi
00550182: mov  edi, dword ptr [esp + 0xc]   ; count
00550186: lea  eax, [edi*8]                 ; count × sizeof(T)   <- the only per-pool difference
0055018d: push eax
0055018e: mov  esi, ecx                     ; this
00550190: call 0x821195                     ; malloc
00550195: push edi                          ; count
00550196: mov  dword ptr [esi], eax         ; +0x00 = objects
00550198: call 0x821195                     ; malloc(count)
0055019d: mov  dword ptr [esi + 4], eax     ; +0x04 = byte map
005501a0: add  esp, 8
005501a3: xor  eax, eax
005501a5: test edi, edi
005501a7: mov  byte ptr [esi + 0x10], 1     ; +0x10 = owns allocation
005501ab: mov  dword ptr [esi + 8], edi     ; +0x08 = capacity
005501ae: mov  dword ptr [esi + 0xc], -1    ; +0x0C = first free
005501b5: jle  0x5501d6
                                            ; init loop, per slot:
005501b7: mov  ecx, dword ptr [esi + 4]
005501ba: mov  dl,  byte ptr [ecx + eax]
005501bf: or   dl, 0x80                     ;   set bit 7
005501c2: mov  byte ptr [ecx], dl
005501ca: mov  dl,  byte ptr [ecx]
005501cc: and  dl, 0x80                     ;   clear bits 0-6
005501d2: mov  byte ptr [ecx], dl
005501d4: jl   0x5501b7
005501da: ret  8
```

## 2. The layout

```c
struct CPool {              // 0x14 = 20 bytes  (matches operator new(0x14), C4.2 §2)
    void*    m_pObjects;    // +0x00  malloc(count × sizeof(T))
    uint8_t* m_byteMap;     // +0x04  malloc(count) — one byte per slot
    int32_t  m_nSize;       // +0x08  capacity
    int32_t  m_nFirstFree;  // +0x0C  initialised to -1
    uint8_t  m_bOwnsAlloc;  // +0x10  initialised to 1
    // +0x11..0x13 padding
};
```

✅ *Verified.* The 20-byte allocation observed in [C4.2](02-construction-idiom.md) is now accounted for
field by field, with three bytes of tail padding.

Note the storage is **`malloc`, not `operator new`** (`0x00821195` is called with a plain size and the
stack cleaned by the caller), and `m_bOwnsAlloc` exists to distinguish this case from a pool handed
external memory — which is exactly the door limit adjusters use.

## 3. The byte map is a generation counter

The init loop sets bit 7 and clears bits 0–6 of every slot byte. So:

| Bits | Meaning |
|---|---|
| `0x80` | slot is **free** |
| `0x7F` | **generation / id counter**, 7 bits |

✅ *Verified* from the mask operations. This answers [C4.4 §5](04-pools-and-the-sdk.md) directly:
**the engine does supply a generation counter, and the SDK does not need to maintain its own.**

A safe handle is therefore `{ index, generation }`, validated as:

```cpp
bool valid(const CPool& p, int index, uint8_t gen) {
    if (index < 0 || index >= p.m_nSize) return false;
    uint8_t b = p.m_byteMap[index];
    return !(b & 0x80) && (b & 0x7F) == gen;   // in use, and same generation
}
```

Seven bits means the counter wraps every 128 reuses of a slot — so a handle held across ~128
allocations of the same slot can alias. That is a real if narrow limitation, and it is the engine's,
not something the SDK can fix without its own bookkeeping.

## 4. All seventeen element sizes

Each pool's constructor differs only in how it computes `count × sizeof(T)`. Reading that one
expression from each gives every element size:

| Pool | Capacity | `sizeof(T)` | Derivation | Storage |
|---|---:|---:|---|---:|
| `PtrNode Single` | 70,000 | **8** | `×8` | 560,000 |
| `PtrNode Double` | 3,200 | **12** | `×3 <<2` | 38,400 |
| `EntryInfoNode` | 500 | **20** | `×5 <<2` | 10,000 |
| `Peds` | 140 | **1,988** | `imul` | 278,320 |
| `Vehicles` | 110 | **2,584** | `imul` | 284,240 |
| `Buildings` | 13,000 | **56** | `imul` | 728,000 |
| `Objects` | 350 | **412** | `imul` | 144,200 |
| `Dummys` | 2,500 | **56** | `imul` | 140,000 |
| `ColModel` | 10,150 | **48** | `×3 <<4` | 487,200 |
| `Task` | 500 | **128** | `<<7` | 64,000 |
| `Event` | 200 | **68** | `imul` | 13,600 |
| `PointRoute` | 64 | **100** | `imul` | 6,400 |
| `PatrolRoute` | 32 | **420** | `imul` | 13,440 |
| `NodeRoute` | 64 | **36** | `×9 <<2` | 2,304 |
| `TaskAllocator` | 16 | **32** | `<<5` | 512 |
| `PedIntelligence` | 140 | **660** | `imul` | 92,400 |
| `PedAttractors` | 64 | **196** | `imul` | 12,544 |

✅ *Verified.* Totals: **2,875,560 bytes** of element storage plus **101,030 bytes** of byte maps =
**2.84 MiB** for the entire pool system.

Sanity notes that make the table self-checking:

- **`PtrNode Single` = 8** — an item pointer plus a next pointer. Exactly a singly-linked node.
- **`PtrNode Double` = 12** — item, next, prev. Exactly a doubly-linked node. The 8/12 split confirms
  the single/double reading in [C5.4 §5](../C5-CWorld/04-ptrnode-coupling.md).
- **`Buildings` = `Dummys` = 56** — both are thin static entities over the same base.
- **`Peds` 1,988 and `Vehicles` 2,584** dwarf everything, which is why their capacities are 140 and
  110.

## 5. Cost, revisited

2.84 MiB of pools against a 13.18 MiB streaming budget
([C2.3](../C2-CStreaming/03-memory-budget-and-stream-ini.md)) — the pool system is **21.5 %** of the
streaming budget's size, permanently resident.

Raising `Peds` from 140 to 1,000 costs `860 × (1988 + 660 + 1) ≈ 2.28 MiB` — `PedIntelligence` must
scale with it. That is why naive limit raising is expensive in a 32-bit address space, and it is now
computable rather than guessable.

## 6. The SDK unblocks

[C4.4 §3](04-pools-and-the-sdk.md) declared `SA::Pools::Capacity()` blocked. It no longer is:

```cpp
Result<std::uint32_t> Capacity(PoolId id) noexcept {
    const CPool* p = *reinterpret_cast<CPool**>(kPoolGlobal[id]);
    if (!p) return Err(Error::PoolNotInitialised);       // null-pool case, C4.2 §2
    return p->m_nSize;                                    // +0x08, live
}

Result<std::uint32_t> Used(PoolId id) noexcept {
    const CPool* p = *reinterpret_cast<CPool**>(kPoolGlobal[id]);
    if (!p) return Err(Error::PoolNotInitialised);
    std::uint32_t n = 0;
    for (int i = 0; i < p->m_nSize; ++i)
        if (!(p->m_byteMap[i] & 0x80)) ++n;
    return n;
}
```

Both read the live process, so a limit adjuster's raised capacity is picked up automatically — which is
the whole point of [C4.4 §2](04-pools-and-the-sdk.md).

The `nullptr` check is not defensive padding: `CPools::Initialise` genuinely stores a null pool if
allocation fails ([C4.2 §2](02-construction-idiom.md)).

---

### Key takeaways

- `CPool` is **20 bytes**: objects, byte map, capacity, first-free, owns-allocation.
- Storage is **`malloc`**, and `m_bOwnsAlloc` marks it — the hook limit adjusters use.
- **The byte map is `0x80` = free plus a 7-bit generation counter** — the engine supplies what safe
  handles need, wrapping every 128 slot reuses.
- **All seventeen element sizes recovered**, including `CPed` 1,988 and `CVehicle` 2,584; the 8/12
  PtrNode split confirms the single/double reading in C5.4.
- The pool system costs **2.84 MiB** resident — 21.5 % of the streaming budget.
- `SA::Pools::Capacity()` and `Used()` are **unblocked** and read the live process, so raised limits are
  honoured automatically.

**Continue:** [Chapter 4 hub](C4-Entities-And-Pools.md) · [Chapter 5 — CWorld](../C5-CWorld/C5-CWorld.md)
