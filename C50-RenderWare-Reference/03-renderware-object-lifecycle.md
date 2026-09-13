# C50.3 — The RenderWare object lifecycle in SA

## How a model goes from disk to screen

Every model in SA exists as a tree of RenderWare objects. Understanding the lifecycle of these objects — when they are created, how they are connected, and when they are freed — explains the behavior of the streaming system, the render pipeline, and the memory budget simultaneously.

## The object hierarchy

```
RwEngine
 └─ RpWorld (the scene graph)
     └─ RpClump  (one per model instance — "the car")
         ├─ RwFrame  (transformation node — position/rotation)
         └─ RpAtomic  (one renderable mesh)
             ├─ RwFrame  (sub-object frame, e.g. wheel)
             └─ RpGeometry  (the vertex/index data)
                 └─ RpMaterial (per-surface material)
                     └─ RwTexture → RwRaster (the pixel data)
```

This hierarchy is decoded in C50.1. The lifecycle maps to the streaming system as follows.

## Phase 1: Streaming arrival

When CStreaming (C2) determines a model is needed, it requests the `.dff` file from the CdStream layer (C1). The `.dff` is a RenderWare binary stream — a sequence of typed chunks (C7). The stream parser (`RpClumpStreamRead`) builds the `RpClump` object tree from the binary data.

After parsing, `CModelInfo::AddModel` (C3) stores a pointer to the `RpClump` in the model-info table. The model is now "in memory" — it can be instanced.

## Phase 2: Instancing

When an entity (a ped, vehicle, prop) is created in the world, `CPlaceable::SetModelIndex` or equivalent sets the entity's model reference to the loaded `RpClump`. The entity does not own the `RpClump` — it holds a reference to the shared clump in the model-info table.

For dynamic entities (vehicles, peds), the engine clones (instances) the geometry data so the entity can have its own per-instance state (damage state, color). For static props (buildings, trees), the engine shares the same `RpGeometry` across all instances — only the `RwFrame` (position/rotation) differs per instance.

## Phase 3: Per-frame rendering

`CRenderer` (C40) builds the render list each frame. For each atomic in view:
1. The `RwFrame`'s world transformation matrix is computed (from the entity's position)
2. `RpAtomicRender` is called — this dispatches to the RenderWare pipeline (the P2 pipeline, C44)
3. The pipeline submits draw calls to Direct3D 9 using the `RpGeometry`'s vertex buffer and the `RpMaterial`'s shader

The per-frame cost is proportional to the number of visible `RpAtomic`s — each atomic is one draw call. High-poly mods that increase geometry density increase the draw-call count proportionally.

## Phase 4: Unloading

When `CStreaming` decides a model is no longer needed (it is out of the streaming distance), `CModelInfo::RemoveModel` is called:
1. `RpClumpDestroy` is called on the stored clump pointer
2. RenderWare frees the `RpGeometry`, `RpMaterial`, `RwTexture`, and `RwRaster` objects
3. The model-info entry's clump pointer is set to null
4. The 224-sector streaming buffer slot is freed (C2)

If any entity still holds a reference to the freed clump (i.e., an entity was not cleaned up before the model was unloaded), reading that reference causes undefined behavior. This is the root cause of the "streaming pop" crash — an entity with a dangling model reference gets rendered, accesses freed geometry, and crashes.

## The texture lifecycle: RwTexture and RwRaster

Textures follow the same streaming path but are stored in texture dictionaries (TXD, C9):
1. `CTxdStore` (C3) holds the loaded TXD as an `RwTexDictionary` — a collection of `RwTexture` objects
2. Each `RwTexture` owns an `RwRaster` — the pixel data, uploaded to VRAM on first use
3. When the TXD is unloaded, `RwTexDictionaryDestroy` frees all textures and rasters
4. D3D9 VRAM is reclaimed when the `RwRaster`'s backing `IDirect3DTexture9` is released

The GPU upload happens lazily — the `RwRaster` is uploaded to VRAM the first time it is used in a draw call, not when it is parsed from disk. This means the first frame a model is visible may have a slight stutter as its textures are uploaded.

## Why the hierarchy matters for modding

- **Model replacement:** replacing a `.dff` file in the IMG archive replaces the `RpClump` that gets built from it. If the replacement has a different geometry structure (different number of atomics, different material count), the entity using it must be compatible with the new structure — particularly for peds and vehicles with animation rigs.
- **Texture replacement:** replacing a `.txd` in the archive replaces the `RwTexDictionary` for that model. The texture names within the TXD must match what the model's materials reference — a material looks up its texture by name in the active dictionary.
- **Plugin extensions:** the 5 RenderWare plugins compiled into SA (skin2, hanim, matfx, uvanim, anisot — C50.2) add extra data to the standard RW object types. A `.dff` with skin2 data requires the skin2 plugin to parse and render correctly. Third-party DFF tools must support these SA-specific plugin extensions.

**Previous:** [C50.2 — Plugins, the driver, and pipelines](02-plugins-driver-pipelines.md)  
**Continue:** [C50.4 — The RenderWare data flow in SA →](04-renderware-data-flow-in-sa.md)
