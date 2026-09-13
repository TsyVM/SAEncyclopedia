# C6.4 — Collision detection and physics integration

## The role of collision in the physics loop

Collision meshes (C6.1–C6.3) are data structures — they define the shape of the world. The physics engine (C42) uses them every frame to answer: "does this moving entity overlap with this geometry?" The answer drives resolution (pushing objects apart, applying friction, damaging vehicles).

`CCollision::ProcessCollisions` (the C51.4 Phase 5 function) is the frame-tick entry point. It:
1. Determines which entity pairs are close enough to need collision checking (broad-phase, using the sector system — C5)
2. For each close pair, tests their respective collision geometry for intersection (narrow-phase)
3. If intersecting, computes contact points and normals
4. Calls the physics engine with the contact data to resolve the penetration

## The broad-phase: sector-based culling

The world is divided into `CWorld` sectors (C5). Each sector holds a list of entities that overlap it. The broad-phase checks entity pairs only within the same or adjacent sectors — this reduces the O(n²) pair count to roughly O(entities_per_sector²), which is manageable even with hundreds of entities.

For moving entities (vehicles, peds, projectiles), the broad-phase also checks against static world geometry (CBuilding pool entities). Buildings use `CColStore`'s (C6.3) stored collision meshes; dynamic entities carry their own collision data from their model's `.col` file.

## The narrow-phase: sphere, box, and mesh tests

The narrow-phase uses a hierarchy of shape tests:
1. **Sphere-sphere:** bounding-sphere test (fastest). The bounding sphere is stored in the COL header (C6.2, the 28-byte bounds). If spheres don't overlap, no further testing.
2. **Box-box:** bounding box test (fast). The bounding box is also in the COL header.
3. **Mesh-mesh:** triangle-level test (expensive). Only reached if both bounding tests pass.

For vehicles vs world geometry, the narrow-phase runs sphere tests first (using the vehicle's approximate bounding sphere), then triangle tests for close-enough cases. The COL mesh format (C6.1) stores triangles with per-triangle surface properties (the `surface_id` connecting to C24's surface table), so the collision result also returns what material was struck.

## What a collision contact produces

A contact point carries:
- World-space position of the contact
- Contact normal (the surface's outward-pointing direction)
- Penetration depth (how far the objects overlap)
- Surface ID (from C24's surface namespace — determines friction, damage multiplier, audio)

The physics engine (C42) uses the contact normal and depth to compute the impulse that separates the objects. The surface ID determines the restitution (bounciness) and friction coefficients applied to the impulse.

## The COL–surface connection (C24)

Every COL triangle has a `surface_id` byte (the C6 format encodes this in the material index). The surface ID indexes C24's surface table (`surfinfo.dat` + the per-surface physics record). This is the connection between geometry and material properties:
- A COL triangle with `surface_id = SAND` → low friction, no damage multiplier
- A COL triangle with `surface_id = METAL_SOLID_LARGE` → high bounce, vehicle damage on impact

This linkage means modding the surface type of a collision mesh (by editing the COL file's material indices) changes the physical behavior without changing the geometry shape.

## Damage from collision

When a vehicle's collision contact exceeds the damage threshold (a function of relative velocity at contact × contact mass), `CVehicle::ProcessCarPhysics` (C42/C47) calls the vehicle's damage system:
- Body panel deformation (C45)
- Health damage
- Sound (tire screech, impact sounds keyed by surface ID via C20)

The COL mesh is not modified by damage — the visual deformation (C45's mesh morphing) is separate from the collision geometry. A visually-crushed vehicle still has its original COL bounding shape.

## Patching collision meshes

To change a world-prop's collision shape:
1. Extract the prop's `.col` file from `gta3.img` (C1) — either a standalone `.col` or the embedded COL data in a `.dff`
2. Edit the triangle mesh using a COL editor
3. Re-import to the archive

The COL mesh is version-aware (C6.1 documents COL1/COL2/COL3 differences). SA uses COL3 for most complex meshes. A COL editor must output the correct version header or `CColStore::AddCollision` will silently reject the mesh.

**Previous:** [C6.3 — CColStore and binding](03-colstore-and-binding.md)  
**Up:** [C6 — Collision](C6-Collision.md)
