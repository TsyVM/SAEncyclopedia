# C47.1 — The four-wheel suspension

Where [C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md) sized the handling *parameters*, this page reads
the *state* they act on: the four per-wheel suspension springs `CAutomobile` integrates every frame. The
model is four wheels, each a spring that compresses under load, expressed as parallel `float[4]` arrays.

## The wheel arrays

`CAutomobile` stores wheel data as a bank of parallel arrays, one entry per wheel, four wheels
(`MAX_CARWHEELS = 4`: front-left 0, rear-left 1, front-right 2, rear-right 3). Every array is a `float[4]`
(or `bool[4]`/`CColPoint[4]`), and they sit consecutively:

| Offset | Array | Meaning |
|---|---|---|
| `0x724` | `m_wheelColPoint[4]` (`CColPoint`) | where each wheel touches the ground ([C6](../C6-Collision/C6-Collision.md)) |
| `0x7D4` | **`m_fWheelsSuspensionCompression[4]`** | **compression, `[0..1]`** (0 = fully compressed, 1 = relaxed) |
| `0x7E4` | `m_fWheelsSuspensionCompressionPrev[4]` | last frame's compression (for damping) |
| `0x828` | `m_wheelRotation[4]` | spin angle |
| `0x838` | `m_wheelPosition[4]` | visual wheel position |
| `0x858` | `m_fWheelBurnoutSpeed[4]` | wheelspin speed |
| `0x878` | `m_aSuspensionSpringLength[4]` | `SuspensionUpperLimit − SuspensionLowerLimit` |
| `0x888` | `m_aSuspensionLineLength[4]` | `springLength + wheelSize/2` |

## The compression is the core state

The one that matters is `m_fWheelsSuspensionCompression[4]` at **`0x7D4`**. It is the current compression of
each wheel's spring, normalised to `[0..1]` — **0 means the spring is fully compressed** (the car is bottomed
out on that corner), **1 means fully relaxed** (the wheel is hanging or unloaded). The constructor fills all
four with `1.0` (a car spawns with relaxed springs), and every frame `ProcessSuspension` recomputes them from
the wheel's collision point against the ground.

That this is *the* per-wheel value shows in how often it is touched: `derive_vehicle_dynamics.py` counts the
offset `0x7D4` referenced **15 times** in the vehicle-physics code region (`suspension_compression_used`) —
far more than any sibling array — because everything downstream reads it: the suspension force pushed back
into the [C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md) `CPhysical` body, the ride height for the visual
wheel position, the body pitch and roll under braking and cornering, even whether a tyre is planted enough to
have grip.

## The arrays tile at 0x10

Because each array is exactly four 4-byte floats, they step by `0x10`, and the pairs that belong together sit
adjacent:

```
m_fWheelsSuspensionCompression  0x7D4 + 0x10 = 0x7E4  m_fWheelsSuspensionCompressionPrev
m_aSuspensionSpringLength       0x878 + 0x10 = 0x888  m_aSuspensionLineLength
```

`derive_vehicle_dynamics.py` asserts both (`wheel_arrays_tile_float4`). The **current/previous** compression
pair is how the damper works: the *change* in compression between frames (`compression − compressionPrev`) is
the spring's velocity, and the damping force opposes it — a spring-damper per wheel, stored as two adjacent
`float[4]`s.

## Spring length vs line length

Two different "lengths" describe the strut, and the distinction is deliberate. **`m_aSuspensionSpringLength`**
(`0x878`) is `SuspensionUpperLimit − SuspensionLowerLimit` — the travel of the spring itself, straight from
the [C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md) `tHandlingData` limits.
**`m_aSuspensionLineLength`** (`0x888`) is `springLength + wheelSize/2` — the full raycast length from the
strut mount down past the wheel's radius. The physics casts a ray of `lineLength` downward to find the
ground; where it hits determines the compression `[0..1]` back in the spring's travel. So the handling file's
`Upper/LowerLimit` fields ([C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md)) literally set how far each of
these four springs can move — lower the limits and the car sits lower and bottoms out sooner.

## How a handling row becomes a bouncing car

Putting it together: `tHandlingData` ([C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md)) supplies
`SuspensionForceLevel`, `DampingLevel`, `UpperLimit`, `LowerLimit`, `BiasBetweenFrontAndRear` and
`AntiDiveMultiplier`. The spring/line lengths are computed from the limits; each frame, `ProcessSuspension`
raycasts each wheel, sets the `[0..1]` compression, computes a spring force (`ForceLevel × compression`) minus
a damping force (`DampingLevel × (compression − prev)`), and applies it at the wheel through the
[C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md) `CPhysical` body. Four of those, biased front/rear, are
what make a car squat under acceleration, dive under braking, and lean in turns — all from the six handling
fields and these four arrays.

## Key takeaways

- A car has **four wheels** stored as parallel `float[4]` arrays; the core state is
  `m_fWheelsSuspensionCompression[4]` @`0x7D4`, `[0..1]` (0 = compressed), referenced 15× in the physics code.
- The arrays tile at `0x10`; the current/previous compression pair is the spring-damper (velocity =
  compression − prev), and spring length vs line length separates spring travel from the ground raycast.
- The handling file's `Suspension*` fields ([C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md)) set these
  springs' limits and forces — C42 tunes, C47 integrates.

**Continue:** [C47.2 — Tyres and traction →](02-tyres-and-traction.md)
