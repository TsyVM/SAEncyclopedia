# C50.4 — The RenderWare Data Flow From Format to Frame

> **The one-sentence version:** a `.dff` model travels from disk bytes to D3D9 draw calls through
> four well-defined stages — binary stream parse → plugin data attachment → pipeline assignment
> → per-frame submit — and modding this pipeline means intercepting at one of those four stages;
> the D3D9 resource boundary (the point where data leaves CPU RAM and enters VRAM) is the key
> architectural divide.

**Subsystem category:** Rendering — RenderWare pipeline data flow
**Depends on:** [C50.1](01-the-object-model.md), [C50.2](02-plugins-driver-pipelines.md),
[C50.3](03-renderware-object-lifecycle.md),
[C7](../C7-RenderWare-Stream/C7-RenderWare-Stream.md) (DFF binary format),
[C8](../C8-Geometry/C8-Geometry.md) (geometry structure),
[C9](../C9-Materials-And-Textures/C9-Materials-And-Textures.md) (texture loading),
[C40](../C40-Render-Pipeline/C40-Render-Pipeline.md) (the renderer that calls the pipeline),
[C44](../C44-Shaders/C44-Shaders.md) (the D3D9 vertex shader pipeline)
**RE status:** Documented — pipeline stages traced from DFF parse through D3D9 submit
**Confidence:** ✅ for the four-stage structure and D3D9 resource boundary ·
🟡 for the exact `RwD3D9SubmitNode` call sequence (name confirmed, parameter layout inferred)

---

## 1. Stage 1: Binary stream → `RpClump` object tree

A `.dff` file is a sequence of typed, nested **RenderWare chunk** pairs (C7). The streaming parser
(`RwStreamRead`) walks the byte stream:

```
12-byte chunk header: {type (uint32), dataSize (uint32), version (uint32)}
dataSize bytes of payload
    (optionally, child chunk headers+payloads nested inside)
```

The parser dispatches each top-level chunk type to the appropriate constructor:

| Chunk type | Constructor | Result |
|---|---|---|
| `0x10` (Clump) | `RpClumpStreamRead` | `RpClump*` — the whole model |
| `0x0F` (Geometry) | `RpGeometryStreamRead` | `RpGeometry*` — vertices, faces |
| `0x07` (Material) | `RpMaterialStreamRead` | `RpMaterial*` — colour + texture ref |
| `0x06` (Texture) | `RwTextureStreamRead` | `RwTexture*` — name handle for lookup |
| `0x0E` (Frame List) | (internal) | `RwFrame*` hierarchy |
| `0x0D` (Atomic) | `RpAtomicStreamRead` | `RpAtomic*` — geometry + frame binding |

By the time the last byte is consumed, the full `RpClump` exists as a connected object tree in CPU
memory. All `RwFrame` parent-child links are established (the hierarchy that lets a wheel frame
rotate while the body frame stays fixed), and all `RpAtomic` → `RpGeometry` bindings are live.

### 1.1 The version guard

Every chunk header carries a **library version** field. SA's parser expects chunks in the RW 3.x
range (the `RW36Active` builds — C50.2). A `.dff` exported with RW 2.x or RW 4.x chunk versions
will fail the version check, `RpClumpStreamRead` returns null, and the model silently does not load.
This is the most common cause of custom content "not appearing" in SA — a format tool set to the
wrong target version.

**Diagnostic:** read the first 12 bytes of any `.dff`: bytes 8–11 (little-endian uint32) are the
library version. Values `0x34000` to `0x36003` (RW 3.4 to 3.6.3) are SA-compatible; outside this
range, the model will not load.

---

## 2. Stage 2: Plugin data attachment

Each RW object type has an **extension section** at the end of its chunk data. This section
contains zero or more plugin-specific data blocks, each identified by a plugin ID:

| Plugin ID | Plugin name | Data attached to | Content |
|---|---|---|---|
| `0x116` | `skin2` (rpSkin) | `RpGeometry` | Bone weights + indices per vertex |
| `0x11C` | `hanim` (rpHanim) | `RwFrame` | Hierarchical animation rig (bone-to-frame map) |
| `0x120` | `matfx` (rpMatFX) | `RpMaterial` | Dual-texture, env-map, bump-map flags |
| `0x180` | `uvanim` (rpUVAnim) | `RpMaterial` | UV animation keyframe tracks |
| `0x18F` | `anisot` (rpAnisot) | `RpMaterial` | Anisotropic filtering flag |

During stream read, each plugin's registered stream-read callback fires when its ID appears in the
extension section. The callback allocates and fills its plugin-specific block. If an extension
section is absent (no plugin data), the plugin's default state is used (no skinning, no env-map,
standard filtering).

### 2.1 Plugin data mismatch bugs

The most common plugin mismatch: a geometry has `skin2` bone weights but its attached `RwFrame`
hierarchy lacks an `hanim` rig. The skinned vertex shader reads bone-transform matrices from the
`hanim` rig during render; a missing rig means zero-matrix transforms → the geometry collapses to
the world origin. This is the "model disappears" symptom that occurs when a ped DFF is ported from
SA to a game with a different rig hierarchy.

---

## 3. Stage 3: Registration, pipeline assignment, and the D3D9 boundary

After streaming, `CModelInfo::AddModel` stores the `RpClump*`. SA then iterates the clump's atomics:

```cpp
// Pseudocode — actual implementation in SA's model-install path
RpClumpForAllAtomics(pClump, [](RpAtomic* pAtomic, void* data) -> RpAtomic* {
    // Assign render pipeline based on material flags (C44)
    RwD3D9Pipeline pipe = SelectPipeline(pAtomic);
    RpAtomicSetPipeline(pAtomic, pipe);  // assigns the VS pipeline pointer

    if (IsStaticGeometry(pAtomic)) {
        // Upload vertex buffer to VRAM now — one-time cost
        RwD3D9GeometryInstanciate(pAtomic->geometry);
        // pAtomic->geometry->vertices now points to a D3D9 vertex buffer
    }
    // Dynamic geometry (peds, vehicles): remains in system RAM; re-uploaded per frame
    return pAtomic;
}, nullptr);
```

### 3.1 The D3D9 resource boundary

The call to `RwD3D9GeometryInstanciate` (for static geometry) is the **D3D9 resource upload
point** — the moment geometry data crosses from CPU system RAM into GPU VRAM (an
`IDirect3DVertexBuffer9` or `IDirect3DIndexBuffer9`).

This boundary has critical implications for modding:

| Modding operation | Above or below boundary | Technique |
|---|:--:|---|
| Change vertex positions in a running mod | Above (system RAM copy) | Write to `RpGeometry` vertex array directly |
| Replace a texture at runtime | Below (VRAM surface) | `IDirect3DTexture9::Lock` → fill → `Unlock` |
| Animate a building's UV coordinates | Above (for dynamic geometry) | Write UV offsets to `RpGeometry` material |
| Custom vertex color per instance | Above | Write to `RpGeometry` prelit array |
| Gamma/bloom post-process | Below | Hook `Present` (vtable 17, C44.4) and apply to backbuffer |

Static geometry is uploaded once and cannot be cheaply modified per-frame without locking the VRAM
buffer (expensive). Dynamic geometry (peds, vehicles) stays in system RAM and is re-uploaded each
frame — modifications to their vertex data in system RAM take effect on the next frame automatically.

### 3.2 Pipeline selection logic (C44)

`SelectPipeline` examines the atomic's material flags to choose the RW D3D9 pipeline (C44.1):

| Condition | Pipeline selected |
|---|---|
| `matfx` env-map flag set | `CustomEnvMapPipe` — the SA vehicle-paint reflection pipe |
| Skin (bone weights present) | `skin-AllInOne` — the skinned vertex shader path |
| Neither | `AllInOne` (standard world/atomic pipe) |

The pipeline pointer stored by `RpAtomicSetPipeline` is what `RpAtomicRender` dispatches to at
render time. A mod that wants to intercept an individual atomic's rendering calls
`RpAtomicSetPipeline` with a custom pipeline pointer (C44.4 Depth 2 hook).

---

## 4. Stage 4: Per-frame submit

When `CRenderer` (C40) places an atomic in the visible entity list and the render pass reaches it:

```
CRenderer submits atomic
    │
    ▼
RpAtomicRender(pAtomic)
    │
    ├── calls the atomic's pipeline (pointer from RpAtomicSetPipeline)
    │       │
    │       ├── sets VS constants (world matrix, VP matrix, light params) → c0–c11
    │       ├── optionally calls RwD3D9SubmitNode (custom pipeline path)
    │       └── calls DrawIndexedPrimitive → D3D9 rasterizer
    │
    └── (for dynamic geometry): re-uploads vertex data to VB before draw
```

### 4.1 The VS constant layout (C44.2)

The vertex shader constants set by the pipeline before the draw call:

| Constant register | Content | Writable by mod? |
|---|:--:|:--:|
| `c0`–`c3` | World matrix (4 × float4 rows) | ❌ Read-only (engine writes) |
| `c4`–`c7` | View-projection matrix | ❌ Read-only |
| `c8`–`c11` | Lighting direction/colour | ❌ Read-only |
| `c12`+ | (free) | ✅ Custom shader constants |

A custom pixel shader or vertex shader attached via the Depth 2/3 hooks (C44.4) can read `c0`–`c11`
for the engine-set transform/lighting but must use `c12`+ for any custom parameters.

### 4.2 `RwD3D9SubmitNode` — the inner submit

`RwD3D9SubmitNode` is the innermost function that actually calls `DrawIndexedPrimitive`. Hooking it
(by patching the call site inside a specific pipeline function) gives the finest-grained control
over what is drawn — the geometry and material are known at this point and can be inspected or
replaced. This is the C44.4 **Depth 2** hook location.

---

## 5. Texture streaming: the TXD side of the data flow

Models and textures are loaded separately. When `CRenderer` first considers drawing an entity:

1. `CStreaming` checks if the entity's TXD is loaded (C2)
2. If not: the request goes into the streaming queue; the entity may draw with a white/missing texture
   on this frame
3. If yes: `RwTexDictionary::SetCurrent(txd)` makes the TXD active; the material's `RwTexture` name
   lookup finds the `RwRaster` inside the TXD

The texture lookup is by **name** — the `RwTexture` carries the name string; `RwTexDictionaryFindTexture`
scans the current TXD for a match. This is why texture names must be globally unique in the active
TXD stack — a texture named `wheel_128` in two different TXDs will produce an ambiguous result.

---

## 6. Modding entry points, summarised

| Stage | Mod entry point | Effect |
|---|---|---|
| 1: parse | Replace `.dff`/`.txd` file in `gta3.img` | Change geometry or texture before it ever loads |
| 2: plugin | Write to `RpGeometry` plugin data after `AddModel` | Change skinning weights, UV animation |
| 3: assignment | Call `RpAtomicSetPipeline(pAtomic, customPipe)` | Intercept one atomic's rendering |
| 4: submit | Hook `RwD3D9SubmitNode` | Intercept per-primitive D3D9 draw calls |
| 4: post-frame | Hook `IDirect3DDevice9::Present` (vtable 17) | Full backbuffer access for post-effects |

The further down the stage list the hook is, the more specific the interception — and the more
context is available (at Stage 4, you know exactly which geometry and material are being drawn).

---

### Key takeaways

- The data flow is: **disk bytes → `RpClump` object tree** (stream parse) → **plugin data
  attachment** (skinning, env-map, UV anim) → **D3D9 resource upload** (static geometry to VRAM;
  dynamic stays in RAM) → **per-frame submit** (`RpAtomicRender` → pipeline → `DrawIndexedPrimitive`).
- The **D3D9 boundary** (static geometry upload vs. dynamic re-upload) is the key architectural
  divide: above it, geometry modifications are cheap; below it (VRAM textures/buffers),
  modifications require `Lock`/`Unlock`.
- VS constants `c0`–`c11` are engine-reserved (world/VP matrix, lighting); custom shaders must use
  `c12`+ for their own parameters.
- Texture lookup is by **name** through the active TXD stack — globally unique names are required.
- The four-stage structure gives four distinct modding intervention points, from file replacement
  to per-draw-call interception.

**Previous:** [C50.3 — The RenderWare object lifecycle](03-renderware-object-lifecycle.md)
**Up:** [C50 — RenderWare Reference hub](C50-RenderWare-Reference.md)
