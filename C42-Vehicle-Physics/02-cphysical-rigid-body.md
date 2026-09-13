# C42.2 — The CPhysical rigid body

The handling data ([C42.1](01-handling-data-to-runtime.md)) is only parameters. The thing that actually
*moves* is `CPhysical` — the rigid-body base class every vehicle, and every dynamic object
([C25](../C25-Object-Physics/C25-Object-Physics.md)), inherits. This page lays out its state and the
Newtonian integrator that turns forces into motion.

## What a rigid body stores

`CPhysical` extends `CEntity` (which gives it a position and matrix) and adds the dynamics — **`0x138`
bytes** in total (`gta-reversed`'s `VALIDATE_SIZE(CPhysical, 0x138)`; 🟡 here, the size is corroborated but
not tiled cold this pass). Its rigid-body state:

| Field | Meaning |
|---|---|
| `m_vecMoveSpeed` | linear velocity (units/frame) |
| `m_vecTurnSpeed` | angular velocity (the body's spin) |
| `m_fMass` | mass |
| `m_fTurnMass` | turn mass — the moment of inertia (resistance to spin) |
| `m_fAirResistance` | drag coefficient |
| `m_fElasticity` | bounce on collision |
| `m_fBuoyancyConstant` | float/sink in water |
| `m_vecCentreOfMass` | where forces pivot about |

Two velocities and two "masses" is the signature of a **6-degree-of-freedom rigid body**: `m_vecMoveSpeed`
with `m_fMass` handles translation, `m_vecTurnSpeed` with `m_fTurnMass` handles rotation. Everything a car
does — accelerating, spinning out, rolling — is these four quantities being integrated.

## The integrator: ApplyForce / ApplyTurnForce

Forces enter through three functions, all resolved in `.text` (`derive_physics.py`:
`cphysical_functions_in_text`):

| Function | VA | Role |
|---|---|---|
| `ApplyForce` | `0x542B50` | apply a force at a point → changes both `m_vecMoveSpeed` and `m_vecTurnSpeed` |
| `ApplyTurnForce` | `0x542A50` | apply a force that produces pure rotation about the centre of mass |
| `ApplyFrictionMoveForce` | `0x5430A0` | apply a friction impulse to linear velocity |

`ApplyForce(force, offset)` is the general case: a force applied at an offset from the centre of mass both
pushes the body (linear) and twists it (angular), split according to where it hits. That single function is
why a car struck at the corner both slides and spins, while one struck dead-centre only slides — the
geometry of `offset` decides the split.

## The impulse formula — where turn mass earns its keep

The reason `m_fTurnMass` exists is visible in `CPhysical`'s mass-based impulse response, the factor it scales
a contact impulse by:

```
response = 1 / ( |cross(pos, dir)|² / turnMass  +  1 / mass )
```

Read physically: `1/mass` is how much a force moves the body's translation; `|cross(pos, dir)|² / turnMass`
is how much it rotates the body, weighted by how far off-centre (`pos`) and off-axis (`dir`) the hit is. A
hit through the centre of mass has `cross(pos, dir) = 0`, so the rotation term vanishes and the response is
just `1/mass` — pure translation. A hit at the extremity has a large lever arm, so the turn-mass term
dominates. This is the standard rigid-body point-impulse equation, and finding it verbatim in `CPhysical`
confirms San Andreas runs a real (if arcade-tuned) rigid-body simulation, not a scripted motion model. It is
also why `m_fMassRecpr` is cached in the handling data ([C42.1](01-handling-data-to-runtime.md)): `1/mass` is
evaluated on every impulse.

## One base, two clients

`CPhysical` is shared. A **vehicle** fills its `m_fMass`/`m_fTurnMass`/`m_vecCentreOfMass` from its
`tHandlingData` ([C42.1](01-handling-data-to-runtime.md)); a **dynamic object** fills the same fields from
`object.dat` ([C25](../C25-Object-Physics/C25-Object-Physics.md)'s 24-field record, whose first columns are
mass, turn mass, air resistance, elasticity, centre of mass — the exact `CPhysical` fields). So the crate and
the car are the *same integrator* with different parameter sources. That shared base is why
[C25](../C25-Object-Physics/C25-Object-Physics.md)'s "physics" chapter and this one describe two halves of
one system: C25 is where an object's `CPhysical` parameters come from, C42 is where a vehicle's do, and
`CPhysical` is the engine both feed.

## Open items

- ⏳ The exact field **offsets** within `CPhysical` (this pass proved the field *set* and the functions, not
  each offset) — recoverable by disassembling `ApplyForce` for the `m_vecMoveSpeed`/`m_fMass` accesses.
- ⏳ The integration step (Euler? symplectic?) and the fixed timestep the sim runs at.
- ⏳ `ApplyFrictionMoveForce`'s coupling to [C24](../C24-Surfaces/C24-Surfaces.md) surface friction.

## Key takeaways

- `CPhysical` (**`0x138`**) is the rigid-body base: linear velocity + mass and angular velocity + turn mass —
  a full 6-DOF body, inherited by every vehicle and dynamic object.
- Forces enter via `ApplyForce`/`ApplyTurnForce`/`ApplyFrictionMoveForce`; a force at an offset splits into
  translation and rotation, which is why off-centre hits spin the body.
- The impulse formula `1/(|cross(pos,dir)|²/turnMass + 1/mass)` is the textbook rigid-body point-impulse
  equation — proof the sim is real physics — and it is shared with
  [C25](../C25-Object-Physics/C25-Object-Physics.md)'s objects.

**Continue:** [C42.3 — The vehicle step →](03-the-vehicle-step.md)
