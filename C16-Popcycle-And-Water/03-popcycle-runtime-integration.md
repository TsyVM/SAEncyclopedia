# C16.3 — How popcycle.dat drives the runtime population

## The two-file design: data and assignment

`popcycle.dat` (C16.2) and `pedgrp.dat`/`cargrp.dat` (C26) work as a two-level system:
- `popcycle.dat` answers "which group label is active in zone Z at time T?"
- `pedgrp.dat` answers "which ped models are in group label G?"

The runtime population system reads both. `CPopulation` (or its equivalent — class name ⏳) looks up the current zone and time-of-day to find the active group label, then samples `pedgrp.dat`'s array for that label to decide which ped model to spawn.

## Zone and time-of-day lookup

The game world is divided into zones (C22). Each zone has a name (`LSFI`, `SFRI`, etc.) that `popcycle.dat` uses as a row key. At runtime, the population system:
1. Determines the player's current zone from the map-zone system (C22)
2. Reads the current hour from `CTimer::ms_nTimeInMilliseconds` (C52) — converted to an in-game hour via the day/night cycle
3. Looks up `popcycle.dat`'s row for that zone and column for that hour
4. The cell value is a `POPCYCLE_GROUP_*` label — the index into `pedgrp.dat`'s group array

The "broken invariant" (C16.2§5) — 41 rows where a group label is out of the expected range — causes these rows' lookups to return the wrong group, effectively using the wrong population for those zone+time combinations. The effect is subtle (peds from a neighboring group appear) rather than a crash.

## The spawn decision: from group label to individual ped

Once the active group label is resolved:
1. `CPopulation::AddPed` (⏳) is called when a ped spawn slot opens
2. The function samples the group's model array (21-slot flat array at `0xC0F358`, indexed by group) 
3. A model index is returned (using the repeats-as-frequency heuristic — C26.4§5)
4. The model is requested from the streaming system (C1/C2)
5. Once streamed, a `CPed` is allocated from the ped pool (C53) and placed at the spawn point

## Day/night transitions and hysteresis

When the in-game hour changes, the active group label for each zone may change. The population system does not immediately replace all existing peds — that would cause visible "teleporting" of pedestrians. Instead:
- Existing peds continue using the previous group's behavior until they despawn naturally (by moving out of the streaming cell)
- New spawns use the new group label
- The transition is gradual, driven by the natural ped lifecycle

This is why a zone transition from "business day" to "night" results in a gradual shift in ped types over several in-game minutes rather than an instant swap.

## The water.dat runtime role

The water quads and triangles decoded in C16.1 are placed in the world by `CWaterLevel` (class name ⏳). The 301 quads and 6 triangles define the SA water surface. At runtime:
- `CWaterLevel::GetWaterLevel` (used by physics, vehicles, and peds) queries this table to determine if a point is below the water surface
- `CWaterLevel::RenderWater` draws the water plane using the quad/triangle vertices
- The X/Y bounds of −3000 to +3000 match the world boundary (C22), confirming the water plane covers the entire playable area

## The −3000 to +3000 world boundary

The exact correspondence between the water plane's bounds and the world boundary (C22) is not coincidental. `CWorld` enforces a hard limit on entity positions — an entity that moves past ±3000 on X or Y is at the edge of the playable world. The water plane being exactly that size means there is no "gap" between playable land and the water edge; the entire non-land area has a water surface.

This also means mods that expand the world boundary (by patching the ±3000 clip) will have water only up to the original boundary — the water quads are data-defined and not auto-extended.

## Modding implications

### Changing zone populations at specific times

To change who appears in Los Santos at 2am: find the `LSFI` (or relevant zone name) row in `popcycle.dat`, locate the column for hour 2 (0-indexed), and change the value to the desired `POPCYCLE_GROUP_*` index. The group itself is defined in `pedgrp.dat` (C26).

### Adding new time-of-day variation

`popcycle.dat` has one row per zone and 24 columns (one per hour). To add day/night differentiation to a zone that currently has the same group all day, simply change the night-hour columns to a different group label. No exe patching required.

### The 8.5% broken-invariant rows

The 41 out-of-range rows (C16.2§5) can be corrected by replacing their values with valid group indices. The effect is predictable: those zone+time combinations will use the correct group rather than whatever group the out-of-range index happens to resolve to.

**Previous:** [C16.2 — popcycle.dat and its broken invariant](02-popcycle-dat.md)  
**Up:** [C16 — Popcycle and Water](C16-Popcycle-And-Water.md)
