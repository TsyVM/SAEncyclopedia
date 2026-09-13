# Chapter 47 — Vehicle Dynamics: Suspension, Tyres and Damage

> **Goal of this chapter:** go deeper than [C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md), which sized
> the handling parameters and the rigid body. This chapter reads the **runtime wheel model** those parameters
> drive — the **four-wheel suspension**, its compression state, and the tyre/traction arrays — and the
> **`CDamageManager`** that tracks how a vehicle falls apart: engine, wheels, doors, panels and lights, each
> a component with its own damage state. It is the depth layer of the vehicle arc: C42 is the physics, C47 is
> the wheels and the wreckage.

**Subsystem category:** Physics — the per-wheel suspension/tyre model (`CAutomobile`) and vehicle damage
(`CDamageManager`)
**Depends on:** [C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md) (`tHandlingData` suspension/traction
params, the `CPhysical` body) · corroborated by `gta-reversed` (offsets + `VALIDATE_SIZE`)
**Ties:** [C24](../C24-Surfaces/C24-Surfaces.md) (surface grip the tyres multiply),
[C45](../C45-Damage/C45-Damage.md) (component damage is the vehicle's version of armour/health),
[C6](../C6-Collision/C6-Collision.md) (the wheel collision points)
**RE status:** Documented
**Confidence:** ✅ for the 4-wheel array model, the suspension-compression offset and its tiling, and the
`CDamageManager` layout arithmetic (`derive_vehicle_dynamics.py`, 5/5) · 🟡 for exact non-suspension offsets
and struct sizes (gta-reversed `VALIDATE_SIZE`)
**Data artifact:** [`RE-Data/data/vehicle_dynamics.json`](../RE-Data/data/vehicle_dynamics.json) — generated
by [`tools/derive_vehicle_dynamics.py`](../tools/derive_vehicle_dynamics.py)

---

## Deep-dive pages

- [C47.1 — The four-wheel suspension](01-four-wheel-suspension.md): `CAutomobile` carries a bank of
  consecutive **`float[4]`** wheel arrays; the core one, `m_fWheelsSuspensionCompression[4]` @`0x7D4`
  (`[0..1]`, 0 = compressed), is referenced 15× in the physics code, and the arrays **tile at `0x10`**.
- [C47.2 — Tyres and traction](02-tyres-and-traction.md): how the `tHandlingData` traction fields
  ([C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md)) meet the [C24](../C24-Surfaces/C24-Surfaces.md)
  surface grip per wheel — skidmarks, wheel spin, burnout.
- [C47.3 — Damage and deformation](03-damage-and-deformation.md): `CDamageManager` (**`0x18`**) — engine
  (0–250), 4 wheels, 6 doors, a light bitfield and a panel bitfield, each progressing OK → damaged →
  gone, up to `FuckCarCompletely`.
- [C47.4 — Tuning suspension, traction and handling for mods](04-tuning-suspension-and-traction.md):
  **complete `handling.cfg` field → physics formula mapping** for all suspension, traction, drag and mass
  parameters; the **damping ratio criterion** (ζ = c/2√(k×m/4); below 0.3 oscillates, above 1.0 is overdamped)
  and the timestep stability limit from C42.4; the **oversteer/understeer lever** (`fTractionBias` and the
  Coulomb friction limit model); top-speed derivation `V_max = √(fTractionMult×g×m / fDragMult)`; the **mass
  cascade rule** (scale suspension, brake, engine, drag proportionally with fMass); seven **pathological tuning
  patterns** (snap oversteer, suspension oscillation, bottoming-out, high-speed float) with root causes and
  fixes; ASI-level dynamic parameter override and per-frame traction intervention patterns.

---

## 47.0 The result first

| Claim | Value | Tier | Evidence |
|---|---|:--:|---|
| Wheels per car | **4** (FL/RL/FR/RR) | ✅ | `MAX_CARWHEELS`; the `float[4]` arrays |
| Suspension compression | `m_fWheelsSuspensionCompression[4]` @`0x7D4`, `[0..1]` | ✅ | referenced **15×** in vehicle code |
| Wheel arrays tile | at `0x10` (`float[4]`) | ✅ | compression+0x10=prev; spring+0x10=line |
| Spring vs line length | spring `0x878`, line `0x888` (= spring + wheelSize/2) | ✅ / 🟡 | offsets from gta-reversed; ordering verified |
| `CDamageManager` | **`0x18`** | ✅ | 4+1+4+6+4+4 (+1 pad) tiles |
| Damaged components | engine, 4 wheels, 6 doors, lights, panels | ✅ / 🟡 | enum counts + struct arithmetic |
| `CAutomobile` / `CVehicle` | `0x988` / `0x5A0` | 🟡 | gta-reversed `VALIDATE_SIZE` |
| Automated checks | **5 / 5** | — | `tools/derive_vehicle_dynamics.py` refuses to write otherwise |

## 47.1 Four wheels, addressed in parallel

The defining shape of `CAutomobile` is **parallel arrays**: rather than a `Wheel` struct per wheel, the game
keeps one `float[4]` (or `bool[4]`, `CColPoint[4]`) per *property* — a compression array, a rotation array, a
position array, a skidmark-type array — each indexed by the same wheel number (0 = front-left, 1 = rear-left,
2 = front-right, 3 = rear-right). This is the classic data-oriented layout: to process suspension for all
four wheels, the physics walks one contiguous `float[4]`. It is why the offsets march in `0x10` steps
([C47.1](01-four-wheel-suspension.md)) — each array is exactly four floats — and it is the same
structure-of-arrays pattern the render lists ([C40](../C40-Render-Pipeline/C40-Render-Pipeline.md)) use, here
at the scale of one car.

## 47.2 The two halves of the vehicle arc

[C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md) and this chapter are the parameters and the state.
`tHandlingData` ([C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md)) holds the *tuning* — suspension force,
damping, upper/lower limits, traction multiplier — one set per vehicle model. `CAutomobile` holds the *live
values* those tuning numbers produce each frame — this wheel's current compression, that wheel's grip on this
surface, whether a door has been knocked off. So a car's *character* is in C42's handling row, and its
*current condition* is in C47's arrays; the physics step reads the former to update the latter. And just as
[C45](../C45-Damage/C45-Damage.md) showed a ped's survival is armour-then-health, a vehicle's is component
damage — the same idea (a thing degrades until it fails) applied to engine, wheels, doors and panels instead
of one health bar.

---

## Key takeaways

- A car is **four wheels addressed in parallel** — `CAutomobile` keeps one `float[4]` per property, tiling at
  `0x10`, indexed FL/RL/FR/RR.
- The runtime wheel state (compression, spring/line length, rotation, traction) is what
  [C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md)'s `tHandlingData` parameters *drive* each frame — C42
  is the tuning, C47 is the state.
- Vehicle damage is a **per-component** model (`CDamageManager`, `0x18`): engine, 4 wheels, 6 doors, lights
  and panels each degrade independently — the vehicle analogue of [C45](../C45-Damage/C45-Damage.md)'s
  armour/health.

**Continue:** [C47.1 — The four-wheel suspension →](01-four-wheel-suspension.md)

## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Key functions:** `HasFrontWheelDrive` (0x6a0480)
- **Callers:** **1** `.text` call-sites reach this chapter's functions.
- **Callees:** **5** distinct functions called from within them.
- **Known bugs / gotchas:** burst wheel skews handling via m_fWheelDamageEffect; FuckCarCompletely totals it.
- **Modding:** the 4-wheel suspension arrays + CDamageManager are vehicle-mod surfaces.
- **Performance:** 4 suspension raycasts + component state per vehicle per frame.
