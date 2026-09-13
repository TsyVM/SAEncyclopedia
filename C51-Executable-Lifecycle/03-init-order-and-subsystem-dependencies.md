# C51.3 — Init order and subsystem dependencies

## Why init order is a hard constraint

`CGame::Initialise` calls 142 subsystem initialisers in a fixed order. This order is not arbitrary — it is determined by dependency: a subsystem cannot initialise before the subsystem it depends on. The streaming system (C1) must be ready before collision (C6) can load. The model stores (C3) must be ready before the world (C5) can place entities.

Understanding the init order answers a practical question for modders: **when can I safely hook each subsystem?** An ASI that hooks at `DllMain` runs before `CGame::Initialise`, so the target subsystem may not yet be initialised. An ASI that hooks `WinMain` can run code at any point in the startup sequence.

## The init sequence in outline

The 142 callees of `CGame::Initialise` @`0x53BC80` fall into rough phases:

### Phase 1: Foundation (pre-model systems)

1. `CGame::Initialise` starts
2. `RenderWare` engine initialisation (C50) — `RwEngineInit`, `RpWorldPluginAttach`, plugin attaches
3. D3D9 device creation (C44)
4. Memory allocator setup
5. `CFileLoader::Initialise` — the file system and IMG archive registration (C1.2)
6. `CdStreamInit` — IO thread start (C1)
7. `CTxdStore::Initialise` — texture dictionary store (C3)
8. `CColStore::Initialise` — collision store

### Phase 2: Data loading

9. `CModelInfo::Initialise` — model info table (C3)
10. `CStreaming::Initialise` — streaming system tables and buffer (C2)
11. `CPools::Initialise` @`0x5503A0` — all 13 entity pools allocated (C53)
12. `CTheZones::Init` — map zone system (C22)
13. `CPathFind::Initialise` — path network (C12)
14. `CTimeCycle::Initialise` — timecycle table (C15)
15. `CWeather::Initialise` — weather system

### Phase 3: World construction

16. `CFileLoader::LoadLevel("DATA/DEFAULT.DAT")` — begins loading all world data (C11)
    - Calls `CFileLoader::LoadObjectTypes` for IDE files
    - Calls `CFileLoader::LoadMapData` for IPL files (C11)
    - Loads `ped.dat`, `pedgrp.dat` (C26), `object.dat` (C25), `handling.dat` (C13)
17. `CWorld::Initialise` (C5) — scene graph setup
18. `CPopulation::Initialise` — ped spawning system (C26, C16)

### Phase 4: Game system setup

19. `CTheScripts::Init` — SCM bytecode loading, root thread creation (C32)
20. `CPad::Initialise` — input (C46)
21. `CPickups::Initialise` — pickup pool (C29)
22. `CGarages::Init` — garage system (C29)
23. `CVehicleRecording::Init` — recording-file table (C34)
24. `CConversations::Clear` — conversation arrays (C35)
25. `CTimer::Initialise` — game timer (C52)
26. ... (remaining 117 callees follow similar patterns)

## Key dependency chains for modders

| If your hook targets... | You need to be after... |
|---|---|
| Any entity creation | `CPools::Initialise` (step 11) |
| Model loading/replacing | `CModelInfo::Initialise` (step 9) |
| Script execution | `CTheScripts::Init` (step 19) |
| Pickup creation | `CPickups::Initialise` (step 21) |
| Streaming buffer size | `CStreaming::Initialise` (step 10) — BEFORE, not after |
| Pool size patching | `CPools::Initialise` (step 11) — BEFORE, not after |

The last two items are critical: **pool and streaming buffer patches must happen before the init function that allocates them, not after.** An ASI that patches the CPed pool size after `CPools::Initialise` has already run will not work — the memory is already allocated. Hook the init function itself, patch the incoming constant, then call the original.

## The CRT entry → WinMain → CGame::Initialise chain

```
CRT entry @0x824570
    │
    ▼
WinMain @0x748C50 (⏳ — address approximate)
    │
    ├─ RenderWare engine init
    ├─ Window creation, D3D9 setup
    │
    ▼
CGame::Initialise @0x53BC80
    │  (142 callees in dependency order)
    ▼
... game loop starts ...
```

The `CRT entry` at `0x824570` is the PE `AddressOfEntryPoint` — confirmed by C51's `derive_lifecycle.py`. This is where execution begins after Windows loads the executable. The CRT startup code runs before `WinMain`, initializing static C++ objects and the MSVC runtime.

**ASI loader position:** most ASI loaders (`d3d9.dll` proxy) intercept during DLL loading, before `WinMain`. This is the safest injection point for hooks that must be in place before `CGame::Initialise` runs. Hooks installed at `DllMain` run before step 1 above.

**Previous:** [C51.2 — The frame loop and shutdown](02-frame-loop-and-shutdown.md)  
**Continue:** [C51.4 — The frame-loop subsystem sequence →](04-frame-loop-subsystem-sequence.md)
