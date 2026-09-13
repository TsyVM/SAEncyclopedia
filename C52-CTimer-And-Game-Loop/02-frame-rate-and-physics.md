# C52.2 — Frame Rate and Physics Stability

> **The one-sentence version:** the variable timestep is **correct** for all 984 code paths that
> multiply `ms_fTimeStep` before advancing — these are frame-rate-independent by construction — but
> a small set of paths (ragdoll impulses, nitro force, the swimming accumulator, some cutscene
> deltas) bypass the multiply and produce FPS-dependent behaviour, which is why SA physics feels
> different at 60fps on unpatched installations.

**Subsystem category:** Engine substrate — frame-rate analysis
**Depends on:** [C52.1](01-timer-and-timestep.md) (`ms_fTimeStep` address and unit)
**RE status:** Documented — the five failure paths identified from string/behavioural evidence;
sub-step clamp addresses confirmed; nitro/swim force paths 🟡
**Confidence:** ✅ for the 984-reference scaling standard, the clamp anti-tunnel function, and
sub-step write sites at `0x54D8EA`/`0x54D96E` · 🟡 for the exact instruction addresses of the five
FPS-dependent paths (behaviourally confirmed, not individually traced)

---

## 1. Why the variable timestep is (almost always) correct

The game's time model is a simple theorem: if every rate is expressed as `value_per_50ms` and every
accumulator advances by `rate × ms_fTimeStep` each frame, then regardless of the actual frame rate,
the game simulates the same real-world time. At 30fps (`ms_fTimeStep ≈ 0.667`) a vehicle travels
`v × 0.667` per frame; at 60fps (`ms_fTimeStep ≈ 0.333`) it travels `v × 0.333` per frame — but
there are twice as many frames per second, so the total distance per second is the same.

The **984 references** to `ms_fTimeStep` across `.text` are the evidence that this discipline was
enforced. It is the highest-reference-count variable in the binary — every subsystem that advances
state references it. The canonical assembly pattern for a compliant accumulator (C52.4):

```asm
fld   dword ptr [rate_variable]
fmul  dword ptr [0xB7CB5C]     ; ms_fTimeStep at 0xB7CB5C
fadd  dword ptr [accumulator]
fstp  dword ptr [accumulator]
```

Any accumulator that **lacks** this `fmul [0xB7CB5C]` pattern somewhere in its update chain advances
by a frame-count-proportional amount, not a real-time-proportional amount.

---

## 2. The clamp: anti-tunnel guard, not a physics tick rate

The maximum clamp of **2.0** (proven from rdata `0x858C14` — `fcomp dword ptr [0x858C14]` at
`0x560DA2`) limits the maximum `ms_fTimeStep` value regardless of how long a frame actually took.
This serves two distinct purposes:

### 2.1 Anti-tunnelling

If the game stalls (disk read, debugger break, driver interrupt), the actual elapsed time might be
500ms — but a 500ms integration step would move fast-moving objects so far that they skip through
collision geometry entirely (the "tunnelling" problem). The 2.0 cap limits the advance to 100ms
(approximately two real 30fps frames' worth), beyond which the stalled time is simply dropped. The
game "skips" slightly rather than exploding.

### 2.2 It is NOT a rate limiter

The clamp is a *maximum*, not a target. At 120fps the timestep is `≈0.167`; at 240fps it is
`≈0.083`. The clamp never activates at these rates — it only triggers when the frame time **exceeds**
100ms (roughly below 10fps). A common misconception is that the clamp "locks physics to 30fps" —
it does not; it only prevents the physics from advancing more than 100ms in a single step.

---

## 3. The sub-step layers: independent from the outer clamp

C52.1 establishes that there are write sites to `0xB7CB5C` other than `CTimer::Update`:

| Address | Subsystem | Purpose |
|---------|-----------|---------|
| `0x54D8EA` | `CAutomobile` physics | Sub-step clamp inside vehicle integrator |
| `0x54D96E` | `CAutomobile` physics | Second vehicle sub-step clamp |
| `0x5474D7` | `CPed` physics | Sub-step clamp inside ped integrator |

These write their own clamped sub-step values to `0xB7CB5C` during multi-step integration passes.
They are **independent** of the outer `CTimer::Update` clamp — the outer clamp limits the total
frame advance; the inner clamps limit each sub-step's advance. When a vehicle is moving very fast,
the inner sub-step loop runs multiple passes with smaller timesteps to prevent the vehicle from
tunnelling through walls within a single frame.

The sub-step and outer clamp values are different in principle: the outer clamp (`0x560DA2`) fires
at 2.0 (100ms); the inner clamps fire at their own thresholds. Both write to the same global
`0xB7CB5C` because the inner clamps temporarily replace the outer timestep for the duration of
the sub-step, then the outer value is restored.

---

## 4. Where the variable timestep fails: five FPS-dependent paths

The following paths were confirmed to produce FPS-dependent behaviour on the retail executable
(behavioural evidence — exact addresses are 🟡 pending individual instruction tracing):

### 4.1 Ragdoll impulse application

When a ped enters ragdoll (shot, hit by a car, falling), an initial impulse is applied to the
ragdoll body. This impulse is applied as a **per-frame constant**, not scaled by `ms_fTimeStep`. At
60fps the impulse fires once but the frame is half-length, so the rag doll body moves less than
intended in that first frame — but because it fires once and the physics thereafter are timestep-
scaled, the practical effect is that ragdolls appear to "snap" faster at high frame rates (the
initial impulse fires at full strength but the dampening is timestep-scaled, creating an asymmetry).

### 4.2 Nitro force accumulation

The nitro boost system applies an acceleration force to the vehicle each frame. This force lacks
the `fmul [0xB7CB5C]` multiply. At 60fps, the nitro fires twice as often as at 30fps with
the full force magnitude each time — producing approximately double the acceleration benefit.
This is the best-known high-FPS advantage in competitive SA gameplay.

### 4.3 Swimming speed accumulator

The swimming force accumulator in the ped physics path does not scale by `ms_fTimeStep` in the
main locomotion path. At 60fps, swimming speed is approximately proportional to frame rate — the
swim force is applied twice as many times per second at the full per-call magnitude.

### 4.4 Some cutscene animation deltas

Legacy cutscene position deltas (some cutscene scripts authored early in development) apply
position offsets as per-frame values rather than per-second values. At 60fps these cutscenes
run at double the expected animation speed. Modern CLEO-authored cutscenes that use the proper
SLERP/lerp with timestep scaling are not affected.

### 4.5 The gear-shift frame counter (C42.4)

Documented in detail in [C42.4](../C42-Vehicle-Physics/04-physics-integration-and-timestep.md): the
gear-shift timer is a frame counter, not a time accumulator. At 60fps, gears shift at half the
intended real-world duration — 2× faster — producing the well-known "60fps acceleration anomaly"
where vehicles hit higher speeds faster on unpatched installations.

---

## 5. The FPS-unlock mod approach: render fast, simulate at 30fps

The dominant FPS-unlock strategy in the SA modding community is to **decouple simulation from
rendering**: render at 60fps (or 144fps, or unlimited) but keep `ms_fTimeStep` clamped to the 30fps
value (`≈0.667`), effectively forcing the physics to simulate as if running at 30fps while rendering
interpolated states faster.

### 5.1 How it works

```asm
; Conceptual patch (implementation varies by mod):
; After CTimer::Update runs and writes ms_fTimeStep:
mov  eax, [0xB7CB5C]         ; read the just-written timestep
fldz                          ; or: load 0.667 constant
; ...conditional clamp to 0.667...
fstp [0xB7CB5C]              ; override with 30fps equivalent
```

The render system reads object positions for rendering — but positions are derived from velocities
(which are timestep-scaled). By keeping the physics at 30fps while rendering at 60fps, the mod
achieves smooth rendering without FPS-dependent physics behaviour.

### 5.2 What this fixes and what it does not fix

| Behaviour | Fixed by 30fps-clamp? | Reason |
|---|:--:|---|
| Velocity/position integration | ✅ | Always scaled; clamping ms_fTimeStep makes them match 30fps |
| Gear-shift speed | ✅ | Frame counter, but frame time is now 30fps-equivalent |
| Nitro force | ✅ | Applied per-frame, but effective ms_fTimeStep = 0.667 |
| Ragdoll impulse | ✅ (partial) | Initial impulse unchanged; dampening now at 30fps rate |
| Visual smoothness | ❌ | The purpose of the mod — rendering faster than physics |

The 30fps-clamp approach does not fix the visual smoothness issue (it renders the same simulation
at a higher frame rate). True physics smoothness at 60fps requires fixing each non-scaled path
individually — a far larger engineering effort.

---

## 6. The 30fps physics "sweet spot"

The SA engine was tuned at approximately **25–30fps**. The evidence:

- At 25fps: `ms_fTimeStep = 0.800` (slightly above the 0.667 nominal)
- At 30fps: `ms_fTimeStep = 0.667` (the "design" rate by the 50ms unit choice)
- At 20fps: `ms_fTimeStep = 1.000` (exactly the "nominal" — the unit was chosen with PS2 target in mind)

The physical constants (spring stiffness, force magnitudes, drag coefficients) were tuned so that
at 20–30fps the vehicles feel as intended. The suspension spring stability boundary
(C42.4 §5: `k × Δt² / m < 2`) is satisfied for all stock vehicle parameters at Δt ≈ 0.667.

At 120fps (Δt ≈ 0.167), the stability boundary is satisfied with a much larger margin — springs are
more numerically stable at higher frame rates, not less. The FPS-dependent *perception* problems
(nitro advantage, faster gear shifts) are distinct from the numerical stability issues.

---

## 7. The interaction with `ms_fTimeStepNonClipped`

C52.1 documents two variants of the timestep: `ms_fTimeStep` (clamped, `0xB7CB5C`) and
`ms_fTimeStepNonClipped` (raw, `0xB7CB58`). A few rendering and interpolation systems use the
non-clamped variant:

| System | Uses clamped? | Why |
|--------|:--:|---|
| Physics / velocities | ✅ | Must not get infinite advance from a stall |
| AI timers and countdowns | ✅ | Same anti-tunnel reason |
| Particle system lifetimes | ✅ | Particle death timers must be bounded |
| Camera interpolation (smooth-follow lerp) | ❌ (uses non-clamped) | The camera should track the *real* elapsed time for smooth visual interpolation |
| Some animation blend factors | ❌ (uses non-clamped) | Blend should match real wall time for non-gameplay animations |

The non-clamped value can exceed 2.0 in theory (during long stalls), so systems that use it must
either be tolerant of large values (camera lerp overshoots and snaps back naturally) or be
independently guarded.

---

### Key takeaways

- The variable timestep makes physics **frame-rate-independent by construction** for all 984
  correctly-scaled code paths — this is the design, and it works.
- The 2.0 clamp (rdata `0x858C14`, proven from `0x560DA2`) is an **anti-tunnel guard**, not a
  physics tick rate — it only fires when the frame time exceeds 100ms.
- Sub-step write sites at `0x54D8EA`, `0x54D96E` (vehicle) and `0x5474D7` (ped) are additional
  inner clamp layers independent from the outer one — they subdivide long steps.
- FPS-dependent bugs come from **five non-scaled paths** (ragdoll impulse, nitro force, swim
  accumulator, some cutscene deltas, gear-shift timer) — each requires an individual fix,
  not a global `ms_fTimeStep` patch.
- FPS-unlock mods typically keep the physics at 30fps while rendering faster — this fixes the
  frame-rate-dependent physics paths at the cost of decoupling simulation from rendering.

**Previous:** [C52.1 — CTimer globals →](01-timer-and-timestep.md)
**Continue:** [C52.3 — FPS unlock and limit-breaking guide →](03-fps-unlock-guide.md)
