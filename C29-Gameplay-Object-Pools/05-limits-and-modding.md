# C29.5 — Limits and modding the gameplay pools

## Two pool architectures in one chapter

C29 covers two fixed-size containers that share no implementation code:

| Pool | Architecture | Size | Base address |
|---|---|---|---|
| CPickups | Flat static array, 32-byte records | 620 × 32 = 19,840 bytes | `0x9788C4` |
| CGarages | Flat static array, 216-byte records | 50 × 216 = 10,800 bytes | `0x96C048` |

Neither uses `CPool<T>` (C53). There is no byteMap, no generational handle version counter, no `m_bOwnsMemory` flag. Both are compiled into the PE's `.data` or `.bss` section and their sizes are fixed by the executable layout. Raising either limit requires patching every site in the code that computes an end-of-array address from the base address.

## CPickups: what constrains the 620 limit

The 620 limit is defined by three things working together:

1. **Array base address:** `0x9788C4` — this is where the PE loader placed the static array.
2. **Array end address:** `0x9788C4 + (620 × 32) = 0x9788C4 + 0x4D80 = 0x97D644` — used in loop termination checks in `FindPickUpForThisObject` and `Init`.
3. **Handle encoding:** `(counter << 16) | slot_index` — `slot_index` is a 16-bit value in the low half, so the theoretical maximum is 65,535 slots. The 620 limit is not a handle encoding constraint; it is a memory layout decision.

To raise the pickup limit, a mod must:
1. Allocate a new block of `(new_count × 32)` bytes (heap or a private static in the ASI)
2. Patch `0x9788C4` references to the new base address
3. Patch `0x97D644` (and any similar end-address constants) to `new_base + (new_count × 32)`
4. Patch any hardcoded `620` loop bounds

**Known ASI approaches:** the `Limit Adjuster` plugin for SA patches the pool using VirtualProtect to write the new constant over the original. This is the recommended approach — direct memory writes after `VirtualProtect` to `PAGE_READWRITE`.

## CGarages: what constrains the 50 limit

The garage count is stored in a global (`0x96C024`). The array base is `0x96C048` and each record is 216 bytes. The practical limits come from:

1. **GetGarageNumberByName:** walks from garage 0 to `[0x96C024]`, comparing name strings. If the count global is simply incremented beyond 50 without a new array, the walk reads past the static array into adjacent BSS/data memory — undefined behavior, likely a crash.
2. **Shutdown:** uses a hardcoded end address (`0x96EAC8 = 0x96C048 + 50 × 216`). This would not free the extra memory.

Unlike the pickup pool, the garage pool is written to rarely — only at map load. Raising it is feasible but requires careful end-address patching.

## What breaks at the limit: per-pool failure modes

### CPickups at 620

- `GenerateNewOne_WeaponType` returns immediately (silent fail) — no crash
- The scripted pickup (mission objective) simply does not appear in the world
- The player may be unable to complete the mission if the objective pickup can't spawn
- Weapon drops from killed NPCs are also blocked once the pool is full
- **Detection:** poll the pool's occupation count with the byteMap scan equivalent (for pickups: count slots where `record[i * 32 + 0x18] != 0`)

### CGarages at 50

- `GetGarageNumberByName` fails to find new garages added beyond position 50
- Scripts calling `SET_GARAGE_ACTIVE` or `OPEN_GARAGE` for extra garages get index -1 back
- All garage-triggered events (respray, save, mission-vehicle entry) for extra garages silently fail
- **Detection:** `GetGarageNumberByName` returning -1 for a known garage name

## Patching CPickups with an ASI

The same pattern used in C54 for entity pools applies here:

```cpp
// Raise pickup pool to 1200
#include <windows.h>

DWORD old;
const DWORD kPickupBase = 0x9788C4;
const DWORD kPickupEnd_site1 = 0x97D644; // FindPickUpForThisObject loop end
const int   kNewCount = 1200;
const DWORD kNewEnd = kPickupBase + (kNewCount * 32);

void PatchPickupPool()
{
    // Patch end-of-array constants
    VirtualProtect((void*)kPickupEnd_site1, 4, PAGE_EXECUTE_READWRITE, &old);
    *(DWORD*)kPickupEnd_site1 = kNewEnd;
    VirtualProtect((void*)kPickupEnd_site1, 4, old, &old);
    // ... patch additional sites similarly
}
```

The base address cannot be moved with this approach — you would need to patch all references to `0x9788C4` and allocate a new buffer. Most limit-adjuster mods instead use a separate allocation and pointer-redirect pattern for this reason.

## Interaction with the entity pools (C53)

CPickups wraps a `CObject` for each active pickup. This means:
- Every live pickup occupies **one CObject pool slot** (C53's 1000-slot `CObject` pool)
- Filling the pickup pool to near-620 simultaneously fills ~620 CObject slots
- The effective pickup limit is `min(620, available_CObject_slots)`
- A mod raising the pickup pool to 1200 without also raising the CObject pool will hit the CObject limit first

This interaction is the most common invisible limit in SA modding: the pickup pool appears to be 620 but is actually gated by the smaller CObject pool if many other objects are loaded.

## Interaction with the script system (C32)

Handles to pickups and garages are passed through the script system as integer variables (C32). The `GENERATE_PICKUP` opcode creates a pickup and stores the 32-bit handle in a script variable. The `GET_PICKUP_COORDINATES`, `IS_PICKUP_COLLECTED`, etc. opcodes read the handle back. The handle validation (`GetActualPickupIndex`) is what prevents use-after-free when a pickup is collected and the slot reused.

For garages, SCM opcodes use a garage index (not a handle) — the `GetGarageNumberByName` function returns the index, which is then stored in a script local and used for all subsequent `*_GARAGE_*` opcodes. This means there is no version counter for garages: if garage index 3 is somehow reused (not possible in normal gameplay since garages are map-static), the script would silently operate on the wrong garage.

## Summary: what modders control

| What | How | Limit |
|---|---|---|
| Number of pickups | Patch end-address constants in exe | 620 base; CObject pool is the real gate |
| Pickup types in the world | Authoring: SCM/ASI spawns | Any type 1–14 valid |
| Garage slot count | Patch array size and count global | 50 base |
| Garage types | `ChangeGarageType` call / map edits | 0–11 types decoded |
| Ring-buffer recency window | Patch ring size at `0x978628` | 20 entries base |
| Pickup rotation speed | Hook `CPickups::Update` | No static constant; done in Update logic |

**Previous:** [C29.4 — Garage types and mission integration](04-garage-types-and-mission-integration.md)  
**Up:** [C29 — Gameplay Object Pools](C29-Gameplay-Object-Pools.md)
