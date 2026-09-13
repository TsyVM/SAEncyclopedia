# C25.3 — Physics scalars and their runtime roles

## What the continuous fields actually do

C25.1 established the grammar; C25.2 catalogued the values. This page closes the open ⏳ — the physical units and runtime interpretation of the seven continuous scalars. Each scalar's role is derivable from `CPhysical`'s integration code (C42.2) and how object.dat is consumed at runtime.

## The seven scalars and their engine roles

### mass (field 2, `%f`)

**Runtime role:** used as `1.0 / mass` (inverse mass) in the rigid-body impulse solver. When an entity receives a force impulse (from a vehicle collision, explosion, or player punch), the resulting velocity change is `F × (1/mass)`. The inverse mass scales the response.

**The 99999.0 sentinel:** 514 objects are authored with mass 99999.0 — the de-facto "immovable" value documented in C25.2. At `1/99999.0 ≈ 0.00001`, the inverse mass is so small that any plausible force impulse produces negligible velocity. The physics engine does not special-case this value — it simply produces near-zero velocity, which is indistinguishable from static.

**The header's "1–50000 kg" cap:** is the authoring guideline, not an engine constraint. The `sscanf` parser accepts any float and the physics engine accepts any positive inverse mass. The cap exists because values above ~50,000 produce the same effective result as 99999 at lower precision, but the engine never enforces it.

**Modding note:** to make an object heavier (resists player kicks and vehicle bumps), increase mass toward 99999. To make a prop lighter and more reactive (a trash can, a cardboard box), use values in the 10–100 range.

### turn_mass (field 3, `%f`)

**Runtime role:** the rotational analog of mass — the moment of inertia. Used as `1.0 / turn_mass` in the torque solver. A high turn_mass means the object resists spinning; low turn_mass means it spins easily. For most props, turn_mass is authored close to mass. An elongated object (a telegraph pole) has a higher turn_mass than a cube of the same total weight.

### air_resistance (field 4, `%f`)

**Runtime role:** per-frame velocity damping. Each frame, the object's velocity is multiplied by `1.0 - air_resistance × ms_fTimeStep` (approximately). Values near 1.0 give the object minimal drag; values near 0 kill velocity quickly. Most objects in the file cluster near 1.0 (low drag), which is appropriate for hard props. Soft or lightweight objects (newspapers, leaves) would use lower values.

**The `ms_fTimeStep` interaction:** because this is a per-frame multiplier, air resistance is timestep-scaled and is therefore frame-rate independent — the same velocity decay happens at 30fps or 60fps.

### elasticity (field 5, `%f`)

**Runtime role:** the coefficient of restitution for collisions. When two objects collide, elasticity determines how much velocity is reflected. A value of 0.0 is perfectly inelastic (no bounce); 1.0 is perfectly elastic (full rebound). Most objects are authored near 0.0–0.2 to prevent unrealistic bouncing. Balls and rubber objects would be higher.

### percent_submerged (field 6, `%f`)

**Runtime role:** the depth percentage of the object's bounding volume that must be underwater before buoyancy forces kick in. The value is a percentage (the header says 10–120, with `dump1` at 150 — C25.2). At 50% (the file's modal value), the object starts floating/sinking when half submerged. At 10%, buoyancy activates with very shallow submersion. At values >100%, the object must be more than fully submerged — effectively pinned underwater for most practical depths.

**The `dump1` exception:** 150% means the object needs to be 1.5× its height underwater before buoyancy activates — a deliberate design decision to make the dump truck sink and stay sunk rather than floating.

### uproot_limit (field 7, `%f`)

**Runtime role:** the minimum force magnitude required to dislodge a "fixed" or "static-until-hit" object from its placement. Objects with `cdamage_effect = 0` (standard) use `uproot_limit` as a threshold — forces below the limit have no effect; forces above it begin moving the object. For lamp posts and traffic lights (which have very high uproot_limit values), only a vehicle at significant speed can knock them over. For loose props, low values make them easily moved.

### cdamage_multiplier (field 8, `%f`)

**Runtime role:** scales how much of an incoming collision impulse is converted to damage on the object. A value of 1.0 means full collision-force-to-damage conversion. Values greater than 1.0 amplify damage (fragile objects); values near 0.0 make objects collision-tolerant. This multiplier feeds into the damage system rather than the velocity integration — it affects how quickly the object's health (if it has the breakable collision codes) degrades under repeated impacts.

## How object.dat data reaches the runtime

The load path:
1. `CObjectData::Initialise` reads `object.dat` (the `sscanf` loop at `0x5B5360`)
2. Each parsed record is stored in a `CObjectData` array indexed by model ID
3. When a `CObject` is created (`CPools::ms_pObjectPool.New()`), its model ID is used to look up the `CObjectData` entry
4. `CPhysical` picks up `mass`, `turn_mass`, `elasticity`, and `air_resistance` from the `CObjectData` record for its integrator
5. `cdamage_multiplier` and `cdamage_effect` are used by the collision-damage handler when another entity hits this object
6. `uproot_limit` is checked by the impulse application code before velocity is modified

## The breakable scalars: break_smash_mult through break_sparks

The 7 break-info fields (present only in breakable records — C25.1) are not physics integrator inputs. They govern what happens when the object's health falls to zero:

| Field | Role |
|---|---|
| `break_smash_mult` | Multiplies the velocity imparted to fragments when the object shatters |
| `break_vx/vy/vz` | Initial velocity direction of debris fragments |
| `break_v_rand` | Randomness factor for fragment velocity direction |
| `break_gun_mode` | How gunfire interacts with the breakable: 0=no gun damage, 1=gun shatters, 2=gun chips |
| `break_sparks` | 1=create sparks effect on break |

These fields feed the particle/effect system (C21) rather than CPhysical. When a breakable object's health reaches zero, its destruction handler reads these to spawn the correct fragment velocities and particle effects.

## The `special_cdr` field (J)

`cdamage_effect`'s companion field `special_cdr` (special collision-damage response) is an integer flag that modifies how the collision response *behaves* rather than just *damages*. The documented values 0–9 cover: standard response (0), smash window (1), hit peds-only (2), lamppost style (3), etc. The one out-of-range value (`imy_bbox = 20`) falls through the switch to the default handler — a case where an undocumented value in a shipped file happens to be handled gracefully by a default branch.

## Key takeaways

- `mass` and `turn_mass` drive inverse-mass impulse response — 99999.0 is not special to the engine, just produces near-zero velocity change.
- `air_resistance` is a per-frame velocity damping factor that is `ms_fTimeStep`-scaled and therefore frame-rate independent.
- `elasticity` is the coefficient of restitution — 0 = no bounce, 1 = full bounce.
- `percent_submerged` gates when buoyancy forces activate relative to submersion depth.
- `uproot_limit` is the minimum force to dislodge a prop.
- `cdamage_multiplier` scales collision-force-to-damage conversion (affects breakability rate).
- Break-info scalars feed the destruction/particle system, not CPhysical.

**Previous:** [C25.2 — The census and its named exceptions](02-census-and-exceptions.md)  
**Continue:** [C25.4 — Modding object.dat →](04-modding-object-dat.md)
