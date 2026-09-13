# C11.4 — The loader and runtime integration

## How CFileLoader reads IDE and IPL

`CFileLoader::LoadLevel` (called from `CGame::Initialise`, Phase 3 — C51.3) is the entry point for all world data loading. It reads `DATA/DEFAULT.DAT` (and level-specific `.dat` files), which contain directives like:
- `IDE data/maps/leveldes/leveldes.ide` — load this IDE file
- `IPL data/maps/leveldes/leveldes.ipl` — load this IPL file
- `TEXDICTION models/gta3.txd` — load this TXD

For each IDE file, `CFileLoader::LoadObjectTypes` is called. For each IPL file, `CFileLoader::LoadMapData` is called.

## LoadObjectTypes: IDE to CModelInfo

`LoadObjectTypes` parses each IDE section type and populates `CModelInfo` (C3):
- `objs` section → `CSimpleModelInfo` per entry
- `tobj` section → `CTimeModelInfo` (time-gated visibility variant)
- `weap` section → `CWeaponModelInfo`
- `hier` section → `CHierarchyModelInfo`
- `peds` section → `CPedModelInfo` (model ID + PEDTYPE + statistics)
- `cars` section → `CVehicleModelInfo`
- `2dfx` section → `C2dEffect` data attached to the model (C10)

Each IDE row's model ID maps to a slot in the `CModelInfo` table. The model name string is used to look up the corresponding file in the streaming archive (C1) — the streaming system maps model names to archive positions.

## LoadMapData: IPL to CWorld

`LoadMapData` parses each IPL section and places entities in the world:
- `inst` section → each row calls `CWorld::Add` with a `CBuilding` entity
- `cars` section → parked vehicle spawn points stored in a parked-car table
- `enex` section → entry/exit transitions (door links between zones)
- `pick` section → scripted pickup positions (seed data for `CPickups` — C29)
- `grge` section → garage definition (passed to `CGarages` — C29)
- `zone` section → named zone rectangles for `CTheZones` (C22)

The `CBuilding` entities placed by `inst` rows are static (non-physics, non-scripted). They are placed once at load time and never moved. They consume slots from the `CBuilding` pool (C53) — the 13,000-slot building pool is exactly this use case.

## The cross-check: 185/185 and IDE definitions

C11.3 established that all 185 distinct ped models in `pedgrp.dat` resolve against `peds.ide`. This works because the IDE files and the pedgrp data are compiled from the same model name namespace. The streaming system (C1) maps model names to archive positions; the model name is the coupling point across all three systems.

## Binary IPL: the large-map optimization

For very large maps (particularly `gta3.img`'s binary `.ipl` files), SA uses a binary IPL format rather than text. The binary IPL packs the same data (model ID, position, rotation, LOD flag) more densely. `CFileLoader::LoadBinaryIPL` reads the binary format; `LoadMapData` handles both text and binary depending on the file's magic header.

## LOD chaining

Every `inst` row has a LOD index — a reference to the lower-detail version of the same prop. The LOD chains are resolved at load time by `CFileLoader::SetupLodLinks`. The LOD system (C3's LOD model management) uses these chains to pick the correct detail level based on camera distance.

## Modding the map: IPL and IDE workflow

To add a new static prop to the world:
1. Add the model (`model_name.dff` + `model_name.txd`) to `gta3.img` or a custom `.img`
2. Add an IDE row to an existing or custom IDE file: `MODEL_ID, model_name, texture_txd, draw_dist, flags`
3. Add an IPL `inst` row with the desired position and rotation: `MODEL_ID, model_name, interior_id, x, y, z, rx, ry, rz, rw, lod_id`
4. Register the IDE and IPL in `gta3.dat` (or a custom level dat file)

The model ID must be unique and within the valid model ID range (0–20,000 for standard objects). Collision (`model_name.col`) must also be added for the prop to have physical presence.

**Previous:** [C11.3 — What placements prove](03-what-placements-prove.md)  
**Up:** [C11 — IDE and IPL](C11-IDE-And-IPL.md)
