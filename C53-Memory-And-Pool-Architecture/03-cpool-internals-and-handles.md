# C53.3 — CPool<T> internals and handle encoding

## The 20-byte structure in full

Every pool is a `CPool<T>` — a 20-byte management header. The five fields and their proven addresses (from the pool-1 CPed constructor at `0x54F7F0`):

```
struct CPool<T> {
  /* +0x00 */ void*    m_pObjects;       // heap block: count × stride bytes
  /* +0x04 */ uint8_t* m_byteMap;        // parallel flag array: count bytes
  /* +0x08 */ int32_t  m_nSize;          // slot count (written once at ctor)
  /* +0x0C */ int32_t  m_nFirstFree;     // free-slot cache; -1 if unknown
  /* +0x10 */ uint8_t  m_bOwnsMemory;    // 1 = pool frees its own blocks
};
```

Evidence VAs: `m_pObjects` → `0x54F420`, `m_byteMap` → `0x54F434`, `m_nSize` → `0x54F74A`, `m_nFirstFree` → `0x54F74F`, `m_bOwnsMemory` → `0x54F746`.

## The byteMap

`m_byteMap` is a parallel byte array — one byte per slot. Its two roles:

**Bit 7 (mask `0x80`):** The in-use flag.
- `m_byteMap[slot] & 0x80 == 0x80` → slot is **free** (the inverted convention trips up first-time readers)
- `m_byteMap[slot] & 0x80 == 0x00` → slot is **in use**

**Bits 6-0 (mask `0x7F`):** The allocation counter (sometimes called the "stamp" or "version"). This is a 7-bit value incremented each time a slot is allocated. Its purpose is handle validation — if a slot is freed and reallocated, the version number changes, making stale handles from the old occupant invalid.

At construction, `m_byteMap` is initialised with all bytes set to `0x80` (all slots free, version = 0).

## The allocation loop

When a new entity is requested (`CPool::New()`), the allocator:

1. Checks `m_nFirstFree`. If ≥ 0 and `m_byteMap[m_nFirstFree] & 0x80` (still free), uses it.
2. Otherwise, linear-scans `m_byteMap[0..m_nSize-1]` for the first byte with bit 7 set.
3. If no free slot found → returns null (pool full — silent fail).
4. For the chosen slot `i`:
   - Clears bit 7: `m_byteMap[i] &= 0x7F` (marks in-use)
   - Increments the version counter: `m_byteMap[i] = (m_byteMap[i] + 1) & 0x7F` (if that wraps to 0, it continues to 1 to avoid the "version 0 = just-freed" ambiguity)
   - Returns a pointer to `m_pObjects + i × stride`
5. Updates `m_nFirstFree` to `i + 1` as a hint for the next allocation.

The linear scan in step 2 is O(n) — this is why very large pool counts with many allocations/frees per frame have a performance cost. The `m_nFirstFree` cache makes the common case O(1) when allocations are sequential, but any free in the middle of the pool invalidates the cache.

## The handle encoding

A **handle** is a 32-bit value the game uses to refer to an entity across frames and script calls. It encodes both the slot index and the byteMap version:

```
handle = (slot_index << 8) | (m_byteMap[slot_index] & 0x7F)
```

Decoding a handle to get a live pointer:
```
slot_index  = handle >> 8
version     = handle & 0xFF
entity_ptr  = m_pObjects + slot_index × stride
is_valid    = (m_byteMap[slot_index] & 0x7F) == version
           && (m_byteMap[slot_index] & 0x80) == 0   // slot must be in use
```

Evidence VA for the encoding: `0x54F441` (the `CPool::GetSlot`/`GetHandle` path inside the CPed pool constructor area).

The version bits are why handle 0 is not valid — `slot 0, version 0` would be ambiguous with "no handle". When a new slot at index 0 is first allocated, the version is incremented from 0 to 1, so the first real handle for slot 0 is `0x0001`, not `0x0000`.

## What handle validation protects against

The version mechanism prevents **use-after-free** entity references:

1. Entity A is allocated at slot 5. Handle is `0x0501` (slot 5, version 1).
2. Entity A is freed. `m_byteMap[5]` is set to `0x80 | 0x01` = `0x81` (free, version still 1).
3. Entity B is allocated into slot 5. byteMap increments to `0x02`, clears bit 7 → `m_byteMap[5] = 0x02`. Handle for B is `0x0502`.
4. Old code still holding handle `0x0501` calls `CPool::GetAt(0x0501)`:
   - Slot 5's current version = 2; handle version = 1. Mismatch → returns null.
   - The stale reference is safely detected.

Without version bits, slot reuse would cause the old handle to silently return entity B — the wrong entity.

## The free path

`CPool::Delete(entity_ptr)`:

1. Computes `slot_index = (entity_ptr - m_pObjects) / stride`
2. Sets bit 7 on the byteMap entry: `m_byteMap[slot_index] |= 0x80`
3. The version bits are NOT cleared — they stay at their current value. This is how the version scheme works: the version only changes when the slot is next allocated, not when it is freed.
4. Updates `m_nFirstFree = slot_index` if slot_index < m_nFirstFree (prefers to reuse earlier slots, keeping allocations near the front of the pool).

The destructor for the entity (`~CPed`, `~CVehicle`, etc.) is called by the higher-level entity manager, NOT by `CPool::Delete` — the pool itself is type-agnostic and just manages the slot.

## The slot-count relationship to handle ranges

Because handles encode `(slot << 8)`, the maximum valid slot index for a pool with count `N` is `N - 1`. The maximum handle value for the CPed pool (140 slots) is `(139 << 8) | 0x7F` = `0x8B7F`. Script code that tests handles against ranges can break if the pool count is raised and previously-impossible handle values start appearing. **This is the main reason never to reduce pool counts below default** — scripts may use handle ranges as entity-type tests.

## Relationship to the global pool pointers

Each of the 13 pool globals (e.g., `0xB74490` for CPed) holds a pointer to its `CPool<T>` header. The header points to the heap-allocated `m_pObjects` and `m_byteMap` blocks. The layout:

```
0xB74490 → [CPool<CPed> header 20 bytes]
               m_pObjects → [CPed[0] | CPed[1] | ... | CPed[139]]  (140 × 1988 bytes)
               m_byteMap  → [flag[0] | flag[1] | ... | flag[139]]  (140 × 1 byte)
```

All three allocations — the pool header, the objects block, and the byteMap — are on the game heap via `operator new` at `0x820595`.

## Key takeaways

- `m_byteMap` Bit 7 = **0** means in-use (inverted), Bit 7 = **1** means free.
- Bits 6-0 are the **version counter** — incremented on every new allocation into that slot.
- Handle encoding: `(slot << 8) | (version)`. Validation requires both the version match AND the slot being in-use.
- The allocation loop is **O(n)** in the worst case (linear scan); the `m_nFirstFree` cache makes sequential allocation O(1).
- The free path does not clear the version — version changes only on the next allocation, enabling safe detection of stale references.

**Previous:** [C53.2 — Memory budget](02-memory-budget.md)  
**Continue:** [C53.4 — Pool exhaustion and streaming →](04-pool-exhaustion-and-streaming.md)
