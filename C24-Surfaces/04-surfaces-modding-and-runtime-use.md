# C24.4 — Surfaces at runtime and modding implications

## How the surface ID travels from collision to physics

Every COL collision face (C6) carries an 8-bit `surface_id` in its surface material byte. That ID is an index into the `surfinfo.dat` table (C24.2): 179 entries, each with an adhesion group, a friction coefficient, and a set of physics flags. The runtime path:

1. Collision detection (C6.4) returns a `CColPoint` with `nSurfaceTypeA` and `nSurfaceTypeB` — the surface IDs of both objects that touched
2. `CPhysical::ApplyCollisionAlt` looks up both IDs in `surfinfo`
3. The friction coefficient of the lower-grip surface governs the impulse calculation (vehicles skid on ice, not on ice AND concrete simultaneously)
4. The audio material flags from `surfaud.dat` (C24.3) select the impact sound (stone scrape vs. metal vs. mud)

This means a single `surface_id` byte on a COL face controls both the physics friction AND the sound effect — one value, two systems.

## The gang-territory surface ID convention

Gang territory (C22.4) does not use the zone system for the ground surface — it uses the collision surface IDs of the road/terrain beneath the player. The `surfinfo` flags include a `gangzone` bit that marks surfaces as gang-territory-eligible. When the game samples the surface under the player's feet, a `gangzone`-flagged surface in a gang territory zone activates the gang AI response.

This creates a modding constraint: custom map areas without correctly assigned surface IDs will not trigger gang territory mechanics, even if a `info.zon` gang zone is placed over them. Both the zone rectangle AND the surface flag are required.

## The C24 → C47 traction path

C47.2 documented that traction force = `fTractionMult × surface_grip × wheel_load`. The `surface_grip` component comes from `surfinfo.dat`'s `friction` column for the surface under each wheel. This value is looked up per-wheel per-frame using the collision point from the wheel's suspension raycast.

Changing surface friction in `surfinfo.dat` directly changes how vehicles handle on that surface type — a global change, affecting every vehicle on that surface simultaneously. This is the correct place to adjust "ice physics" (surface 16 = ICE in the stock table) vs. the per-vehicle `fTractionMult` which adjusts one vehicle globally.

## Modding surface properties

The `surfinfo.dat` file is a direct-parse text file. Each row is `surfaceID, adhesionGroup, friction, ...`. To change the friction of concrete:
1. Find the surface ID for TARMAC (ID 0 in the stock table)
2. Change the `friction` column value
3. All concrete collision faces immediately use the new friction (no compile step, takes effect on next game load)

**Surface ID 0 is the default** — faces with no explicit surface assignment fall back to ID 0. Poorly-authored COL files often have all faces at ID 0 (TARMAC), which silently gives correct audio for roads but wrong audio for everything else (a metal bridge sounds like a road).

## Limits and the modding boundary

The surface namespace is 179 entries (0–178). IDs 179–255 are out of range — the `surfinfo` table has no entry for them, and the lookup will either return a zero row or read adjacent memory. Modded COL files should not assign surface IDs above 178.

The `surfaud.dat` table (C24.3) has separate entries for 179 surfaces but its audio material IDs reference the audio system (C20). Assigning a surface in `surfinfo.dat` without a corresponding `surfaud.dat` row silently falls back to the default audio material — the surface will have correct physics but wrong sound.

## The surface-to-scorch connection

The engine marks surfaces as "burned" based on the surface ID. The fire damage system checks `surfinfo`'s flammability flag: surfaces flagged as combustible can be set on fire (grass, wood, carpet) while others cannot (metal, concrete). This flag is why SA's countryside burns and urban streets don't — not a special "grass zone" but a per-face flag in every grass collision model.

**Previous:** [C24.3 — `surface.dat` and `surfaud.dat`](03-friction-matrix-and-audio.md)  
**Up:** [C24 — Surfaces](C24-Surfaces.md)
