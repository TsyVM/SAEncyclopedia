# C22.4 — Zone runtime and gameplay integration

## How zones drive the running game

The zone tables (C22.1) and the radar grid (C22.2) are loaded once at startup. At runtime, multiple subsystems poll zone membership constantly to determine what should happen near the player. The zone system is not just a map label — it is a runtime dispatch key.

## Zone lookup: what calls it each frame

Every frame, the running game needs zone information for several purposes:

| Query | Caller | Why |
|---|---|---|
| "What zone is the player in?" | `CTheZones::GetZoneInfo` | Population density and type (C16.3) |
| "What zones does this entity overlap?" | `CWorld::FindWorldSector` | Streaming cell culling (C2/C31) |
| "Is this coordinate inside zone X?" | Mission scripts (C32) | Mission area-trigger conditions |
| "What radar zone is the player in?" | HUD radar system (C23) | Zone-name display in corner |
| "What pop-density applies here?" | `CPopulation::Update` | Ped and vehicle spawn rates (C16) |

The most frequent query is the streaming-cell overlap — it runs every entity position update.

## The GXT zone name connection

Zone names in the tables are GXT keys (C19). When the radar HUD (C23) displays the zone name in the lower-left corner, it calls `CText::Get(zone_name_key)` to retrieve the display string. This is why zone names are localized — the key is a fixed ASCII string in the zone table, but the display text is in `american.gxt`.

Modding zone display names requires only a GXT edit; changing zone boundaries requires editing the zone table (binary map data or IPL `zone` sections — C11.4).

## Gang territory zones

The gang territory system (C26.3) uses named zones to define gang-controlled areas. A sub-type of zone entries marks areas as belonging to specific gangs. When the player enters a gang zone, the gang-war proximity system (`CGangWars`) checks the zone's gang flag and spawns the appropriate gang peds from the gang's model group.

The zone boundary is the gang territory boundary — it is not drawn in-game for most zones (it is an invisible rectangle), but it determines which gang's territory count is affected by the player's actions there.

## The radar grid as a fast lookup

C22.2 established the radar grid as a 64×64 (or similar) flat array indexed by world coordinates. The HUD uses this grid for two things:
1. Radar zone label: which named zone cell the player is in
2. Radar blip coordinates: mapping world-space positions to screen-space radar positions

The grid lookup is a direct array index: `cell_x = (world_x + 3000) / cell_size`, `cell_y = ...`. This is O(1) regardless of how many zones exist — the grid pre-computes zone membership spatially. C22.3 showed how the `gridref` (the grid reference) indexes this flat array.

## Zone interaction with CStreaming

The streaming system (C2) uses a region-of-interest centered on the player to determine which models to load. This region is not a named zone — it is a radius around the player's position in world coordinates. However, named zones do interact with streaming in one indirect way: the population system (C16) uses zone information to select which ped and vehicle groups to populate the streaming region with.

## Interior zones

SA has multiple interiors (the casino, the gym, the prison) that are geometrically placed far from the main map (at high-Y or underground coordinates). Interior zones are named zones that map these positions to their displayed zone labels. When the player is in an interior, the zone system returns the interior zone name rather than the exterior zone that would correspond to the interior's map position.

The entry/exit transitions (CEntryExitManager — C30.1) handle the teleportation between exterior and interior coordinates. Zone lookup continues working correctly because the interior's position in the zone table is at the interior's actual world coordinate, not the corresponding exterior position.

## Modding zones

Adding new zones:
1. Add an IPL `zone` row: `zone_name, zone_type, x1, y1, z1, x2, y2, z2, interior_level, gxt_key, unk`
2. Add the display name to `american.gxt` under the `zone_name` key
3. Register it in `gta3.dat` or equivalent

Changing zone boundaries: edit the zone coordinates in the IPL `zone` row. The radar grid is rebuilt at load time from zone data, so any zone coordinate changes automatically update the radar grid.

**Previous:** [C22.3 — Gridref and the build tree](03-gridref-and-the-build-tree.md)  
**Up:** [C22 — Map Zones](C22-Map-Zones.md)
