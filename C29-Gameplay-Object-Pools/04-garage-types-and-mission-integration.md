# C29.4 — Garage types and mission integration

## The type byte at +0x4C

C29.2 established that the type byte at garage `+0x4C` is written by `ChangeGarageType` and compared by `ActivateGarage` against value 11 (`cmp [eax+0x4C], 0xB`). The full type enum is partially decoded from the methods:

| Type value | Decoded name | Evidence |
|---|---|---|
| 0 | Inactive / generic | Default / `ActivateGarage` fallback |
| 1 | Pay 'n' Spray (respray) | `IsCarSprayable`, `AllRespraysCloseOrOpen` |
| 2 | Save/Safehouse | Door state machine loop |
| 3 | Mission pay garage | `SetTargetCarForMissionGarage` |
| 4 | Bomb shop | `ChangeGarageType` switch branch |
| 5 | Tuning garage | `ChangeGarageType` switch branch |
| 9 | Hideout storage | `CountCarsInHideoutGarage` |
| 11 | Trigger-open (cutscene / mission) | `ActivateGarage` `cmp ..., 0xB` |

Types not listed are ⏳ — the `ChangeGarageType` switch has additional branches not fully traced. The key principle: `+0x4C` is the gameplay category that determines which logic runs when the door transitions.

## The mission-garage mechanism

`SetTargetCarForMissionGarage` (`0x00447C40`) writes a vehicle pointer at garage `+0x40`:

```
01561699  imul eax, eax, 0xD8       ; index * 216
0156169F  add  eax, 0x96C048        ; + pool base
015616A2  mov  dword ptr [eax+0x40], ecx  ; store target vehicle pointer
```

This means offset `+0x40` in the 216-byte garage record is a **CEntity/CVehicle pointer** — the vehicle the mission wants to bring into this garage. The mission script sets a garage to type 3 (mission pay), sets the target vehicle pointer, and the garage logic then watches for that specific vehicle to enter its trigger zone.

The absolute address confirmation: `0x96C048 + 0x40 = 0x96C088` — this address is readable and patchable to redirect a mission garage's target vehicle at runtime.

## The respray logic: `IsCarSprayable`

`CPickups::IsCarSprayable` rejects a fixed list of model IDs from the Pay 'n' Spray. This method contains a long comparison chain checking the vehicle's model ID against the excluded list (police, emergency vehicles, boats, aircraft, unique mission vehicles). The exclusion is model-ID-based, not PEDTYPE-based — this is why adding a new resizable vehicle to the game requires editing this exclusion list if you want it to be resprayable.

`AllRespraysCloseOrOpen` walks all 50 garage slots looking for type 1 (respray) garages and opens or closes all of them simultaneously — used when the player gets a wanted level while inside a garage.

## The flags byte at +0x4E

The flags byte encodes the garage's enabled/disabled state:
- `DeActivateGarage` → `or [eax+0x4E], 2` (sets bit 1 = disabled)
- `ActivateGarage` → `and [eax+0x4E], 0xFD` (clears bit 1 = enabled)

These are exact inverses. Additional bits in the flags byte are ⏳ — bit 1 is the activation flag, other bits' roles require further disassembly.

## The TriggerMessage mechanism (most-called method)

`CGarages::TriggerMessage` has the highest call count (17 callers) in the class. It displays an on-screen garage notification (e.g., "RESPRAY SUCCESSFUL" or "GARAGE FULL"). The text is stored in a **500-byte region at `0x96C00C–0x96C020`** (name region, ~24 bytes) and a broader buffer for the message text. This is the only garage method that directly touches the GXT text system (C19).

## The hideout inventory: `CountCarsInHideoutGarage`

For safehouse garages (type 9), `CountCarsInHideoutGarage` counts how many stored vehicles are currently inside the garage's trigger zone. This feeds into the "garage is full" check before a new vehicle is admitted. The count is per-garage, not global — each safehouse garage has its own independent vehicle count derived from entity proximity.

## The door state machine and gameplay connection

The door state machine (C29.2§3) connects to gameplay at two points:

1. **Player entry trigger:** the garage's trigger zone (a CColBox or similar proximity zone in the map data) sends a "player entered" event. This is what transitions the door from `0 closed → 3 opening`. The map data sets up the trigger zone; `CGarages` handles the transition.

2. **Mission condition gates:** many missions check `IsGarageOpen` or `IsGarageClosed` as a condition for progression. A mission script might wait for `IsGarageClosed` before triggering the "car is safe" cutscene. The door state byte is the runtime value these checks poll.

## The 50-slot limit and modding

The garage pool is a **flat static array** (not a `CPool<T>`) at `0x96C048`. Unlike the main entity pools (C53), there is no CPool<T> wrapper, no byteMap, and no generational handle system. The count global at `0x96C024` is the live count of defined garages; `GetGarageNumberByName`'s loop bound is this count.

To raise the limit beyond 50, you must patch:
- The array end address in `Shutdown` (`0x96EAC8`)
- The count limit in `GetGarageNumberByName` if it has a hardcoded bound
- Any other method that uses an absolute end address

The static-array design means the memory cost is fixed at `50 × 216 = 10,800 bytes` regardless of how many garages are actually defined. Unlike the entity pools, this cannot be changed at init time — it is part of the `.bss` or `.data` section and is fixed by the PE image layout.

**Previous:** [C29.3 — Pickup types and the full pickup lifecycle](03-pickup-types-and-lifecycle.md)  
**Up:** [C29 — Gameplay Object Pools](C29-Gameplay-Object-Pools.md)
