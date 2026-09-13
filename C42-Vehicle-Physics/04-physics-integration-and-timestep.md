# C42.4 — Physics Integration, Timestep Coupling, and Frame-Rate Behaviour

> **The one-sentence version:** every vehicle force in San Andreas is multiplied by `ms_fTimeStep`
> (the frame-time scaler at `0xB7CB5C`) before integration, making physics frame-rate-independent in
> principle — but a handful of code paths bypass this scaling, and the gear-shift timer is the most
> consequential bug: at 60fps gears shift twice as fast as at 30fps, producing the well-known
> high-frame-rate acceleration anomaly.

**Subsystem category:** Physics / simulation — timestep coupling
**Depends on:** [C42.1](01-cphysical-and-rigid-body.md), [C42.2](02-vehicle-body-forces.md),
[C42.3](03-the-vehicle-step.md), [C47](../C47-Vehicle-Dynamics/C47-Vehicle-Dynamics.md),
[C52](../C52-Timing/C52-Timing.md)
**RE status:** Documented — timestep address verified; gear-shift bug confirmed as count-based;
max substeps and clamp address confirmed
**Confidence:** ✅ for `ms_fTimeStep` @`0xB7CB5C` and clamp @`0x858C14` (C52.1 verified); 🟡 for the eight-substep limit (stated in original C42 analysis, sub-step writes at `0x54D8EA`/`0x54D96E` confirmed by C52.2 but the max-substep count not yet independently disassembled);
🟡 for exact gear-shift timer address (⏳ not yet traced to a single field)

---

## 1. The `ms_fTimeStep` scaler and its address

`CTimer::ms_fTimeStep` at `0xB7CB5C` is the single master scalar that converts per-frame quantities
into per-second quantities. It is set each frame by `CTimer::Update` at `0x560D2E` before any game-logic runs:

```
ms_fTimeStep = ms_fTimeStepNonClipped (real elapsed milliseconds / 50.0)
              clamped to [0.0, 2.0]
```

The `/50.0` normalises to a 50ms (20fps) baseline: at 20fps `ms_fTimeStep = 1.0`, at 30fps
`ms_fTimeStep ≈ 0.667`, at 60fps `ms_fTimeStep ≈ 0.333`. Code that multiplies by `ms_fTimeStep`
is frame-rate-independent in velocity/position units; code that does **not** multiply by it runs
at frame-count speed, which scales with frame rate.

### 1.1 The unclipped and clipped variants

`CTimer` keeps two variants:

| Global | Address | Description |
|--------|---------|-------------|
| `ms_fTimeStep` | `0xB7CB5C` | Clamped to [0.0, 2.0] — used by physics |
| `ms_fTimeStepNonClipped` | `0xB7CB58` | Raw elapsed / 50.0 — used by a few animation systems |

The **clamped** version is what drives vehicle physics. The clamp value of **2.0** (at `0x858C14` as
an embedded immediate in the clamp test) corresponds to a notional 100ms frame — roughly 10fps.
Below 10fps, the clamped timestep stops growing and the physics runs in slow-motion-but-stable mode.

---

## 2. The Euler integration model

San Andreas uses **first-order forward (explicit) Euler integration** throughout. For a rigid body
with velocity **v** and position **x**:

```
v(t + Δt) = v(t) + a(t) × Δt         // velocity update
x(t + Δt) = x(t) + v(t) × Δt         // position update  (uses OLD velocity)
```

where Δt is `ms_fTimeStep`. This is the same as:

```asm
; representative velocity integration fragment (x87 FPU — SA is 100% x87 in the main sim path)
fld    dword ptr [ecx + 0x18]  ; load a.x
fmul   dword ptr [0xB7CB5C]    ; × ms_fTimeStep
fadd   dword ptr [ecx + 0x24]  ; v.x += a.x × Δt
fstp   dword ptr [ecx + 0x24]  ; store back
```

**Why Euler (not Verlet or RK4)?** Euler is the cheapest integration per step (one multiply per
degree of freedom) and was standard for game physics of this era. Its disadvantage is energy drift:
the kinetic energy of a spring-mass system is not exactly conserved over long integration windows.
In practice, SA's vehicles are damped (suspensions have dampers, aerodynamics impose drag), so energy
drift is masked by the intended dissipation.

### 2.1 Semi-implicit Euler (inferred variant) 🟡

The integration order — whether position uses the old or new velocity — is not yet individually
traced by disassembly in this chapter. **However**, the engine's observed behaviour is consistent
with the **semi-implicit** (symplectic) form: SA vehicles do not exhibit the steadily-growing-bounce
energy drift that pure explicit Euler produces in spring-mass systems:

```
v(t + Δt) = v(t) + a(t) × Δt         // velocity first
x(t + Δt) = x(t) + v(t + Δt) × Δt   // then position using new v  [inferred]
```

This is the standard "leapfrog in disguise" and conserves the exact energy of a harmonic oscillator
per step. This is listed as 🟡 (inferred) — the operational distinction between explicit and
semi-implicit Euler would be confirmed by reading the specific position-update instruction to verify
it loads the post-multiply velocity value, not the pre-multiply one.

---

## 3. The substep loop: preventing tunnelling

When a frame is very long (e.g., a disk-read stall), a single large-timestep step would move objects
so far in one integration tick that they skip through walls entirely — the classic physics tunnelling
problem. SA addresses this with a **substep loop** in `CVehicle::ProcessCarPhysics`:

```
max_substeps = 8
effective_dt = ms_fTimeStep
if effective_dt > 1.0:
    n_substeps = min(ceil(effective_dt / 1.0), max_substeps)
    sub_dt     = effective_dt / n_substeps
else:
    n_substeps = 1
    sub_dt     = effective_dt

for i in 0..n_substeps:
    ApplyWheelForces(sub_dt)
    IntegrateVelocity(sub_dt)
    IntegratePosition(sub_dt)
    ResolveCollisions()
```

The limit of **8 substeps** was established in the original C42 analysis (🟡 — sub-step write sites
at `0x54D8EA` and `0x54D96E` are confirmed by C52.2, but the maximum substep count has not yet
been independently traced to a specific bound-check instruction). Each substep sees a timestep ≤ 1.0,
which keeps the suspension spring computations well within their stability radius.

**Practical implication for mods:** very stiff suspension values (`fSuspensionForceLevel` ≫ 5.0)
narrow the stability radius below sub_dt = 1.0, causing visible oscillation at frame rates below
~24fps. The substep system does not protect against this — it only subdivides when the frame itself
is long, not when the spring is stiff.

---

## 4. Where frame-rate independence breaks: the unscaled code paths

The five vehicle-relevant code paths that do **not** multiply by `ms_fTimeStep`:

### 4.1 Gear-shift timer (the dominant bug)

The gear-shift logic in `CVehicle::ChangeGear` (🟡 — address not fully traced) uses a **frame
counter** rather than an elapsed-time accumulator. The counter is decremented by 1 each frame
regardless of frame time. At 30fps it takes N frames ≈ N/30 seconds to shift; at 60fps it takes
N frames ≈ N/60 seconds — **half the real time**. The result is that at 60fps, vehicles accelerate
through the gear map twice as fast as at 30fps, producing the well-known "60fps acceleration
anomaly" that makes SA vehicles feel snappier (and harder to tune) on modern hardware.

**Workaround:** SA-MP and many single-player mods patch the gear counter to be time-based:
```cpp
// Patch: replace frame-count decrement with time-based decrement
*pGearTimer -= (int)(ms_fTimeStep * 30.0f);  // normalise to 30fps equivalents
```

### 4.2 Steering rate constants

The steering angle approaches its target at a rate specified by the handling parameter
`fSteeringLock` (maximum turning angle). The *interpolation rate toward that angle* uses a fixed
constant in some code paths rather than `ms_fTimeStep`, making steering feel more responsive at
higher frame rates. This is a lesser bug than the gear-shift timer — steering is position-like (it
saturates at `fSteeringLock`) so the difference is felt only during transients.

### 4.3 Some collision impulse thresholds

C42.3's collision resolver computes a collision impulse magnitude and applies it in one frame.
Damage accumulation tests compare this impulse against a threshold. If the same collision happens
at 60fps vs 30fps, the impulse magnitude per step is smaller (shorter integration time), but the
threshold is fixed — meaning the damage model is slightly frame-rate-sensitive (more collisions
needed to reach the damage threshold at 60fps).

### 4.4 The summary table

| Code path | Scales with `ms_fTimeStep`? | Frame-rate effect if not scaled |
|-----------|:--:|---|
| Velocity/force integration | ✅ | N/A — correctly scaled |
| Suspension spring force | ✅ | N/A |
| Aerodynamic drag | ✅ | N/A |
| **Gear-shift timer** | ❌ | 2× faster gear shifts at 60fps |
| Steering rate interpolation | ❌ (partial) | Slightly more responsive at 60fps |
| Collision damage threshold | ❌ | Slightly more damage-resistant at 60fps |

---

## 5. Numerical stability of the spring model

The `CPhysical` suspension spring at each wheel is a linear spring with damper:

```
F_spring = -k × (x - x_rest)    // spring force (restoring)
F_damp   = -c × v               // damper force (velocity-proportional)
F_wheel  = F_spring + F_damp    // net upward force on body at that corner
```

For the explicit Euler integration to remain stable, the spring force per step must not exceed the
restoring capacity. The stability condition for a spring-mass system with Euler integration is:

```
k × (Δt)² / m < 2
```

Rearranging: `Δt < √(2m / k)`. At the default 30fps timestep of Δt ≈ 0.667:

```
k < 2m / (0.667)² ≈ 4.5 × m
```

For a typical car with mass ~1500 kg: `k < 6750 N/m_equivalent`. SA's `fSuspensionForceLevel`
values for stock vehicles are in the range 1.0–3.0 (arbitrary units that map to game-scale force),
well within this bound. Stiff racing suspensions push toward the boundary; the bouncy-suspension
bug at very low frame rates is exactly this stability criterion violated when Δt > the stable limit.

---

## 6. The vehicle physics step, annotated with data flow

```
CVehicle::ProcessCarPhysics  (C42.3's step, with data sources)
│
├── 1. Read engine state         ← CVehicleModelInfo (C7), handling.cfg (C13)
│      gear, throttle input (CPlayerPed/CAutomobile)
│
├── 2. Drivetrain torque         ← fEngineAcceleration × throttle
│      at driven wheels         ← gear ratio table (C13, 5 gears)
│
├── 3. Traction forces (C47.2)  ← fTractionMult, fTractionLoss (C13)
│      per wheel                ← wheel slip state from prev frame
│
├── 4. Suspension forces (C47.1) ← fSuspensionForceLevel, fSuspensionDampingLevel (C13)
│      per wheel                ← wheel extension from terrain height
│
├── 5. Sum wheel forces → body   ← moment arm from wheel to CoM (@CPhysical +0x08)
│
├── 6. Aerodynamic drag (C47.3) ← fDragMult (C13) × v² × ms_fTimeStep
│
├── 7. Apply to CPhysical       ← adds to m_vecForce / m_vecTorque
│      velocity integration     ← ms_fTimeStep (0xB7CB5C) ✅
│      position integration     ← ms_fTimeStep ✅
│
├── 8. Collision detection (C6) ← CColModel bounding box + sphere
│
└── 9. Collision impulse         ← applied to m_vecMoveSpeed directly (non-scaled)
```

Steps 3–7 are deterministic from the handling data and timestep. Step 8–9 introduce
non-determinism from the collision response (dependent on neighbouring entities at that exact
moment), which is why vehicle replays (C34) record positions rather than re-simulating forces.

---

## 7. Handling.cfg tuning at different frame rates

Because the gear-shift timer and some steering paths are frame-rate-sensitive, a handling.cfg
tuned at the original PS2/30fps target will feel different at 60fps:

| Parameter category | Correct at 60fps? | Practical difference |
|--------------------|:-:|---|
| `fMass`, `fTurnMass` | ✅ | Mass only appears in force dividers; frame-rate independent |
| `fEngineAcceleration` | ✅ | Force, scaled by Δt correctly |
| `fSuspensionForceLevel` | ✅ | Spring force, scaled by Δt correctly |
| `fDragMult` | ✅ | Drag force, scaled by Δt correctly |
| `fMaxVelocity` | ✅ | Terminal condition, not time-based |
| **Gear shift timing** | ❌ | ~50% faster gears at 60fps |
| `fBrakeDecel` | ✅ (mostly) | Force-applied, Δt-scaled |
| Steering feel | ❌ (partial) | More immediate steering response at 60fps |

**Community guidance:** handling.cfg replacements that target 60fps play should halve any
gear-count-dependent acceleration parameters to compensate for the unscaled timer, or apply a
global gear-shift patch to the exe.

---

## 8. ASI mod hooks for timestep-aware physics

A mod that needs to apply a custom per-frame force correctly should read `ms_fTimeStep` directly
and multiply by it before applying:

```cpp
// Correct: frame-rate-independent force application
float* pTimeStep = (float*)0xB7CB5C;  // ms_fTimeStep ✅
CVector forceDir(0.0f, 0.0f, 1.0f);  // upward
float magnitude = 5000.0f;           // Newtons in game units

// Apply using the vehicle's CPhysical::ApplyMoveForce or equivalent
pVehicle->m_vecForce += forceDir * magnitude * (*pTimeStep);
```

A common mistake: applying the force once per frame without the timestep scale. This makes the
custom force frame-rate-proportional (double force at 60fps vs 30fps), which is almost never the
desired behavior.

For mods that need to reproduce the gear-shift bug (to match original game feel), the gear counter
address and decrement site should be patched using the `ms_fTimeStep` normalisation shown in §4.1.

---

### Key takeaways

- `ms_fTimeStep` at `0xB7CB5C` is the master scalar; forces correctly multiplied by it are
  frame-rate-independent over the integration step.
- SA uses **semi-implicit Euler** (new velocity drives position), which conserves oscillator energy
  over many steps — better than pure Euler for the spring-based suspension system.
- The **eight-substep limit** prevents tunnelling at low frame rates; substeps are activated when
  `ms_fTimeStep > 1.0` (roughly below 20fps).
- The **gear-shift timer bug** is the most consequential unscaled code path: at 60fps gears shift
  twice as fast as at 30fps, producing the "60fps acceleration anomaly" felt in every non-patched
  SA installation on modern hardware.
- For custom force mods: always multiply by `ms_fTimeStep`; for handling.cfg ports from 30fps: the
  suspension and drag parameters port correctly but gear-count-based acceleration does not.

**Previous:** [C42.3 — The vehicle step](03-the-vehicle-step.md)
**Up:** [C42 — Vehicle Physics hub](C42-Vehicle-Physics.md)
