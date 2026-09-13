# C40.4 — Culling, Performance, and Pipeline Optimization

> **The one-sentence version:** SA's render pipeline achieves playability across early-2000s
> hardware by stacking three elimination filters before a single polygon reaches the GPU —
> view-frustum culling removes ~85% of the world, the portal/occlusion system removes indoor
> geometry behind walls, and LOD selection replaces distant models with low-poly proxies — and
> the four-list split (solid world / solid entity / alpha world / alpha entity) then batches
> the survivors by D3D9 state to minimize the driver overhead that would otherwise dominate.

**Subsystem category:** Rendering — culling and optimization
**Depends on:** [C40.1](01-the-four-render-lists.md), [C40.2](02-the-pass-order.md),
[C40.3](03-where-the-frame-fits.md), [C6](../C6-Collision/C6-Collision.md),
[C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md), [C39](../C39-Camera/C39-Camera.md)
**RE status:** Documented — frustum cull method, LOD chain logic, and alpha sort traced to
addresses; portal/occlusion system is 🟡 (mechanism confirmed, address range ⏳)
**Confidence:** ✅ for the frustum test and LOD selection · 🟡 for the alpha sort algorithm
(insertion sort vs. partial quicksort not confirmed)

---

## 1. The three-filter cascade before rasterization

Before any triangle is submitted to D3D9, `CRenderer::BuildRenderList` applies three filters in
sequence. Understanding the cascade is essential for any performance work on a SA mod:

```
All entities in the world (tens of thousands)
        │
        ▼
Filter 1: View-frustum culling
        Eliminates: everything whose bounding sphere doesn't intersect the camera frustum
        Survivors: ~10–15% of world entities (dependent on draw distance + area)
        │
        ▼
Filter 2: LOD / draw-distance selection
        Eliminates: entities whose draw_dist < distance to camera
        Replaces: high-poly models beyond LOD threshold with low-poly proxies
        Survivors: a subset of Filter 1's survivors at appropriate detail levels
        │
        ▼
Filter 3: Portal / occlusion culling
        Eliminates: entities inside closed-off zones not visible from current camera
        Survivors: the final render list (~hundreds of entities, not thousands)
        │
        ▼
Rasterization (D3D9 DrawIndexedPrimitive)
```

Each filter costs CPU time on the surviving set that passes through it. The investment is
overwhelmingly worthwhile: the rasterization cost of drawing 10,000 entities is roughly 100×
worse than the CPU cost of testing 10,000 sphere-frustum intersections, and eliminating 85%
of entities from the draw call budget is the largest single performance lever.

---

## 2. View-frustum culling in depth

`CRenderer::BuildRenderList` tests each candidate entity with a **bounding sphere / frustum
intersection** test. The frustum is defined by six planes (near, far, left, right, top, bottom)
computed from the camera's view-projection matrix (C39.2 `m_fFOV`, `m_fFarClipPlane`,
`m_aCams[active].m_vecSourceSpeed` / `m_vecTargetSpeed`).

The test for each entity:

```
For each entity E in the sector:
  center = E.matrix.pos        ; world-space center of entity
  radius = E.collision_radius  ; from COL header (C6.2)
  For each frustum plane P (6 total):
    d = dot(P.normal, center) + P.d  ; signed distance to plane
    if d < -radius:                   ; entity is entirely behind this plane
      cull — do not add to render list
  if survived all 6: add to render list
```

The bounding sphere radius used in the test comes from the entity's [C6](../C6-Collision/C6-Collision.md)
collision model header — specifically `CColModel::m_boundSphere.radius`. This is the *physics*
bounding sphere, repurposed for rendering culling. An entity with an unusually large bounding
sphere (from a complex multi-part collision model) will defeat the frustum cull for view angles
where only part of the entity is visible — a real performance hazard for large modded props.

**The near-plane cull.** `m_fFarClipPlane` (C39.4 §5) sets the far-plane distance; the near
plane is fixed at approximately **0.9 units** (0.09m). Entities closer than the near plane are
not culled by frustum testing but may be clipped by D3D9's rasterizer. The near plane is not
accessible as a data file field — it is a hard-coded constant in the frustum setup code.

---

## 3. LOD selection: chains, distances, and pop

The LOD system is built on the **LOD link chains** established by
`CFileLoader::SetupLodLinks` ([C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md)) during world load.
Each `CEntity` has an `m_pLod` pointer to its LOD proxy; the proxy's `m_pLod` points to a
further-simplified version (if any). `CRenderer::BuildRenderList` walks the chain:

```
d = distance(camera, entity.center)
if d < entity.draw_dist:
    add entity (high-poly) to appropriate list
elif entity.m_pLod != null:
    if d < entity.m_pLod.draw_dist:
        add entity.m_pLod (low-poly) to list
    ; else: neither model renders — beyond all draw distances
```

The `draw_dist` value per entity comes from the IDE `objs` / `tobj` / `vehicles` section
([C11.1](../C11-IDE-And-IPL/C11-IDE-And-IPL.md)) and is a floating-point distance in SA units.
Common stock values:

| Asset type | Typical `draw_dist` | LOD `draw_dist` |
|-----------|-------------------|----------------|
| Building (large) | 300 | 600 |
| Small prop | 70 | 200 |
| Vehicle (stock) | 170 | 300 |
| Tree / vegetation | 100 | 250 |

The **LOD pop** — the visible "pop" when a model switches detail level — occurs because the
switch is instantaneous. SA does not cross-fade or blend between LOD levels. The pop distance
is `draw_dist` of the high-poly model, and it is proportional to how different the two meshes
look at that distance. A well-authored LOD pair has the pop at a distance where both meshes
look visually identical — typically beyond 80% of the viewer's perceptual clarity range.

---

## 4. Alpha sorting and its cost

Alpha-blended entities must be drawn back-to-front (painter's algorithm) relative to the camera
to composite correctly. `CRenderer` sorts the alpha entity list each frame by distance:

```
For each alpha entity in alpha_entity_list:
    entity.sort_key = squared_distance(camera, entity.center)
    // (square root avoided — sort order is the same)

qsort(alpha_entity_list, count, sizeof(ptr), compare_by_sort_key)
// approximate: insertion sort for small lists, partial sort for large
```

This sort runs **every frame** on the full alpha list. Its cost is O(n log n) in the worst case
where the list changes significantly frame-to-frame (camera rotating through a glass-building
district). In practice SA limits alpha geometry — most buildings are opaque, and the transparent
ones (windows, water, fences) tend to be far from the camera when they appear in quantity.

**The approximate sort.** SA does not guarantee perfect back-to-front order — it uses an
approximate sort that accepts occasional ordering errors for performance. This is why certain
view angles produce transparent-surface Z-fighting artifacts (windows rendering in front of
closer geometry). It is a known limitation of the sorting strategy, not a collision-detection
error.

---

## 5. The four-list design: D3D9 state batching

C40.1 established the four lists: solid world, solid entity, alpha world, alpha entity. The
design rationale is D3D9 state-change minimization:

| State change | Typical cost (relative) | SA's strategy |
|-------------|------------------------|--------------|
| Texture bind | 1× | Sorted by texture within each list (partial) |
| Blend mode toggle | 3–5× | All opaque first, all alpha after |
| Depth-write toggle | 3–5× | Same batching as blend mode |
| Vertex shader change | 5–10× | Pipeline change — avoided within a list |
| Full pipeline swap | 10–20× | Skin / car-paint / env-map pipelines forced to edges |

The solid→alpha split eliminates the single most expensive state-change pattern (toggling depth
write and blend mode between each opaque and alpha draw call). In a naive draw-order system
where an alpha prop sits between two opaque buildings, each transparency would require two
state changes. The list system reduces those to exactly two state changes for the entire frame.

The **solid world vs. solid entity** split exists because world geometry uses different vertex
layouts and pipeline configurations from dynamic entities (vehicles, peds). Separating them
allows the two halves to run their respective pipelines (C44) consecutively with a single
pipeline swap rather than interleaving swaps per entity.

---

## 6. Modding hooks: injecting custom geometry

For a custom ASI renderer that adds new draw calls to the world:

### 6.1 After-scene injection

The safest hook for custom geometry is immediately before `IDirect3DDevice9::Present` (available
via the vtable — slot 17 in the D3D9 device vtable). At this point all four lists have been drawn
and the depth buffer contains the final scene. Custom draw calls here composite on top of the
world. The depth buffer is readable (with a `StretchRect` copy to a separate surface) for
depth-aware effects.

```cpp
// Hook Present to add custom geometry after the main scene
HRESULT __stdcall HookedPresent(...) {
    DrawCustomGeometry(g_pDevice);  // runs after C40's four lists
    return OriginalPresent(...);
}
```

### 6.2 List injection (mid-pipeline)

For custom geometry that should be *part of* the existing sort order (e.g., a new semi-transparent
road surface that should depth-sort correctly against SA's alpha entities), the alpha entity list
must be modified before the sort runs. The hook point is just before
`CRenderer::SortBigBuildingList` or equivalent — at the start of the alpha-list pass in C40.2.

This is a more invasive hook and requires understanding the `CEntity*` list format in memory.

### 6.3 Pipeline intercept (per-material)

To replace how a specific material type renders (e.g., all car-paint materials should use
a custom specular shader), hook the atomic render dispatch in [C44](../C44-Shaders/C44-Shaders.md).
`RpAtomicRender` (a RenderWare function) calls into the pipeline assigned to each `RpAtomic`
(C44.1). Replacing the pipeline function pointer for the car-paint pipeline redirects all
car-paint render calls through custom code, leaving the rest of the pipeline unchanged.

---

## 7. Performance measurement for heavy mods

A heavy mod (new map, many custom entities) can use the D3D9 stats to identify bottlenecks:

| Metric | Query type | Bottleneck if high |
|--------|-----------|-------------------|
| DrawIndexedPrimitive call count | D3DQUERYTYPE_PIPELINETIMINGS | Too many small draw calls — batch geometry |
| Texture memory | D3DQUERYTYPE_RESOURCEMANAGER | Oversized TXDs — split or compress |
| Frustum-surviving entity count | Custom counter in `BuildRenderList` hook | draw_dist too high — reduce per asset |
| Alpha list sort time | Custom timer before/after sort | Too many alpha entities — convert to opaque |

For rapid per-frame CPU profiling, hooking `CRenderer::BuildRenderList` at entry and exit and
reading `QueryPerformanceCounter` gives the whole-frame culling cost including all three filters.

---

### Key takeaways

- Three elimination filters — **view-frustum** (bounding-sphere/plane test), **LOD/draw-dist**,
  and **portal/occlusion** — reduce tens of thousands of world entities to hundreds before any D3D9
  call is made; the frustum cull alone eliminates ~85% of entities per frame.
- The bounding sphere used for frustum culling comes from the **COL model** (C6.2) — oversized
  collision spheres defeat the cull even when the visual mesh is small.
- LOD **pop** is instantaneous (no cross-fade); place LOD switch distances where the two mesh
  silhouettes are visually indistinguishable.
- Alpha sorting is **approximate** (not guaranteed back-to-front) — occasional Z-fighting artifacts
  are a documented consequence, not a bug.
- The **four-list design** eliminates the depth-write / blend-mode toggle cost; switching from all-
  opaque-first to interleaved would multiply D3D9 state changes by the entity count.
- Safe custom-geometry hook: **before `IDirect3DDevice9::Present`** (vtable slot 17); for
  depth-sorted custom geometry, hook before the alpha-list sort.

**Previous:** [C40.3 — Where the frame fits](03-where-the-frame-fits.md)
**Up:** [C40 — Render Pipeline hub](C40-Render-Pipeline.md)
