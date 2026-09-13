# C51.2 — The frame loop and shutdown

Once `CGame::Initialise` returns ([C51.1](01-startup-and-init.md)), the game enters its loop and stays there
until you quit. Each iteration is two halves — an ordered **update** and a **render** — and the order of the
update is where the game's causality lives. This page reads that order and the teardown.

## The update: `CGame::Process` @`0x53BEE0`

`CGame::Process` is the per-frame update hub — `derive_lifecycle.py` measures **81** direct calls from it. The
subsystem-update order (from `gta-reversed`) is fixed and meaningful:

```
CPad::UpdatePads          input           (C46) — read the devices FIRST
CStreaming::Update        streaming       (C1)  — bring in/evict models for where we are
CCutsceneMgr::Update      cutscenes
CTheZones::Update         zones           (C22)
CCover::Update            AI cover
CAudioZones::Update       audio zones     (C20)
CClock::Update            time            (C15)
CWeather::Update          weather         (C15/C16)
CTheScripts::Process      mission scripts (C18/C32) — run the SCM threads
CCollision::Update        collision       (C6)
CTrain::UpdateTrains      / CHeli::UpdateHelis
CDarkel / CSkidmarks / CGlass / CCreepingFire / CSetPieces
CWanted::UpdateEachFrame  wanted level    (C41)
CPopulation::Update       spawn/despawn   (C26) — populate the world LAST
```

Then, after `Process` returns, the [C40](../C40-Render-Pipeline/C40-Render-Pipeline.md) render passes run
(`ScanWorld` → `ConstructRenderList` → the draw passes), and the frame is presented.

## Order is causality

The value of reading this list is that it explains the game's timing behaviour:

- **Input is first**, so a mission script (`CTheScripts::Process`, later in the list) that checks a key sees
  *this* frame's press — controls feel immediate.
- **Streaming is second**, so by the time the render list is built ([C40](../C40-Render-Pipeline/C40-Render-Pipeline.md),
  after `Process`) the models for the current position are resident — which is also why fast travel can
  out-run streaming and show the low-detail pop-in [C40](../C40-Render-Pipeline/C40-Render-Pipeline.md)
  describes.
- **Scripts run before collision and physics**, so a script that teleports the player this frame has the
  world react to the new position the same frame.
- **Population is last**, so newly-spawned peds/vehicles get their first update next frame — a deliberate
  one-frame delay that avoids acting on half-constructed entities.
- **Events are queued** ([C41](../C41-Ped-AI-Tasks-Wanted/C41-Ped-AI-Tasks-Wanted.md)): a gunshot seen this
  frame enters a ped's event group and is *responded to* next frame, because the task tree is walked from the
  queue — the one-frame reaction delay players feel in combat.

So "why does the game do X a frame later?" is answered by *where X sits in this list relative to what triggers
it*. The order is a dependency sort, exactly like init ([C51.1](01-startup-and-init.md)) but for the steady
state.

## Update / render split

The frame is cleanly **update then render**: `CGame::Process` mutates world state (positions, health, spawns),
and only afterwards does `CRenderer` ([C40](../C40-Render-Pipeline/C40-Render-Pipeline.md)) read that state to
draw. Nothing in the render passes changes gameplay state; nothing in `Process` issues draw calls. This is why
the [C39](../C39-Camera/C39-Camera.md) camera (updated in the tail of `Process`) and the
[C40](../C40-Render-Pipeline/C40-Render-Pipeline.md) render list (built after `Process`) always agree — the
matrix is settled before the renderer reads it.

## Shutdown: `CGame::Shutdown` @`0x53C900`

Quitting calls `CGame::Shutdown`, which tears the subsystems down — freeing pools, closing streaming, shutting
audio, releasing RenderWare — broadly in reverse of the init order ([C51.1](01-startup-and-init.md)), because
teardown has the mirror dependency (you cannot free the pools while entities still reference them). Then
RenderWare closes (`RwEngineStop`/`RwEngineClose`) and control returns up through `WinMain` to the CRT exit.

## Frame pacing — a variable timestep

San Andreas runs a **variable timestep**, not a fixed one. `CTimer` publishes a per-frame **`ms_fTimeStep`**
at `0xB7CB5C`, and it is the single most-multiplied scalar in the engine — `derive_openitems.py` counts it
referenced **984** times in `.text`. Every rate in the game (velocities, timers, animation advance, damage
decay) is multiplied by `ms_fTimeStep`, so the world advances by real elapsed time each frame rather than a
fixed tick. To keep the physics stable when the frame rate drops, `CTimer` keeps two values:

| Global | VA | Role |
|---|---|---|
| `ms_fTimeStep` | `0xB7CB5C` | the **clamped** per-frame step (what physics/gameplay use) — 984 refs |
| `ms_fTimeStepNonClipped` | `0xB7CB58` | the **raw** elapsed step (uncapped) |
| `game_FPS` | `0xB7CB50` | measured frame rate |
| `bSkipProcessThisFrame` | `0xB7CB89` | skip this frame's update (loading/hitch) |

So the pacing model is: measure elapsed time → clamp it (the clip between `ms_fTimeStep` and
`ms_fTimeStepNonClipped`) so a long frame cannot tunnel objects through walls → multiply every rate by the
clamped step. This is why San Andreas *speeds up* at very high frame rates on modern hardware for some
physics (the clamp has an upper bound tuned for ~25–30 fps): rates that assume `ms_fTimeStep ≈ 1` per tick
over-advance when frames are tiny. The `derive_openitems.py` `variable_timestep` check asserts the 984-ref
dominance of `ms_fTimeStep`. This closes the earlier "⏳ frame pacing" item.

## Open items

- ⏳ The exact **clamp bounds** on `ms_fTimeStep` (the min/max the clip enforces).
- ⏳ The **loading-screen / state-machine** around gameplay (front-end vs in-game vs cutscene states).
- ⏳ The precise **init/shutdown order** of all 142 subsystems (this page gives the dependency shape, not the
  full list).

## Key takeaways

- The frame is **update (`CGame::Process`, 81 calls) then render ([C40](../C40-Render-Pipeline/C40-Render-Pipeline.md))**;
  the update order is fixed: input → streaming → world/weather → scripts → collision → AI → population.
- **Order is causality**: input-first means immediate controls, population-last and queued events mean
  one-frame reaction delays — every "a frame later" behaviour is explained by position in this list.
- `CGame::Shutdown` (`0x53C900`) tears down in reverse; the clean update/render split is why the camera and
  renderer always agree on the frame's state.

**Continue:** [back to the C51 hub →](C51-Executable-Lifecycle.md) · or [C40 — Render Pipeline](../C40-Render-Pipeline/C40-Render-Pipeline.md) (the render half of the frame).
