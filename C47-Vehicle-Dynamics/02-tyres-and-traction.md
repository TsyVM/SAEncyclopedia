# C47.2 — Tyres and traction

A suspended wheel ([C47.1](01-four-wheel-suspension.md)) only matters once it grips. This page follows the
tyre side: how the [C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md) traction parameters and the
[C24](../C24-Surfaces/C24-Surfaces.md) surface under each wheel combine into the grip that drives, brakes and
slides the car — plus the visible by-products (skidmarks, wheelspin).

## Grip is handling × surface, per wheel

Traction in San Andreas is a product of two independently-documented systems meeting at the tyre:

- From **[C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md)** (`tHandlingData`): `m_fTractionMultiplier`
  (base grip), `m_fTractionLoss` (how much is lost when sliding), `m_fTractionBias` (front/rear grip split).
  These are the car's tyres — a sports car has more `TractionMultiplier` than a truck.
- From **[C24](../C24-Surfaces/C24-Surfaces.md)** (`surfinfo.dat`): the surface under the wheel supplies a
  grip factor — tarmac grips, sand and grass do not. Each wheel's [C6](../C6-Collision/C6-Collision.md)
  collision point ([C47.1](01-four-wheel-suspension.md)'s `m_wheelColPoint`) carries the surface id it is
  standing on.

The per-wheel grip is the handling traction *scaled by* the surface factor. This is the cross-subsystem
payoff the two chapters set up separately: the same car slides on sand and grips on road because the identical
`TractionMultiplier` is multiplied by a different [C24](../C24-Surfaces/C24-Surfaces.md) surface value under
each of the four wheels. Only when the springs ([C47.1](01-four-wheel-suspension.md)) have the wheel planted —
non-zero compression — does that grip apply; a wheel in the air (compression near 1, no collision point)
contributes none, which is why a car going over a crest loses drive.

## Losing grip: the slide

`m_fTractionLoss` is what turns a planted tyre into a sliding one. When the force demanded at a tyre (from
steering, braking or engine torque) exceeds what the grip can supply, the tyre breaks loose and the effective
grip drops toward the `TractionLoss` value — the car understeers, oversteers or spins. Because grip is
per-wheel, this happens corner by corner: braking hard shifts load forward (the front springs compress, the
rear relax — [C47.1](01-four-wheel-suspension.md)), so the lightly-loaded rear tyres lose grip first, which
is the physical basis of a handbrake slide. The front/rear `TractionBias` is what the designers tuned to make
some cars tail-happy and others push wide.

## The visible by-products

The tyre state surfaces in `CAutomobile`'s per-wheel arrays ([C47.1](01-four-wheel-suspension.md)):

- **Skidmarks** — `m_wheelSkidmarkType[4]` (`0x810`), `m_wheelSkidmarkMuddy[4]` (`0x824`),
  `m_wheelSkidmarkBloodState[4]` (`0x820`): each wheel independently lays a mark whose *type* depends on the
  [C24](../C24-Surfaces/C24-Surfaces.md) surface (a muddy surface leaves a muddy trail), and blood if the car
  drove through a corpse. Four wheels → up to four parallel skidmark trails.
- **Wheelspin / burnout** — `m_fWheelBurnoutSpeed[4]` (`0x858`): how fast each driven wheel is spinning
  beyond the car's actual speed, which is the smoke-and-noise of a burnout and feeds back as lost grip.
- **Rotation** — `m_wheelRotation[4]` (`0x828`): the visual spin angle, so the wheel model turns at a rate
  matching ground speed (or faster, when spinning).

All of these are per-wheel because grip is per-wheel: the tyre that is sliding is the one that smokes and
marks, and the model keeps four independent entries so a single locked wheel behaves correctly.

## Open items

- ⏳ The exact **slip → force curve** (how demanded force past the grip limit maps to `TractionLoss`) — a
  numeric function not pinned here.
- ⏳ The **surface grip factor** values per [C24](../C24-Surfaces/C24-Surfaces.md) surface type as consumed by
  the tyre.
- ⏳ Special-surface effects (the `adhesion` groups of [C24](../C24-Surfaces/C24-Surfaces.md) applied to
  tyres).

## Key takeaways

- Per-wheel grip is **handling traction × surface factor**: [C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md)'s
  `TractionMultiplier`/`Loss`/`Bias` scaled by the [C24](../C24-Surfaces/C24-Surfaces.md) surface under each
  tyre — the same car grips road and slides sand.
- Grip only applies to planted wheels (non-zero compression, [C47.1](01-four-wheel-suspension.md)); load
  transfer from the springs is why braking unloads the rear and enables slides.
- Skidmarks, burnout and wheel rotation are per-wheel `float[4]`/`bool[4]` arrays, so the sliding tyre is
  exactly the one that smokes and marks.

**Continue:** [C47.3 — Damage and deformation →](03-damage-and-deformation.md)
