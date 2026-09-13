# C42.3 — The vehicle step

The parameters ([C42.1](01-handling-data-to-runtime.md)) and the rigid body ([C42.2](02-cphysical-rigid-body.md))
meet once per frame in the vehicle's process step. This page assembles them into the loop that turns
`tHandlingData` numbers into motion, and places that loop relative to collision and rendering.

## The step, force by force

Each frame a vehicle's `ProcessControl` reads its `tHandlingData` and applies a sequence of forces to its
`CPhysical` base. In order of what they model:

1. **Drive force.** The engine torque (from `m_transmissionData` — gears, drive type `'F'`/`'R'`/`'4'`, top
   speed) is applied at the driven wheels. Front/rear/four-wheel drive changes *where* on the body the force
   is applied ([C42.1](01-handling-data-to-runtime.md)'s `+0x88` drive-type byte), which is why FWD and RWD
   cars behave differently under power.
2. **Braking.** `m_fBrakeDeceleration` and `m_fBrakeBias` (front/rear split) apply a decelerating force per
   wheel; ABS (`m_bABS`) modulates it.
3. **Traction.** `m_fTractionMultiplier`, `m_fTractionLoss` and `m_fTractionBias` set how much lateral grip
   each tyre has — and this is where [C24](../C24-Surfaces/C24-Surfaces.md) enters: the surface under each
   wheel supplies a grip factor that multiplies the handling traction, so the same car slides more on sand
   than on tarmac. Grip lost to wheelspin or a slippery surface is what breaks the car into a slide.
4. **Suspension.** The suspension fields (`m_fSuspensionForceLevel`, `DampingLevel`, `UpperLimit`,
   `LowerLimit`, `BiasBetweenFrontAndRear`, `AntiDiveMultiplier`) push each wheel back toward its rest
   position — a spring-damper per wheel that also produces the body pitch/roll under acceleration and
   cornering.
5. **Drag and gravity.** `m_fDragMult` and buoyancy/air resistance from the [C42.2](02-cphysical-rigid-body.md)
   `CPhysical` base finish the frame.

Every one of these is an `ApplyForce`/`ApplyTurnForce` call ([C42.2](02-cphysical-rigid-body.md)) at a wheel
position, so they naturally produce both translation and the pitch/roll/yaw that make the car feel weighted.
The centre of mass (`m_vecCentreOfMass`) is the pivot they all act about — lower it and the car resists roll;
move it forward and it noses under braking.

## Where the step sits in the frame

Vehicle physics is part of the **update** half of the frame, before rendering
([C40](../C40-Render-Pipeline/C40-Render-Pipeline.md)'s stage 1):

1. **Streaming/AI** — the vehicle exists and has a driver ([C41](../C41-Ped-AI-Tasks-Wanted/C41-Ped-AI-Tasks-Wanted.md)
   tasks decide throttle/steer for NPCs).
2. **Collision** — [C6](../C6-Collision/C6-Collision.md) finds contacts; each contact becomes an
   `ApplyForce` impulse through the [C42.2](02-cphysical-rigid-body.md) mass/turn-mass response.
3. **Vehicle `ProcessControl`** — the force sequence above integrates the rigid body.
4. **Camera** — [C39](../C39-Camera/C39-Camera.md)'s `MODE_CAM_ON_A_STRING` follows the now-moved vehicle.
5. **Render** — [C40](../C40-Render-Pipeline/C40-Render-Pipeline.md) draws it at its new transform.

So the physics runs *before* the camera and render read the vehicle's matrix — the same
[C39](../C39-Camera/C39-Camera.md)/[C40](../C40-Render-Pipeline/C40-Render-Pipeline.md) matrix the rendering
chapters consume is the output of this step. The rendering arc and the physics arc meet here: physics writes
the transform, camera and renderer read it.

## What closes the vehicle picture, and what's left

Between [C13](../C13-Vehicle-Data/C13-Vehicle-Data.md) (the data), C42.1 (the runtime array), C42.2 (the
rigid body) and this page (the step), the **vehicle handling pipeline is now end to end**: a `handling.cfg`
row → a `tHandlingData` struct → per-frame forces → a `CPhysical` integration → a moved transform. The gaps
are depth, not coverage:

- ⏳ **The wheel/tyre model** — the exact slip-angle → lateral-force curve (`m_fTractionLoss`/`Bias`) and the
  per-wheel suspension raycast.
- ⏳ **Bikes, planes, boats** — the `tBike/Flying/BoatHandlingData` structs (sized in
  [C42.1](01-handling-data-to-runtime.md)) have their own process steps.
- ⏳ **`CPhysical` field offsets and the integrator's timestep** (from [C42.2](02-cphysical-rigid-body.md)).
- ⏳ **Damage/deformation** — the collision-to-mesh-deform path, a sibling system.

## Key takeaways

- The vehicle step applies a fixed sequence of forces from `tHandlingData` — drive, brakes, traction (×
  [C24](../C24-Surfaces/C24-Surfaces.md) surface grip), suspension, drag — each an `ApplyForce` at a wheel,
  pivoting about the centre of mass.
- It runs in the **update** half of the frame, after [C6](../C6-Collision/C6-Collision.md) collision and
  before the [C39](../C39-Camera/C39-Camera.md)/[C40](../C40-Render-Pipeline/C40-Render-Pipeline.md) camera
  and render read the vehicle's transform — physics writes the matrix the renderer reads.
- With C13 → C42.1 → C42.2 → C42.3 the **vehicle handling pipeline is end to end**; what remains is depth
  (tyre model, suspension raycast, bike/plane/boat, damage).

**Continue:** [back to the C42 hub →](C42-Vehicle-Physics.md) · or [C13 — Vehicle Data](../C13-Vehicle-Data/C13-Vehicle-Data.md) (the handling.cfg rows) · [C25 — Object Physics](../C25-Object-Physics/C25-Object-Physics.md) (the shared CPhysical).
