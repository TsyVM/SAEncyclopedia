# C51.4 — The frame-loop subsystem sequence

## CGame::Process: the 81-call frame update

`CGame::Process` @`0x53BEE0` is called once per frame from the main loop. It calls 81 subsystem update functions in a fixed order. The order encodes both the update dependencies and SA's intended execution contract: systems that feed others run first.

## The frame sequence in rough phases

### Phase 1: Input

1. `CPad::Update` — read gamepad/keyboard state (C46)
2. `CMouseControl::Update` — mouse delta
3. Cheats check (global cheat code state machine)

**Why first:** input state must be fresh before any game logic reads it.

### Phase 2: Timing

4. `CTimer::Update` @`0x560D2E` — advance `ms_nTimeInMilliseconds`, compute `ms_fTimeStep` (C52)
5. `CWeather::Update` — advance day/night cycle, weather interpolation

**Why second:** `ms_fTimeStep` (C52) must be computed before any physics or animation reads it.

### Phase 3: Streaming

6. `CStreaming::Update` — process model requests, advance CdStream completions (C2)
7. `CStreaming::LoadAllRequestedModels` — promote demanded models to active IO (C1/C2)

**Why early:** models need to be in memory before entities can be placed or updated.

### Phase 4: World update

8. `CWorld::Process` (C5) — entity sector management, visibility determination
9. `CCamera::Process` (C39) — camera position, frustum computation
10. `CRenderer::BuildRenderList` (C40) — determine which entities are visible

**Why before physics:** collision and AI need to know which entities are in the world before acting on them.

### Phase 5: Physics and collision

11. `CPhysics::Update` — vehicle and object physics integration, `ms_fTimeStep`-scaled (C42/C47)
12. `CCollision::ProcessCollisions` — detect and resolve collisions (C6)
13. `CCarCtrl::GenerateVehicles` — vehicle density accumulator, spawn/despawn (C54)

### Phase 6: Population and AI

14. `CPopulation::Update` — ped spawn/despawn decisions (C26/C16)
15. `CPedAI::Update` (C41) — ped AI task processing
16. `CTheScripts::ProcessAllScripts` (C32) — script thread execution for this frame

**Why scripts late:** scripts frequently set up conditions that world/AI update acts on. Running scripts after world/AI means scripts see a fully-updated world state — they are not working on stale data.

### Phase 7: Audio and effects

17. `CAudioEngine::Update` (C20) — stream audio, spatial sound positioning
18. `CParticles::Update` (C21) — particle system advancement
19. `CTheScripts::DrawScriptSpheres` — debug visualizations, gizmos

### Phase 8: HUD and display

20. `CHud::Draw` (C23) — 2D HUD overlay (health bars, minimap, etc.)
21. `CFont::DrawFonts` (C19/C23) — queued text rendering

**Note:** this is the render preparation phase. The actual rendering of the 3D world happens after `CGame::Process` returns, in the renderer passes (C40). `CGame::Process` does not submit 3D draw calls itself.

## The critical ordering constraints for modders

Two constraints matter most for ASI mods:

### 1. "Before CTimer::Update" vs "after CTimer::Update"

A frame-tick hook that runs before `CTimer::Update` (step 4) sees the *previous frame's* `ms_fTimeStep`. A hook that runs after sees *this frame's* `ms_fTimeStep`. If the mod uses `ms_fTimeStep` for physics calculations, it must run after step 4.

### 2. Scripts run after world update

`ProcessAllScripts` runs after `CWorld::Process` and `CPedAI::Update`. This means:
- Scripts that spawn entities see those entities immediately (they were just updated)
- Scripts that query ped AI state see the AI's just-updated state
- Scripts that try to move a ped and then immediately read its position will see the *un-updated* position — the physics hasn't run yet in the *next frame*

This is a common source of "one-frame delay" bugs in mission scripts.

## The relation to CTimer and bIsFrozen (C52)

When `bIsFrozen` is true:
- `CTimer::Update` still runs but does not advance `ms_nTimeInMilliseconds` or `ms_fTimeStep`
- `CPhysics::Update` steps by ~0.0 (the frozen value ≈0.01)
- `CTheScripts::ProcessAllScripts` still runs — scripts are NOT frozen by `bIsFrozen`

This distinction matters: `bIsFrozen` freezes the physics simulation (C52.5) but not script execution. A script can therefore run during a "frozen" loading screen.

## CGame::Shutdown: the reverse order

`CGame::Shutdown` @`0x53C900` tears down subsystems approximately in reverse init order (C51.3). The key constraint is the same: a system that another system depends on must be shut down last. In practice, Shutdown calls each subsystem's `Shutdown`/`Free`/`Clear` method, frees all pool allocations, and releases D3D9 resources.

**Previous:** [C51.3 — Init order and subsystem dependencies](03-init-order-and-subsystem-dependencies.md)  
**Up:** [C51 — Executable Lifecycle](C51-Executable-Lifecycle.md)
