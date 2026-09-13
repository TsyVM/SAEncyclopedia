# C53.2 — Memory budget and safe expansion

## Baseline pool memory cost

These are the heap bytes allocated by `CPools::Initialise` for the four typed pools plus pool 0, at default slot counts:

| Pool | Count | Stride | Objects block | byteMap | Total |
|---|---|---|---|---|---|
| Unknown/20B (0) | 500 | 20 | 10,000 | 500 | **10,500** |
| CPed (1) | 140 | 1988 | 278,320 | 140 | **278,460** |
| CVehicle (2) | 110 | 2584 | 284,240 | 110 | **284,350** |
| CBuilding (3) | 13,000 | 56 | 728,000 | 13,000 | **741,000** |
| CObject (4) | 350 | 412 | 144,200 | 350 | **144,550** |
| Pools 5–12 (est.) | varies | TBD | ~500KB est. | ~14KB | **~514,000 est.** |
| **All pools total** | | | | | **≈ 1.97 MB** |

The five typed pools account for roughly 1.46MB. Pool memory is a small fraction of SA's total process footprint (~500–800MB at runtime), so raising pool counts within reason has negligible memory impact.

## Memory cost per additional slot

| Pool | Stride (bytes) | Cost per extra slot |
|---|---|---|
| CPed | 1988 | ~2KB/slot |
| CVehicle | 2584 | ~2.5KB/slot |
| CBuilding | 56 | ~56B/slot |
| CObject | 412 | ~412B/slot |

Adding 60 extra ped slots (140→200) costs 60 × 1988 = ~116KB. Adding 7,000 extra building slots (13,000→20,000) costs 7,000 × 56 = ~384KB. Both are trivial relative to available address space.

## 32-bit address space constraint

SA is a 32-bit process with a 2GB virtual address space limit (3GB with `LARGEADDRESSAWARE` — but the stock HOODLUM exe is not marked `LARGEADDRESSAWARE`). The practical usable range is typically 1.5–2GB before fragmentation causes allocation failures.

The stock process already uses approximately 500–800MB at runtime (game + streaming + audio + GFX). The pool expansions recommended below add at most a few MB — well within the available space.

**The risk is not raw memory size. It is OOM from the malloc call inside CPool<T>::Ctor if the requested block is too large or the heap is fragmented.** If `operator new` at `0x820595` returns null for a pool allocation, the subsequent pointer write crashes immediately. The safe maximums below are conservative enough that this should not occur on a standard machine.

## Known-safe maximum slot counts (community-verified)

These values have been validated by the SA modding community over many years and are consistent with the memory calculations above:

| Pool | Default | Safe maximum | Reasoning |
|---|---|---|---|
| CPed | 140 | **250** | ~220KB extra; ped AI loops scale O(n) but remain fast below ~300 |
| CVehicle | 110 | **160** | ~129KB extra; traffic spawner density cap (see C54) becomes the real limit before this |
| CBuilding | 13,000 | **20,000** | ~392KB extra; IPL loading loop is the practical bottleneck above ~25,000 |
| CObject | 350 | **1,000** | ~272KB extra; physics update loop scales O(n); tested stable at 1,000 |

## How to calculate and apply a safe increase

1. Determine stride: read the `imul reg, reg, N` in the pool constructor.
2. Calculate cost: `(new_count - old_count) × stride` bytes.
3. Verify total is reasonable: sum all pool costs; if total > 50MB extra, reconsider.
4. Apply by patching the `push-imm32` at the `evidence_va` for each pool, before `CPools::Initialise` executes.
5. Never decrease counts below default — scripts and the streaming system may rely on handle index ranges being stable.

## Pool exhaustion symptoms

Pool exhaustion is **silent** — no crash, no error message. The game simply fails to allocate the entity:

| Pool | Exhaustion symptom |
|---|---|
| CPed | Peds stop spawning; `CREATE_CHAR` opcode returns handle 0 |
| CVehicle | Vehicles stop spawning; `CREATE_CAR` returns handle 0 |
| CBuilding | Static objects in IPL are silently skipped during load; map looks incomplete |
| CObject | Script objects not created; pickups may disappear |

The only reliable way to detect pool exhaustion during development is to query `CPool::GetNoOfUsedSpaces()` (or the equivalent pool-count read: `m_nSize - count_of_free_slots`) via a trainer or memory viewer.

## The no-runtime-resize rule

`CPool<T>` allocates with `operator new` at construction time and never re-allocates. There is no `Resize()` method. If you need a larger pool than the default allows, you must patch the count before `CPools::Initialise` runs (called from `CGame::Initialise`). Post-startup patches to slot counts have no effect — `m_pObjects` and `m_byteMap` are already the wrong size.

**Up:** [C53 — Memory and Pool Architecture](C53-Memory-And-Pool-Architecture.md)  
**Next chapter:** [C54 — Limits Reference](../C54-Limits-Reference/C54-Limits-Reference.md)
