# Chapter 53 — Memory and Pool Architecture

> **Goal of this chapter:** map San Andreas's runtime entity memory — the 13 fixed-size pools
> allocated once at startup by `CPools::Initialise` at `0x5503A0` and never resized — with every
> slot count proven cold from `gta_sa.exe`. A pool is a **`CPool<T>`**: a slab of `count × stride`
> bytes, a parallel byte-flag array for in-use bits and handle versioning, a slot-count field, and
> a first-free cursor. The four main entity pools (CPed=140, CVehicle=110, CBuilding=13000,
> CObject=350) are the dominant limit-patching targets; this chapter sizes them, explains the
> memory budget, describes the `CPool` allocation and handle protocol, and maps the symptoms of
> silent-fail exhaustion.

**Subsystem category:** Engine substrate — object pool memory management (`CPools`, `CPool<T>`)
**Depends on:** [C4](../C4-CPool-Internals/C4-CPool-Internals.md) (the `CPool` handle protocol),
[C28](../C28-Class-Catalogue/C28-Class-Catalogue.md) (disassembly of the constructor pattern),
[C51](../C51-Executable-Lifecycle/C51-Executable-Lifecycle.md) (`CGame::Initialise` calls
`CPools::Initialise`)
**Ties:** [C54](../C54-Limits-Reference/C54-Limits-Reference.md) (pool counts as patchable limits),
[C29](../C29-Gameplay-Object-Pools/C29-Gameplay-Object-Pools.md) (gameplay pools atop these),
[C1](../C1-Streaming/C1-Streaming.md)/[C2](../C2-CStreaming/C2-CStreaming.md) (streaming feeds
entities into pools)
**RE status:** Documented
**Confidence:** ✅ for all 13 pool counts (proven from `push-imm32` at constructor call sites) and
for the four main strides (CPed=1988, CVehicle=2584, CBuilding=56, CObject=412); 🟡 for strides of
pools 5–12 (constructor register paths not yet decoded)
**Data artifact:** [`RE-Data/data/pool_structure.json`](../RE-Data/data/pool_structure.json)

---

## Deep-dive pages

- [C53.1 — Pool slot counts and the CREATE table](01-pool-slot-counts.md): all **13 pools** and
  their slot counts, strides, global VAs, and constructor VAs proven cold from `CPools::Initialise`
  at `0x5503A0`; the `CPool<T>` layout (20 bytes: `m_pObjects`, `m_byteMap`, `m_nSize`,
  `m_nFirstFree`, `m_bOwnsMemory`); the four main entity pools (CPed **140**, CVehicle **110**,
  CBuilding **13000**, CObject **350**).
- [C53.2 — Memory budget and safe expansion](02-memory-budget.md): per-pool heap cost
  (`count × stride` bytes for objects + `count × 1` bytes for byteMap); total budgeted pool
  memory; practical safe expansion ranges (CPed to ~250, CVehicle to ~200) before other
  subsystems strain; the streaming interaction that makes blind expansion dangerous.
- [C53.3 — CPool internals and handle encoding](03-cpool-internals-and-handles.md): the
  **byteMap bit protocol** — bit 7 set = free slot, bits 0–6 = handle version counter;
  the allocation loop (`m_nFirstFree` fast path, then linear scan); handle encoding
  `(slot_index << 8) | byteMap[slot]`; stale-reference protection via version increment
  on free; `New()`, `Delete()`, `GetAt()` call patterns.
- [C53.4 — Pool exhaustion and the streaming system](04-pool-exhaustion-and-streaming.md):
  **silent-fail semantics** — when a pool is exhausted, `New()` returns null and the entity
  is simply not created, with no crash; per-pool symptom table (empty world when CBuilding
  exhausted, invisible vehicles when CVehicle exhausted); streaming back-pressure when pools
  are full; strategies for detecting exhaustion in mods; safe pool-size planning.

---

## 53.0 The result first

| Pool index | Type | Count | Stride | Global VA | Tier |
|---|---|:--:|:--:|:--:|:--:|
| 1 | CPed | **140** | **1988 B** | `0xB74490` | ✅ |
| 2 | CVehicle | **110** | **2584 B** | `0xB74494` | ✅ |
| 3 | CBuilding | **13000** | **56 B** | `0xB74498` | ✅ |
| 4 | CObject | **350** | **412 B** | `0xB7449C` | ✅ |
| 5 | (unknown, 2500) | **2500** | TBD | `0xB744A0` | ✅ count / 🟡 stride |
| 6 | (unknown, 10150) | **10150** | TBD | `0xB744A4` | ✅ count / 🟡 stride |
| 0, 7–12 | (unknown) | 500, 500, 200, 64, 32, 64, 16 | TBD | `0xB7448C`–`0xB744BC` | ✅ count / 🟡 stride |

All counts proven from the `push-imm32` byte at the typed constructor call in `CPools::Initialise`;
strides for indices 5–12 from constructor register paths not yet decoded.

| Claim | Value | Tier |
|---|---|:--:|
| Initialisation function | `CPools::Initialise` @`0x5503A0` | ✅ |
| Shutdown function | `CPools::Shutdown` @`0x550F10` | ✅ |
| `CPool<T>` struct size | **20 bytes** | ✅ |
| Handle protocol | `(slot << 8) | byteMap[slot]` | ✅ |
| Free-slot indicator | bit 7 of `byteMap[i]` set | ✅ |

## 53.1 Why fixed pools, and what they cost

Thirty-three years of game-engine history have not dislodged the slab allocator for entity
management. The reasons are performance: a `CPool::New()` is at worst a linear scan of a byte
array, at best a single bit-test at `m_nFirstFree`. There is no heap fragmentation, no `malloc`
overhead, and the entire pool for a given type lives in contiguous memory — cache-friendly for any
pass that iterates all entities of one type (the render list, collision detection, AI task walks).

The cost is the fixed ceiling: **there is no pool resize**. When a CPed pool of 140 is full and
the wanted system tries to spawn a 141st ped, `CPool::New()` returns null and the spawn is silently
discarded. The game does not crash; it simply cannot add more peds. This "silent fail" semantic is
both a safety property (the game runs with fewer entities rather than crashing) and a modding hazard
(a pool-full condition looks like a spawn bug, not an out-of-memory error).

## 53.2 The four main pools and their memory cost

```
CPed:      140 × 1988 B =  278,320 B  ≈   272 KB  (byteMap: +140 B)
CVehicle:  110 × 2584 B =  284,240 B  ≈   278 KB  (byteMap: +110 B)
CBuilding: 13000 ×  56 B =  728,000 B  ≈   711 KB  (byteMap: +13000 B)
CObject:   350 ×  412 B =  144,200 B  ≈   141 KB  (byteMap: +350 B)
Total main pools:         ≈ 1,402 KB  ≈  1.4 MB
```

Plus pools 0 and 5–12 at unknown strides, but their combined count (~14,000 entities across 9
pools) contributes the bulk of the memory budget at whatever their strides are. The full SA process
heap at startup is approximately 280 MB (on 32-bit Windows with a standard 2 GB user-space limit),
of which the pools are a relatively small fraction — the texture and geometry streaming cache
dominates.

## 53.3 The CPool handle protocol, in one paragraph

Every entity in a pool is addressed by a **handle** — a 16- or 32-bit integer that encodes both the
slot index and a version counter. The encoding is `(slot_index << 8) | byteMap[slot_index]`, where
the low 7 bits of `byteMap` are the version and bit 7 is the free-flag. When a slot is freed, the
version counter increments — a handle held to a freed slot will no longer match the incremented
version and will be treated as stale, preventing a class of use-after-free bugs. This is the same
protocol [C4](../C4-CPool-Internals/C4-CPool-Internals.md) documents in detail; C53 proves the
slot counts that make the protocol meaningful.

---

## Key takeaways

- All 13 pools are allocated once by `CPools::Initialise` at `0x5503A0` and never resized; the
  slot counts are `push-imm32` arguments proven cold from the exe.
- The four main pools: CPed (**140** × 1988B), CVehicle (**110** × 2584B), CBuilding
  (**13000** × 56B), CObject (**350** × 412B) — these are the primary limit-patching targets.
- `CPool::New()` **silently returns null** on exhaustion — no crash, no error, just a missing
  entity. This makes pool-full conditions look like spawn bugs in mods.
- The handle protocol `(slot << 8) | byteMap[slot]` provides stale-reference protection via
  version increment on free — explained in [C4](../C4-CPool-Internals/C4-CPool-Internals.md).

**Continue:** [C53.1 — Pool slot counts and the CREATE table →](01-pool-slot-counts.md)


## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Key functions:** `CPools::Initialise` (0x5503A0), `CPools::Shutdown` (0x550F10)
- **Callers:** **1** call-site for `Initialise` (startup), **1** for `Shutdown` (exit).
- **Known bugs / gotchas:** silent null on pool exhaustion — spawn fails silently; stale handles via version wrapping after 127 frees on one slot.
- **Modding:** patch `push-imm32` counts in `CPools::Initialise` to raise pool ceilings; see [C54](../C54-Limits-Reference/C54-Limits-Reference.md) for patch bytes.
- **Performance:** contiguous slab layout is cache-friendly for entity iteration; `New()` is O(1) amortized via `m_nFirstFree` cursor.
