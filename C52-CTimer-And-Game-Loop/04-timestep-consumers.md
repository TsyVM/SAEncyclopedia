# C52.4 — How ms_fTimeStep propagates through the engine

## The 984-reference picture

Every rate in the engine is multiplied by `ms_fTimeStep` (`0xB7CB5C`). The count of 984 cross-references in `.text` was measured by a full disassembly scan; no other single variable has anywhere near this reference count. This page categorises those 984 uses by subsystem and explains how to identify the specific code pattern.

## The multiplier instruction pattern

In x87 floating-point code (SA is 100% x87, no SSE in the main sim path), the per-frame timestep multiply looks like:

```asm
fld   dword ptr [some_rate_variable]
fmul  dword ptr [0xB7CB5C]     ; multiply by ms_fTimeStep
fstp  dword ptr [some_rate_variable]
```

Or the combined load-multiply form:

```asm
fld   dword ptr [0xB7CB5C]
fmul  [constant_rate]
fadd  [velocity_accumulator]
fstp  [velocity_accumulator]
```

Any rate-per-frame computation that **lacks** an `fmul dword ptr [0xB7CB5C]` somewhere in its call chain is frame-rate dependent. Finding these is how FPS-fix mods locate the broken paths.

## The six consumer categories

### 1. Entity velocities and positions (largest group)

`CPhysical::ProcessControl` and its vehicle/ped overrides integrate velocity → position by multiplying by `ms_fTimeStep` at the final position-update step. This makes world movement frame-rate independent. The canonical pattern is in the velocity integrator:

```asm
; velocity already in FPU stack
fmul dword ptr [0xB7CB5C]   ; scale by timestep
fadd dword ptr [entity.pos_x]
fstp dword ptr [entity.pos_x]
```

Every moveable entity in the world — peds, vehicles, objects, projectiles — goes through this path.

### 2. Timer countdowns (task timers, script timers, wait timers)

AI task timers, wander timers, combat state transition timers — all decrement by `ms_fTimeStep` per frame. The pattern is `fsub dword ptr [0xB7CB5C]` into a timer float. When the timer reaches zero, the event fires.

**Note on script timers:** The SCM virtual machine's built-in `WAIT` opcode does NOT use `ms_fTimeStep` directly — it uses `ms_nTimeInMilliseconds` (`0xB7CB84`), which is a wall-clock accumulator. This means script `WAIT` delays are real-time (unaffected by FPS) while physics timers are simulation-time (scale with timestep). This distinction matters for mods that control timing-sensitive events.

### 3. Damage and health decay

Damage-over-time effects (fire, drowning, wanted-level health drain) multiply their per-tick rate by `ms_fTimeStep`. A fire that deals `5.0 damage/tick` actually applies `5.0 × ms_fTimeStep` per frame, so it takes the same wall-time to kill the player at 30fps or 60fps.

### 4. Animation and morph blending

`CAnimBlendAssociation` blend weights are interpolated toward their targets by an amount proportional to `ms_fTimeStep`. This keeps blend speeds (e.g., the speed of an aim-blend or a door-open animation) frame-rate independent.

### 5. Camera smoothing

The camera lag system in `CCamera` interpolates the camera position toward the target position using `ms_fTimeStep`. The lag constant (from camera settings) is the fraction of remaining distance to close *per tick* — so at 60fps the camera closes half as much distance each frame but twice as often, ending up at the same position per wall-second.

### 6. Particle and effect timers

`CParticleSystemMgr` updates emitter lifetimes and spawn rates by `ms_fTimeStep`. Particle speed integration also uses it. This is why particles look roughly the same at different frame rates.

## The non-scaled paths: where FPS instability comes from

The known broken paths are **not** in the categories above — they are discrete impulse applications, not continuous integrations. The pattern of a broken path is:

```asm
fld   [constant_force]    ; a fixed float constant
; MISSING: fmul dword ptr [0xB7CB5C]
fadd  [velocity]          ; applied directly per frame, not per second
fstp  [velocity]
```

Finding these requires scanning the vehicle and ped physics functions for `fadd` or `fmul` against rdata float constants that are NOT immediately followed or preceded by an `fmul [0xB7CB5C]`. Known locations (not exhaustive):

- **Nitro force application:** inside CVehicle's nitro boost path, a constant force is added to velocity per frame. Fix: multiply the constant by `ms_fTimeStep` before adding.
- **Ragdoll impulse:** inside CRagDoll's initial velocity application, the rag is given a fixed speed independent of elapsed time.
- **Swim force:** CPed's swim state applies a constant forward force per frame.

Each of these is a one-instruction fix — insert `fmul dword ptr [0xB7CB5C]` before the `fadd`. The reason they exist is that SA was developed at 25fps and these forces were tuned by feel at that rate; at 2× the frame rate they apply 2× as often, doubling their apparent magnitude.

## ms_fTimeStep vs ms_fTimeStepNonClipped

Most engine consumers use `ms_fTimeStep` (the clamped value at `0xB7CB5C`). The unclamped value `ms_fTimeStepNonClipped` (`0xB7CB58`) is used by a smaller number of consumers, mainly:

- **Rendering interpolation:** the renderer may interpolate entity positions between simulation states using the unclamped delta to avoid the slight stutter the 2.0 clamp introduces during hitches
- **CCamera interpolation:** the camera lag uses the unclamped value so the camera doesn't get "stuck" behind the clamped physics advance during a loading hitch

This distinction matters for mod code: if you are computing a visual-only interpolation (a smooth camera, a HUD element), prefer `ms_fTimeStepNonClipped`. If computing a physics or gameplay rate, use `ms_fTimeStep` (the clamped value) — the clamp is there to protect you from stall events.

## ms_nTimeInMilliseconds: the wall-clock accumulator

`ms_nTimeInMilliseconds` at `0xB7CB84` is a `uint32_t` that counts milliseconds of game time elapsed since startup. It is incremented each frame by a value derived from the actual wall-clock delta (not the clamped `ms_fTimeStep`). Its uses:

- **SCM `WAIT` opcode:** scripts wait until `ms_nTimeInMilliseconds >= target_ms`, so script waits are real-time, not physics-time.
- **CTaskSimpleWaitUntilPedIsOutCar and similar tasks:** use millisecond targets for AI timeouts.
- **Replay system:** timestamps replay packets in absolute milliseconds.

The distinction from `ms_fTimeStep` is important: a physics timer counting down via `ms_fTimeStep` will run slowly if the game is paused (bIsFrozen = 1, timestep ≈ 0), but a script WAIT will still advance because `ms_nTimeInMilliseconds` counts real elapsed time.

## Practical guide for mod authors

**If your mod needs a per-frame rate that must be frame-rate independent:** multiply by `ms_fTimeStep`. Example: a per-frame health regen of 1 HP/second is `1.0 / 50.0 × ms_fTimeStep` per frame (since 1.0 = 50ms, and 1 tick per 50ms gives 20 ticks/second, so the per-second value divided by 20 gives the per-tick rate to multiply by `ms_fTimeStep`).

More practically: treat `ms_fTimeStep` as a fractional multiplier. At 60fps it is ≈ 0.333; at 30fps ≈ 0.667. If your rate at 30fps gives the right feel, divide that rate by 0.667 and you have the "per-tick-at-1.0" constant — then multiply at runtime by `ms_fTimeStep` to make it frame-rate independent.

**If your mod needs real-time durations (wait N seconds):** use `ms_nTimeInMilliseconds`. Store the current value, add your duration in milliseconds, wait until the live value exceeds the target.

**Previous:** [C52.3 — FPS unlock guide](03-fps-unlock-guide.md)  
**Continue:** [C52.5 — Pause and freeze states →](05-pause-and-freeze-states.md)
