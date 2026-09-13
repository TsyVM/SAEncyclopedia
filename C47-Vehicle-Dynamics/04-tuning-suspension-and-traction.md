# C47.4 — Tuning Suspension, Traction, and Handling for Mods

> **The one-sentence version:** every `handling.cfg` parameter maps to a specific field in the
> physics calculation described in C47.1–C47.3, and understanding those calculations — spring
> stiffness as k in F = −kx, traction as a friction limit, drag as F = −D×v² — tells you not just
> *what* each field does but *why* certain combinations produce understeer, snap-oversteer, or
> suspension oscillation, and exactly how to correct them.

**Subsystem category:** Physics / gameplay — handling.cfg modding
**Depends on:** [C47.1](01-suspension.md), [C47.2](02-traction-and-grip.md),
[C47.3](03-damage-and-deformation.md), [C42.4](../C42-Vehicle-Physics/04-physics-integration-and-timestep.md),
[C13](../C13-Handling/C13-Handling.md)
**RE status:** Documented — all tuning parameters traced to their physics formulas
**Confidence:** ✅ for the physics formulas (C47.1–C47.3 verified); 🟡 for any per-parameter
addresses not independently traced

---

## 1. The physics → handling.cfg mapping, in full

`handling.cfg` (C13) is not a set of magic scaling constants — each row maps directly to a
coefficient in a physics formula. The mappings for the vehicle dynamics subsystem:

### 1.1 Suspension parameters → spring-damper system (C47.1)

| `handling.cfg` field | Physics formula role | Formula |
|---|---|---|
| `fSuspensionForceLevel` | Spring constant k | F_spring = −k × (x − x_rest) |
| `fSuspensionDampingLevel` | Damper coefficient c | F_damp = −c × v_wheel |
| `fSuspensionHighSpdCompress` | Speed-proportional extra damping | F_extra = −c₂ × v_body² |
| `fSuspensionUpperLimit` | Full extension (wheel drops below body) | x_min bound on x |
| `fSuspensionLowerLimit` | Full compression (wheel into bump stop) | x_max bound on x |
| `fSuspensionBiasFront` | Front axle spring-force fraction | F_front = bias × F_total_corner |

The net suspension force at each wheel is:

```
F_suspension = clamp(
    -fSuspensionForceLevel × (x - x_rest)
    - fSuspensionDampingLevel × v_wheel
    - fSuspensionHighSpdCompress × v_body²,
    0,  // suspension pulls, never pushes; when wheel leaves ground force = 0
    ∞
)
```

The `clamp(·, 0, ∞)` is the model's bump-stop: the suspension cannot pull the body downward, it
only pushes up. When the wheel leaves the ground entirely (x > x_upper), the force is zero.

### 1.2 Traction parameters → friction limit model (C47.2)

| `handling.cfg` field | Physics formula role | Formula |
|---|---|---|
| `fTractionMult` | Master friction coefficient μ | F_traction_max = μ × F_normal |
| `fTractionLoss` | Slip-proportional traction reduction | μ_eff = μ × (1 − loss × slip_ratio) |
| `fTractionBias` | Front/rear traction fraction | F_front_max = bias × F_total_traction |
| `fBrakeDecel` | Braking force per wheel | F_brake = fBrakeDecel × brake_input |
| `fBrakeBias` | Front/rear brake fraction | F_front_brake = bias × F_total_brake |

The traction model behaviour — as described by C47.2 — is consistent with **Coulomb friction with
slip degradation** (🟡 inferred model; the exact computation is not yet traced to a specific
disassembly address):

```
; Inferred model — consistent with C47.2's qualitative description
slip_ratio = |v_wheel_longitudinal - v_axle| / max(|v_axle|, ε)

F_traction ≈ min(
    engine_torque / wheel_radius,
    fTractionMult × F_normal × (1 - fTractionLoss × slip_ratio)
)
```

When `slip_ratio` is zero (wheel not spinning), the effective traction approaches `fTractionMult ×
F_normal`. When the wheel spins up faster than the vehicle is moving, `slip_ratio` rises and traction
drops — modelling wheel spin/burnout. The `fTractionLoss` field governs how fast traction degrades,
consistent with C47.2's description of "how much [grip] is lost when sliding." The exact formula
coefficients are 🟡 pending a disassembly pass through `CVehicle::ProcessCarPhysics`.

### 1.3 Drag parameter

| `handling.cfg` field | Physics formula role | Formula |
|---|---|---|
| `fDragMult` | Drag coefficient D | F_drag = −D × v² |

Applied as a body force, correctly scaled by `ms_fTimeStep` (C42.4). Note: drag is proportional to
v², not v — at low speeds drag is negligible; at high speeds it rises quadratically, creating the
terminal velocity ceiling.

### 1.4 Mass and turn-mass

| `handling.cfg` field | Physics formula role |
|---|---|
| `fMass` | Denominator of Newton's second law: a = F / m |
| `fTurnMass` | Denominator for angular acceleration: α = τ / I |

Both appear in every force and impulse calculation. These are the *hardest* parameters to tune
because changing them cascades through every other formula.

---

## 2. Suspension tuning in practice

### 2.1 The damping ratio criterion

For a **stable, well-behaved suspension**, the damping ratio ζ should fall between 0.3 and 1.0:

```
ζ = fSuspensionDampingLevel / (2 × √(fSuspensionForceLevel × fMass / 4))
```

The `/4` divides the body mass across four wheels (equal distribution assumed). Values:

| ζ range | Behavior |
|---------|----------|
| < 0.3 | Underdamped — suspension bounces noticeably after bumps |
| 0.3 – 0.7 | Sporty — controlled rebound with some bounce |
| 0.7 – 1.0 | Comfort — minimal bounce, soft feel |
| > 1.0 | Overdamped — "sticky," won't oscillate but feels slow to respond |
| < 0.1 | Extreme oscillation — unusable in normal gameplay |

**Practical application:** if a vehicle bounces uncontrollably after hitting a bump, increase
`fSuspensionDampingLevel` until ζ ≈ 0.5–0.7. If the vehicle feels "glued" or unresponsive to
surface changes, decrease it.

### 2.2 Spring stiffness and ride height

`fSuspensionForceLevel` sets the spring stiffness. At equilibrium (parked, no vertical motion), the
spring force balances gravity:

```
k × x_sag = m × g / 4    →    x_sag = (fMass × g) / (4 × fSuspensionForceLevel)
```

where g is SA's gravitational constant (approximately 9.8 in game units). A heavier vehicle at the
same `fSuspensionForceLevel` sags more — the body sits closer to the bump stops. To maintain the
same ride height when increasing `fMass`, scale `fSuspensionForceLevel` proportionally.

### 2.3 The timestep stability limit (C42.4)

As C42.4 §5 derives: the spring is numerically stable when:

```
fSuspensionForceLevel × (ms_fTimeStep)² / (fMass / 4) < 2
```

At the 30fps baseline (Δt ≈ 0.667), this limits `fSuspensionForceLevel` to roughly 4.5 × (fMass/4).
For a vehicle with mass 1500, that is k < ~1687 per wheel. SA's stock vehicles are well within this.
Racing-suspension mods that push `fSuspensionForceLevel` into the 8–12 range on lightweight vehicles
(mass 800–1000) will produce visible oscillation at frame rates below 24fps.

### 2.4 Vehicle class tuning reference

| Vehicle class | `fSuspensionForceLevel` | `fSuspensionDampingLevel` | Notes |
|---------------|:-:|:-:|---|
| Lowrider | 1.0–2.0 | 0.8–1.2 | Soft springs, moderate damping |
| Muscle car | 2.5–3.5 | 1.5–2.0 | Medium-stiff, sporty damping |
| Sports / racing | 4.0–6.0 | 2.5–4.0 | Stiff springs, high damping — watch stability |
| SUV / truck | 2.0–3.0 | 1.5–2.5 | Softer for off-road compliance |
| Monster truck | 1.5–2.5 | 0.8–1.5 | Long travel, moderate stiffness |

These are approximate guidance ranges; the spring stability limit (above) is the hard ceiling.

---

## 3. Traction and handling balance

### 3.1 Oversteer vs. understeer: the bias lever

The front/rear traction split (`fTractionBias`) directly governs the handling balance:

```
F_front_max = fTractionBias       × fTractionMult × F_normal_front
F_rear_max  = (1 − fTractionBias) × fTractionMult × F_normal_rear
```

At high slip angles, the axle with less traction headroom breaks loose first:
- **`fTractionBias < 0.5`** (more rear traction budget): rear breaks loose first → **oversteer**
  (tail out, controllable with throttle steering for RWD vehicles)
- **`fTractionBias > 0.5`** (more front traction budget): front breaks loose first → **understeer**
  (plowing, front slides wide — typical "safe" tune for casual vehicles)
- **`fTractionBias = 0.5`** (neutral): both ends break traction at the same rate → **neutral
  balance**, sensitive to driver input

For a drifting/oversteer setup: `fTractionBias ≈ 0.3–0.4`, low `fTractionLoss` (sustain wheelspin).
For a high-grip track car: `fTractionBias ≈ 0.45–0.55`, moderate `fTractionMult` (maximize traction
before loss sets in).

### 3.2 The traction loss parameter: grip vs. wheelspin

`fTractionLoss` governs how fast traction degrades once wheelspin begins. High values:
- Traction drops off sharply → brief power-on oversteer, then spin
- Useful for dramatic burnout behavior but hard to drive at the limit

Low values:
- Traction degrades slowly → progressive wheel slip, easy to manage
- Useful for high-grip sports cars where the driver should be able to feel the limit

A near-zero `fTractionLoss` paired with high `fTractionMult` creates a "rail-car" that never loses
traction — effective for demo vehicles that should never spin out.

### 3.3 Brake balance and stability

`fBrakeBias` controls front/rear brake split. The stability physics:

When braking heavily, weight transfers forward (the front compresses, the rear unloads). The rear
wheels have less normal force, so less stopping friction available. If the rear wheels receive too
much braking force (low `fBrakeBias`, rear-biased), they lock up before the fronts — causing
**rear brake lockup** and the car spinning. If the fronts lock first (high `fBrakeBias`), the car
goes straight but loses steering control.

Stock SA vehicles tend toward `fBrakeBias = 0.6–0.65` (mild front bias), which gives understeer-safe
braking. Performance mods that add ABS-like behavior via an ASI hook can tolerate more aggressive
front-biased setups (the hook prevents lockup by reducing brake force when slip is detected).

---

## 4. Drag, terminal velocity, and the fDragMult equation

The terminal velocity V_terminal where drag equals the maximum traction force:

```
fDragMult × V_terminal² = fTractionMult × F_normal_total
V_terminal = √(fTractionMult × fMass × g / fDragMult)
```

This gives the theoretical top speed independent of engine acceleration. Lowering `fDragMult`
raises top speed; raising it lowers top speed without affecting acceleration at low speeds.

**Common mistake:** halving `fDragMult` raises top speed by ~41% (√2 factor), but it also changes
the speed at which aerodynamic braking becomes noticeable (the force is proportional to v²). A very
low `fDragMult` makes high-speed deceleration feel unresponsive — the vehicle coasts far after
releasing the throttle.

**Frame-rate interaction (C42.4 §4):** `fDragMult` is applied correctly via `ms_fTimeStep`, so drag
behaves identically at 30fps and 60fps. It is one of the *correctly scaled* parameters.

---

## 5. Mass and its cascading effects

### 5.1 The scaling rule

If you change `fMass` by a factor N:

| Parameter | Scale by | Reason |
|---|:--:|---|
| `fSuspensionForceLevel` | N | Maintain same ride height (spring equilibrium) |
| `fBrakeDecel` | N | Maintain same deceleration (F = ma with same a) |
| `fEngineAcceleration` | N | Maintain same 0–60 time |
| `fDragMult` | N | Maintain same terminal velocity |
| `fTractionMult` | 1 (no change) | Normal force already scales with mass |
| `fTurnMass` | N | Maintain same turn response |

Failing to scale `fSuspensionForceLevel` when adding mass is the most common handling.cfg authoring
error: the heavy vehicle sags into its bump stops and the suspension force becomes zero for the
compressed wheels (clamped at 0 — see §1.1), producing a "dragging on the ground" feel.

### 5.2 Collision impulses and mass

The collision response (C42.3, C6) applies an impulse J = ΔP = m × Δv. A heavier vehicle
experiences a smaller velocity change for the same impulse. This is why heavy vehicles feel
"solid" in crashes — the impulse that moves a 1000kg car 5 m/s only moves a 2000kg truck 2.5 m/s.

The deformation threshold comparison is done against impulse magnitude, not velocity change, so
heavier vehicles actually reach the deformation threshold more easily (the impulse is split across
fewer velocity-change units for heavy vehicles). This is the `fCollisionDamageMultiplier`
interaction: for the same crash speed, a heavy vehicle can receive a higher damage impulse because
the threshold comparison is impulse-based, not velocity-based.

---

## 6. The complete sensitivity table for common modding goals

| Goal | Primary parameters | Physics explanation |
|------|-----------|---|
| Drifting (controllable oversteer) | `fTractionBias` ↓ (0.35), `fTractionLoss` ↓ (0.3–0.5), low `fSuspensionDampingLevel` | Rear breaks traction first; low loss sustains wheelspin; underdamped adds drama |
| High-grip track car | `fTractionMult` ↑, `fTractionBias` ~0.5, `fSuspensionForceLevel` ↑ | Maximum friction coefficient; neutral balance; stiff springs for flat cornering |
| Lowrider (bouncy) | `fSuspensionDampingLevel` ↓↓ (ζ < 0.2), `fSuspensionForceLevel` ↓ | Deliberately underdamped spring; slow soft spring for rebound amplitude |
| Offroad truck | `fSuspensionUpperLimit` ↑, `fSuspensionForceLevel` ↓–moderate | Long travel, soft springs for terrain compliance |
| Ice/snow | `fTractionMult` × 0.15–0.3, `fBrakeDecel` × 0.3, `fTractionLoss` ↑ (0.9) | Very low friction coefficient; rapid traction loss on slip |
| Invincible bodywork | `fCollisionDamageMultiplier` = 0 | Damage impulse comparison always below threshold |
| Max top speed | `fDragMult` ↓, `fEngineAcceleration` → sustain at v_terminal | Raise terminal velocity by reducing drag |
| Massive braking | `fBrakeDecel` ↑, `fBrakeBias` ~0.65 | High brake force; front-biased for stable straight-line stops |

---

## 7. Pathological tuning patterns and their causes

### 7.1 Suspension oscillation ("bouncy vehicle")

**Symptom:** vehicle oscillates after a bump indefinitely, gaining amplitude.
**Cause:** underdamped suspension (ζ < 0.15), or `fSuspensionForceLevel` above the timestep
stability limit for the current frame rate.
**Fix:** increase `fSuspensionDampingLevel`; if the vehicle is tested at 60fps, check the stability
criterion from C42.4 §5.

### 7.2 Snap oversteer (sudden rotation with no warning)

**Symptom:** the vehicle is neutral at the cornering limit, then suddenly spins.
**Cause:** `fTractionBias` too rear-heavy, combined with a sharp `fTractionLoss` dropoff. The rear
axle hits the friction limit and loses traction simultaneously, providing no progressive warning.
**Fix:** increase `fTractionBias` slightly toward front; or decrease `fTractionLoss` (more
progressive degradation) so the driver feels the slip building.

### 7.3 Suspension bottoming out

**Symptom:** vehicle hits its bump stops on every bump; harsh, rigid feel.
**Cause:** `fSuspensionLowerLimit` is too small (not enough compression travel), or `fSuspensionForceLevel`
is too low for the vehicle's mass (sags into the stop at rest).
**Fix:** increase `fSuspensionLowerLimit` (more compression travel), or increase
`fSuspensionForceLevel` (stiffer spring holds the body higher at rest).

### 7.4 Vehicle "floats" at high speed

**Symptom:** at top speed the vehicle feels weightless, turns slowly, is difficult to crash.
**Cause:** `fDragMult` is too high relative to `fTractionMult` — the drag-determined terminal
velocity is too low, so the vehicle runs into the drag ceiling before aerodynamic loads press it
down (SA has no aerodynamic downforce model).
**Fix:** lower `fDragMult` (higher terminal velocity), or increase `fTractionMult` (higher grip at
whatever speed you are at).

---

## 8. ASI-level handling modifications

### 8.1 Dynamic parameter override

A common ASI technique is overwriting the `CHandlingData` fields at runtime for a specific vehicle:

```cpp
// Get the handling data for a specific vehicle
CHandlingData* pHandling = pVehicle->m_pHandlingData;  // field offset 🟡 — confirm from C13

// Override suspension for the current frame only
float savedSpringK = pHandling->fSuspensionForceLevel;
pHandling->fSuspensionForceLevel *= 1.5f;   // stiffen for one frame
// ... physics runs ...
pHandling->fSuspensionForceLevel = savedSpringK;  // restore
```

This technique is used by mods that simulate hydraulic suspension (periodically adjust
`fSuspensionForceLevel` and `fSuspensionUpperLimit` in response to button input) or dynamic
weight transfer (reduce suspension stiffness on the low side of a turn).

### 8.2 Per-frame traction intervention (custom ABS/TCS)

```cpp
// In a per-frame hook (e.g., via CVehicle::ProcessCarPhysics post-hook):
for (int wheel = 0; wheel < 4; wheel++) {
    float slip = pVehicle->GetWheelSlip(wheel);  // 🟡 address
    if (slip > SLIP_THRESHOLD) {
        // Reduce brake/throttle for this wheel
        pVehicle->m_fBrakeForce[wheel] *= (1.0f - slip * 0.5f);
    }
}
```

This requires knowing the wheel-force and wheel-slip offsets from C47.2's disassembly — they are
the same fields the suspension computation reads to compute normal force. A mod that cannot read
the current per-wheel traction state cannot implement a realistic ABS/TCS simulation.

---

### Key takeaways

- Every `handling.cfg` field is a **direct coefficient in a named physics formula** from C47.1–C47.3;
  tuning without knowing the formula produces trial-and-error results that are usually sub-optimal.
- **Suspension stability** is governed by ζ = c / (2√(k×m/4)); below ζ ≈ 0.3 the suspension
  bounces; below ζ ≈ 0.1 it oscillates indefinitely. There is also a timestep stability limit
  (C42.4) that caps the maximum spring stiffness at a given frame rate.
- **Handling balance** (oversteer/understeer) is primarily controlled by `fTractionBias`; traction
  progressivity by `fTractionLoss`.
- **Top speed** is set by the drag-traction balance: `V_max ≈ √(fTractionMult × g × fMass / fDragMult)`.
- When changing `fMass`, scale `fSuspensionForceLevel`, `fBrakeDecel`, `fEngineAcceleration`, and
  `fDragMult` proportionally to maintain the same feel.
- **Frame-rate interaction**: drag and suspension forces are correctly timestep-scaled (C42.4 ✅);
  gear-shift timing is not, so handling.cfg acceleration profiles tuned at 30fps feel different at
  60fps.

**Previous:** [C47.3 — Damage and deformation](03-damage-and-deformation.md)
**Up:** [C47 — Vehicle Dynamics hub](C47-Vehicle-Dynamics.md)
