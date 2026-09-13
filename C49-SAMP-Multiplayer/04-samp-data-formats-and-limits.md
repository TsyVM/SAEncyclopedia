# C49.4 — SA:MP data formats, limits, and modding considerations

## SAMP.img: a VER2 archive of added content

`SAMP.img` is a standard VER2 IMG archive (C1.1) with 1,673 directory entries. SA:MP uses the base game's own streaming system to load its added content — it does not have a separate asset loader. This means:
- SA:MP assets consume image slots in the 32-slot image table (C1.2)
- SA:MP model IDs (18631 and above per `SAMP.ide`) consume streaming-info entries in the streaming-info table (C31)
- SA:MP assets are subject to the same pool limits as base-game assets (C54)

**The model-ID boundary:** `SAMP.ide` adds objects at ID 18631+. The base game's model IDs run up to approximately 20,000. SA:MP's range avoids collision with standard SA model IDs, but mods that add their own high-ID content must stay clear of SA:MP's range to avoid conflicts.

## SAMPCOL.img: collision in a sub-archive

`SAMPCOL.img` is a COL archive (C6 format — the same binary collision format the base game uses). SA:MP adds collision meshes for its added objects here. The game's collision system loads and processes these using the same code path as base-game collision — there is no special SA:MP collision code.

## The .saa archive format

SA:MP ships audio content in `.saa` files (SA Audio Archive). The internal format is ⏳ — not yet traced to byte level by this project. The files are processed by BASS (`bass.dll`) at runtime. From a modding perspective: `.saa` files require SA:MP's own audio loader and cannot be modified using standard SA audio tools.

## Player and vehicle limits in SA:MP

SA:MP imposes its own limits on top of the base game's pool limits (C53/C54):

| Entity | SA:MP limit | Base-game pool limit | Notes |
|---|---|---|---|
| Players | 500 (server config) | CPed pool: 140 | SA:MP desyncs ped models from the pool |
| Vehicles | 2000 (server config) | CVehicle pool: 110 | Same decoupling |
| Objects | 400 per player | CObject pool: 350 | SA:MP has a separate object system |

SA:MP can exceed the base-game pool limits because it does not use `CPool<T>` (C53) for its networked entities in the same way the base game does. SA:MP maintains its own entity state tracking and only creates base-game objects (CPed, CVehicle instances) for entities near the local player. Entities beyond the base-game pool limit simply have no local representation — they are tracked in SA:MP's network state but don't exist in the game world on that client.

## Modding SA:MP-aware mods

A mod running alongside SA:MP must account for:

1. **Hook ordering:** if the mod also hooks `EndScene` or `GetAsyncKeyState`, it must chain SA:MP's hook pointer rather than overwriting it. The safest approach is to hook early (before SA:MP's `DllMain` runs) so SA:MP's hook chains onto the mod's hook.

2. **Model ID namespace:** custom mod content should use model IDs below 18631 (or well above, with server configuration to extend the range). Collisions cause assets to load the wrong model for SA:MP's objects.

3. **Pool pressure:** SA:MP's added vehicles and peds near the player add to pool pressure. A mod that maximizes entity counts should account for SA:MP's per-client entity creation when estimating pool usage.

4. **Audio:** SA:MP audio (via BASS) bypasses the base game's audio channel system (C20). Mods that modify game audio have no effect on SA:MP's voice or server-streamed audio.

## The closed-source boundary

SA:MP is closed-source. Everything in C49 is derived from:
- String evidence in `samp.dll` (version strings, RakNet identifiers, packet-type names)
- The data files it ships (`SAMP.img`, `SAMP.ide`, `SAMPCOL.img`) — all in base-game formats
- Community documentation (SA:MP wiki, plugin-sdk SA:MP extensions) — corroborated with file evidence, not taken at face value

No claims are made about SA:MP's internal implementation beyond what these sources directly support. The sync-packet taxonomy, model-ID range, and hook points (D3D9, input) are all confirmed by string evidence; internal data structures are ⏳.

**Previous:** [C49.3 — The SA:MP hook layer in depth](03-the-hook-layer-in-depth.md)  
**Up:** [C49 — SA:MP](C49-SAMP-Multiplayer.md)
