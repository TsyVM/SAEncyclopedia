# C52.3 — FPS unlock and limit-breaking guide

## What "FPS unlock" actually means

SA's in-engine frame limiter is a busy-wait loop that targets 25fps. Removing it allows the renderer to run as fast as the hardware allows. The frame limiter can be removed with a single byte patch or via the `notarget` SA Limit Adjuster flag — this page is not about that. This page is about what happens to the game engine when the frame rate rises above the 25–30fps range it was tuned for.

## The three layers of frame-rate dependence

**Layer 1 — Timestep-scaled code (safe).** 984 references to `ms_fTimeStep` (`0xB7CB5C`). These are correct — velocity, damage, timers, AI state machines. They work at any frame rate because the per-tick delta shrinks proportionally.

**Layer 2 — Sub-step integrators (safe).** The vehicle physics at ~`0x54D000` and ped physics at ~`0x547000` internally clamp their per-call timestep to 2.0 and run multiple sub-steps. These are also frame-rate independent; they just do more sub-steps at low frame rates.

**Layer 3 — Non-scaled per-frame constants (broken at high FPS).** These are the hazard. Examples confirmed in the SA community:

| Issue | Description | Approximate FPS threshold |
|---|---|---|
| Ragdoll explosion | Per-frame impulse not multiplied by `ms_fTimeStep` | ~45fps+ |
| Nitro over-boost | Nitro force accumulator incremented per-frame | ~60fps+ |
| Swim speed increase | Swim force missing timestep multiply | ~60fps+ |
| Some cutscene pops | Legacy animation position deltas | varies |

These require patching the specific force/position code, not the timer system.

## What to patch and what to leave alone

### Remove the frame limiter (safe)

The 25fps busy-wait loop is inside the main game loop. To raise the cap, either:
- Use SA Limit Adjuster's `Frame Limiter` setting
- Or find and NOP the QueryPerformanceCounter spin in the main loop

This is safe. `ms_fTimeStep` will simply take smaller values at higher frame rates — all 984 timestep-scaled paths adjust automatically.

### Patch the ms_fTimeStep max clamp (safe, with care)

**VA:** rdata `0x858C14` — the float constant compared by `fcomp` at write site `0x560DA2`

**Current value:** `2.0` (bytes: `00 00 00 40`)

**Effect of increasing this value:** Allows larger `ms_fTimeStep` when a frame stalls longer than 100ms. On a slow machine, this prevents physics from slowing down during hitches — the engine simulates more world-time per frame. On a fast machine it is irrelevant (frames will be shorter than 100ms).

**Effect of decreasing this value:** Reduces the effective physics simulation rate at low frame rates. If you set the clamp below 0.667 (= 30fps), the game's physics will slow down at 30fps.

**Safe range:** Leave at 2.0. Increases to 3.0 (150ms stall protection) are harmless. Decreasing below 0.800 (40ms / 25fps) breaks low-end machine physics.

### Do not patch ms_fTimeStep directly at runtime

`ms_fTimeStep` (`0xB7CB5C`) is written every frame by `CTimer::Update`. Patching the write target has no lasting effect — the next `CTimer::Update` call overwrites your value. The correct place to intervene is the clamp constant at rdata `0x858C14` (for the max) or the per-frame update function itself.

If you need to force a fixed-rate simulation (60fps sim at 144fps render), the approach is:

1. Hook `CTimer::Update` at `0x560D2E`
2. After it runs, if `ms_fTimeStep` < threshold, accumulate the remainder and inject it on the next sub-step

This is complex and not necessary for most mods. It is the approach some FPS-fix mods use.

## Practical guide: enabling a higher frame rate without breaking physics

### Minimum viable approach (raises FPS, accepts some physics glitches)

1. Disable the frame limiter.
2. Cap rendering at 60fps externally (RTSS, driver cap).
3. At 60fps, `ms_fTimeStep ≈ 0.333` — all timestep-scaled physics is correct.
4. Ragdolls and nitro will still be slightly fast. If this matters, add:

### FPS-fix compatible approach (full physics correction)

Use `SA FPS Unlocker` or `SilentPatch` from the SA community, which patches the known non-scaled constants. These mods have been independently verified against the same pool of disassembly this encyclopedia documents.

If writing your own fix from scratch, the targets are:
- The ragdoll impulse code: find write sites near ped velocity update that do not multiply by `ms_fTimeStep`
- The nitro force code: similarly, force application paths in vehicle physics that use fixed float constants instead of `fld [0xB7CB5C]`

Finding these is a follow-up probe task — each is a pattern of `fadd` or `fmul` with a `dword ptr [rdata_const]` that is **not** followed by `fmul [0xB7CB5C]`.

## Summary: the safe modification table

| Target | VA | Current value | Safe to change? | Notes |
|---|---|---|---|---|
| Frame limiter | main loop | 25fps busy-wait | Yes — remove it | Standard practice |
| ms_fTimeStep max clamp | rdata `0x858C14` | 2.0 (100ms) | Yes — can increase | Raising protects slow machines |
| ms_fTimeStep min clamp | rdata `0x858C18` | 0.005 | Leave alone | Prevents divide-by-zero downstream |
| ms_fTimeStep global | `0xB7CB5C` | run-time float | Do not patch directly | Overwritten every frame |
| CTimer::Update VA | `0x560D2E` | — | Hook-friendly | Clean intercept point |

**Previous:** [C52.2 — Frame rate and physics stability](02-frame-rate-and-physics.md)  
**Up:** [C52 — CTimer and the Game Loop](C52-CTimer-And-Game-Loop.md)  
**Next chapter:** [C53 — Memory and Pool Architecture](../C53-Memory-And-Pool-Architecture/C53-Memory-And-Pool-Architecture.md)
