# C52.1 — CTimer globals and the timestep pipeline

## The static block

All CTimer state lives in a contiguous block of statics starting near `0xB7CB40`. These are the proven globals:

| VA | Name | Type | Role |
|---|---|---|---|
| `0xB7CB48` | `bIsFrozen` | `uint8_t` | Non-zero = time is paused (loading screen, cutscene) |
| `0xB7CB49` | `bGameStarted` | `uint8_t` | Tested alongside `bIsFrozen`; both must allow time for the timestep to advance |
| `0xB7CB50` | `game_FPS` | `float` | Measured FPS, written once per frame by `CTimer::Update` (`fstp` at `0x560F86`) |
| `0xB7CB54` | `ms_fTimeStep_prev` | `float` | Previous frame's clamped step, saved before the new one is computed (`0x560D83`) |
| `0xB7CB58` | `ms_fTimeStepNonClipped` | `float` | Raw elapsed step — unclamped. Renderers use this for interpolation. |
| `0xB7CB5C` | `ms_fTimeStep` | `float` | **The integration scalar.** Clamped max = 2.0. Referenced 984× in `.text`. |
| `0xB7CB84` | `ms_nTimeInMilliseconds` | `uint32_t` | Accumulated game-time in milliseconds — incremented every frame |

All addresses confirmed cold by Capstone. The 984-reference count for `0xB7CB5C` was measured by a full `.text` scan; it is the most-referenced single variable in the binary.

## The clamp

`CTimer::Update` at `0x560D2E` performs the clamp in three steps:

```
1. If bIsFrozen || !bGameStarted:
       ms_fTimeStepNonClipped = 0x3C23D70A   (≈ 0.008 — a near-zero frozen value)

2. Clamp raw step against minimum (rdata floor near 0x858B3C)
       if raw_step < floor: use floor

3. Clamp against maximum:
       fcomp [0x858C14]          ; 0x858C14 = 2.0 (proven)
       ; if raw_step > 2.0: use 2.0
       fstp  [0xB7CB5C]          ; write ms_fTimeStep
```

The `fcomp` / `fnstsw` / `test ah, 0x41` / `jp` sequence at `0x560DA2–0x560DAD` is the exact proof: the float at rdata `0x858C14` reads as `2.0` (`0x40000000`) in the flat-file scan, and the write to `0xB7CB5C` immediately follows.

## The unit of ms_fTimeStep

The unit is **50ms per 1.0**:

| Frame rate | Frame time | `ms_fTimeStep` |
|---|---|---|
| 25 fps | 40.0 ms | 0.800 |
| 30 fps | 33.3 ms | 0.667 |
| 60 fps | 16.7 ms | 0.333 |
| 120 fps | 8.3 ms | 0.167 |
| Stall (>100 ms) | — | **2.0 (clamped)** |

Evidence for the 50ms unit: at freeze state the fixed value `0x3C23D70A ≈ 0.008` corresponds to less than 1ms — consistent with the frozen world advancing by nearly nothing. The max clamp of 2.0 corresponds to exactly 100ms, which is a reasonable stall-protection window for a 30fps target.

## What bIsFrozen controls

When `bIsFrozen` is 1, the timestep is forced to the near-zero frozen value — the world continues to be rendered (so the engine is not truly paused from the render side) but every physics and gameplay accumulator advances by ≈0 per frame. This is the loading-screen state. When `bIsFrozen` is 0 and `bGameStarted` is 1, the full variable-rate pipeline runs.

## Key takeaways

- `CTimer::Update` (`0x560D2E`) runs once per `CGame::Process` frame and writes all six CTimer statics.
- `ms_fTimeStep` (`0xB7CB5C`) is the **only** integration variable — 984 references in `.text` mean nothing in the engine bypasses it.
- The clamp max is **2.0** (rdata `0x858C14`), proven from the `fcomp` write site at `0x560DA2`.
- Unit is **50ms = 1.0**, so doubling the frame rate halves `ms_fTimeStep` proportionally.

**Continue:** [C52.2 — Frame rate and physics stability →](02-frame-rate-and-physics.md)
