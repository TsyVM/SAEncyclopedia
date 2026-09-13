# C53.4 — Pool exhaustion and the streaming system

## Pool exhaustion is always silent

SA never crashes on pool exhaustion — it fails silently. The allocator returns null; the calling code checks for null and bails out. This is by design (the PS2 couldn't afford exception handling), but it means the game degrades in ways that look like bugs rather than like an obvious limit being hit.

## Exhaustion symptoms per pool

### CPed pool exhausted (140 slots full)

`CPopulation::AddPed` calls `CPool::New` on the ped pool. If it returns null:
- No ped is spawned. The game continues as if `AddPed` was never called.
- `CREATE_CHAR` SCM opcode returns handle 0.
- A script that does `CREATE_CHAR` and then does `IS_CHAR_NEAR_POINT_3D(0x0000, ...)` will test handle 0 — which always passes the in-use check — but `GET_CHAR_COORDINATES(0x0000, ...)` will return world origin (0,0,0).
- From the player's perspective: the sidewalks empty out. No new peds spawn anywhere in the city. Pre-existing peds (those in the pool before it filled) continue to exist.

**Modding implication:** If your mod creates a large number of NPCs via SCM or a plugin, keep a per-frame count of live peds you own and don't exceed the pool minus the headroom CPopulation needs (~10 slots for ambient peds). The pool is shared between player-created and ambient peds.

### CVehicle pool exhausted (110 slots full)

Same pattern as peds. `CCarCtrl::GenerateOneRandomCar` silently fails. `CREATE_CAR` returns handle 0. Traffic disappears; the player's own vehicle is in the pool and holds one slot permanently while driven.

**Note:** The player always occupies one ped slot and the player's vehicle always occupies one vehicle slot. The "usable for spawning" headroom is effectively one less than the pool count.

### CBuilding pool exhausted (13,000 slots full)

`CWorld::Add` is called by the IPL loader when placing static map geometry into the world. If the building pool is full when an IPL cell tries to place an object, that object is silently omitted from the world. The terrain and roads (which are a different system) continue to render, but static props, walls, and structures that were placed via IPL disappear. The effect looks like incomplete map loading.

This is the most common exhaustion case for custom modded maps — a dense interior or custom city block with thousands of static props routinely exceeds the 13,000 default.

**Streaming interaction:** the CBuilding pool is filled and drained dynamically as the player moves through the world. When a streaming cell is loaded (`CStreaming::RequestModelStream`), its IPL data is processed and each entity added to the CBuilding pool. When the cell is unloaded, those entities are removed from the pool (their slots freed). This is why the CBuilding pool can never be treated as fully-occupied — at any given moment, the number of live building slots is the number of entities in the currently-loaded streaming radius, not all entities in the entire map.

The 13,000 default is sized for the maximum number of static entities that can be visible at once from any point in the game's designed playable space. Mods that increase the number of entities per cell, or increase the streaming radius, can exhaust the pool even though the total map entity count is lower than 13,000.

### CObject pool exhausted (350 slots full)

Dynamic objects (breakable props, ragdoll-attached objects, weapon pickups with collision, `CREATE_OBJECT` entities) fill this pool. When full:
- `CREATE_OBJECT` and `CREATE_OBJECT_NO_OFFSET` return handle 0.
- Weapon pickups that have collision (shotgun ammo, etc.) fail to create; the pickup appears but has no collision.
- Script-spawned props do not appear.

Objects in this pool also include CObject instances created internally by the game (glass shards from broken windows, wreckage), so the pool is never fully available for script use.

## The streaming system's use of pools

The streaming system (`CStreaming`) manages which IPL (map) and IMG (model) data is loaded based on player position. It interacts with pools in two ways:

### 1. CBuilding allocation during cell load

When a streaming cell becomes active, `CStreaming::ProcessLoadedModels` calls `CWorld::Add` for each static entity in the cell. `CWorld::Add` allocates a CBuilding slot for each entity. This allocation failure is silent — if the pool is full, the entity is not placed.

The streaming system has no notion of "pool full" — it does not retry or queue failed placements. If a modded map's dense cells are loaded in a zone where the pool is already heavily used, entities will silently not appear.

### 2. CBuilding deallocation during cell unload

When a streaming cell is removed from memory (`CStreaming::RemoveModel`), `CWorld::Remove` is called for each entity in the cell, which frees its CBuilding slot. This is how the pool recycles — steady-state play cycles slots continuously.

The implication for modders: if you create a static entity via `CREATE_OBJECT` (which allocates a CObject slot, not CBuilding), it persists in the pool until you delete it — it is not subject to streaming removal. If you create many persistent static objects, you are consuming CObject slots permanently.

## Detecting pool exhaustion at runtime

### Method 1 — Query `m_nSize` and the free count

Each pool's `CPool<T>` header at its global VA contains `m_nSize` (+8) and `m_byteMap` (+4). Reading the byteMap to count bytes with bit 7 clear (in-use) gives the live count. Compare to `m_nSize` to get the headroom.

For the CPed pool at `0xB74490`:
```
pool_ptr = *(uint32_t*)0xB74490       // pointer to CPool header
m_nSize  = *(int32_t*)(pool_ptr + 8)  // = 140
m_byteMap = *(uint8_t**)(pool_ptr + 4) // pointer to 140 flag bytes
live_count = count of bytes where (byte & 0x80) == 0
headroom   = m_nSize - live_count
```

A trainer or ASI that monitors this in real-time can display a pool usage HUD.

### Method 2 — Hook CPool::New and count returns

Hook the pool allocation function and count how many times it returns null. A spike of null returns from the CPed pool confirms ped pool exhaustion.

### Method 3 — CIplDef entity count (for CBuilding)

Before patching the CBuilding pool count, count the entities in your IPL files. Each `inst` entry in each `.ipl` file that loads within the player's streaming radius represents one CBuilding slot. A rough upper bound is achievable from the IPL data alone.

## Safe headroom guidelines

These are the recommended minimum free slots to maintain at runtime for stable gameplay:

| Pool | Min free slots | Reason |
|---|---|---|
| CPed | 10 | Ambient ped spawner needs headroom to cycle |
| CVehicle | 5 | Traffic spawner needs room to swap vehicles |
| CBuilding | 500 | Streaming transitions load new cells before unloading old |
| CObject | 30 | Internal glass/wreckage creation needs headroom |

When designing a mod that creates entities, budget against `(pool_count - headroom)` rather than the full pool count.

**Previous:** [C53.3 — CPool internals and handles](03-cpool-internals-and-handles.md)  
**Up:** [C53 — Memory and Pool Architecture](C53-Memory-And-Pool-Architecture.md)  
**See also:** [C54 — Limits Reference](../C54-Limits-Reference/C54-Limits-Reference.md) — patching the pool counts
