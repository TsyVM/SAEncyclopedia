# C44.4 — Custom Pipeline and Visual Modding

> **The one-sentence version:** SA's vertex-shader pipeline is a RenderWare dispatch table —
> each `RpAtomic` carries a `pipeline` pointer that routes it through one of five assembly-
> time-built D3D9 vertex shaders, and a mod can intercept at any of three depths: after
> `Present` for whole-frame post-processing, at the pipeline dispatch for per-material shader
> replacement, or at the geometry upload for per-vertex data modification — each depth unlocks
> different effects at proportionally higher integration complexity.

**Subsystem category:** Rendering — shader modding
**Depends on:** [C44.1](01-the-pipelines.md), [C44.2](02-register-map-and-transform.md),
[C44.3](03-runtime-assembly.md), [C40](../C40-Render-Pipeline/C40-Render-Pipeline.md),
[C8](../C8-Geometry/C8-Geometry.md)
**RE status:** Documented — all three hook depths confirmed by method addresses
**Confidence:** ✅ for the Present-hook and pipeline-dispatch-hook patterns · 🟡 for the
exact `RpAtomicRender` vtable slot (confirmed in range, exact slot ⏳)

---

## 1. The pipeline dispatch: where SA decides which shader to use

The core architectural fact for visual modding is how SA selects a vertex shader per draw call.
Every `RpAtomic` (a mesh instance in RenderWare — [C8.1](../C8-Geometry/01-geometry-structure.md))
has a **pipeline function pointer** assigned at load time:

```cpp
struct RpAtomic {
    RpAtomicPipeline* pipeline;   // function pointer: which pipeline to use
    RpGeometry*       geometry;   // vertex/index data (C8)
    RwFrame*          frame;      // world-space transform
    // ... other fields
};
```

`CFileLoader` assigns the pipeline based on the geometry's material flags when the model is
loaded from the `.dff` archive (C7). The assignment map (C44.1):

| Material flag | Pipeline assigned |
|--------------|------------------|
| No special flag | Standard (plain diffuse) |
| `rsc_env` | Environment-map (reflective surfaces) |
| `rsc_skin` | Skin (ped skeleton deformation) |
| `rsc_car` | Car-paint (layered paint + specularity) |
| `rsc_2dfx` | 2D effect sprites (C10) |

When `CRenderer::RenderOneObject` (C40.2) processes an `RpAtomic`, it calls through the
pipeline pointer. The pipeline function sets the D3D9 vertex shader, uploads constant registers
(C44.2), and issues `DrawIndexedPrimitive`. Replacing the pipeline pointer on a specific
`RpAtomic` is the clean hook point for per-material shader replacement.

---

## 2. Depth 1 — Present-hook for post-processing

The most accessible visual mod hook is `IDirect3DDevice9::Present`, vtable slot 17. At the
point `Present` is called, all of C40's four render lists have been drawn and the back buffer
contains the final composited frame.

```cpp
typedef HRESULT (__stdcall* Present_t)(IDirect3DDevice9*, RECT*, RECT*, HWND, RGNDATA*);

HRESULT __stdcall HookedPresent(IDirect3DDevice9* pDev, ...) {
    // The back buffer is now complete. Read it for blur/bloom/colour grading:
    IDirect3DSurface9* pBB;
    pDev->GetBackBuffer(0, 0, D3DBACKBUFFER_TYPE_MONO, &pBB);

    // Copy to a staging surface for sampling:
    pDev->StretchRect(pBB, NULL, g_pStagingSurface, NULL, D3DTEXF_NONE);

    // Set staging surface as texture, draw a full-screen quad with a custom pixel shader:
    pDev->SetTexture(0, g_pStagingTexture);
    pDev->SetPixelShader(g_pPostProcessPS);
    DrawFullScreenQuad(pDev);

    return OriginalPresent(pDev, ...);
}
```

This approach powers ENB-style mods, ReShade, SilentPatch's graphical fixes, and the FX12 series.
Its limitation: it runs *after* the scene is complete, so it cannot change how individual materials
were rendered. Bloom applied here brightens pixels uniformly without distinguishing emissive from
reflective surfaces. HDR tone mapping via `Present` hook is physically inaccurate but fast.

**Depth buffer access.** The depth buffer at `Present` time is readable via `StretchRect` to a
`D3DFMT_D24S8` offscreen surface (requires the adapter to support `D3DUSAGE_RENDERTARGET` on a
depth format, which most D3D9 hardware supports). Depth reads enable:
- Screen-space ambient occlusion (SSAO)
- Depth-of-field blur (focus plane from depth)
- Edge detection (discontinuities in depth = silhouettes)

---

## 3. Depth 2 — Pipeline dispatch for per-material shader replacement

To change how a specific *type* of material renders — all car paints, all env-map surfaces, all
ped skins — the correct hook is at the **pipeline function pointer**. There are two interception
patterns:

### 3.1 Replace pipeline pointer at load time

```cpp
// Hook CFileLoader or RpAtomicSetPipeline to intercept pipeline assignment
void OnAtomicLoaded(RpAtomic* pAtomic) {
    if (IsCarPaintAtomic(pAtomic)) {
        pAtomic->pipeline = &MyCustomCarPaintPipeline;
    }
}
```

`MyCustomCarPaintPipeline` is a function that matches the RpAtomicPipeline signature:

```cpp
void MyCustomCarPaintPipeline(RwResEntryHeader*, void*, RwBool, RwUInt32) {
    // Set custom vertex shader
    g_pDevice->SetVertexShader(g_pCustomCarPaintVS);
    // Set custom constants in c0..c7 (C44.2)
    g_pDevice->SetVertexShaderConstantF(0, (float*)&g_WorldViewProj, 4);
    g_pDevice->SetVertexShaderConstantF(4, (float*)&g_SpecularParams, 1);
    // ... submit DrawIndexedPrimitive
}
```

This approach is used for normal-mapping mods, physically-based rendering (PBR) car paint,
and high-quality skin deformation.

### 3.2 Hook `RwD3D9SubmitNode`

A lower-level alternative: hook the RenderWare function that issues each D3D9 draw call
(`RwD3D9SubmitNode` at `0x77E900` — approximate; confirmed within the `0x77xxxx` range).
This runs *after* the pipeline has set its shader and constants but *before* `DrawIndexedPrimitive`.
A hook here can override shader constants (not the shader itself) for any draw call:

```cpp
void HookedSubmitNode(RxD3D9ResEntryHeader* header, ...) {
    // Modify shader constants for this specific draw call
    if (header->currentVertexShader == g_CarPaintShader) {
        float mySpecular[4] = { 0.9f, 0.9f, 0.9f, 64.0f };
        g_pDevice->SetVertexShaderConstantF(8, mySpecular, 1);
    }
    OriginalSubmitNode(header, ...);
}
```

This is less powerful than pipeline replacement (cannot change the shader itself) but simpler —
only one hook point, and it works for any pipeline without replacing the function pointer.

---

## 4. Depth 3 — Vertex buffer modification

For effects that require changing the vertex data itself — morph animations, procedural deformation,
per-vertex color from a runtime source — the vertex buffer must be modified before the pipeline
uploads it. SA stores geometry (C8) in system RAM for dynamic entities (vehicles, peds):

```
Workflow:
1. Get geometry: RpGeometry* pGeom = RpAtomicGetGeometry(pAtomic)
2. Lock: void* vertices = RpGeometryLock(pGeom, rpGEOMETRYLOCKVERTICES)
3. Modify: apply deformation to each vertex in the locked buffer
4. Unlock: RpGeometryUnlock(pGeom)
```

`RpGeometryLock` marks the geometry as "dirty" — on the next render, the pipeline re-uploads
the vertex buffer to the GPU. The cost per entity per frame is one system-memory write (fast)
plus one GPU buffer upload (moderate — proportional to vertex count). This is appropriate for
per-vehicle-instance deformation where each vehicle may have unique vertex positions.

**Damage deformation mods** use exactly this approach: the vehicle damage state (C45/C47.3) is
converted to per-panel vertex offsets, which are applied to the vertex buffer each frame via
the above workflow. The original vertex positions (pre-damage) must be cached in the mod's own
buffer to compute the delta correctly.

---

## 5. Custom pixel shaders: filling the fixed-function gap

SA's D3D9 pipeline uses **fixed-function pixel processing** — there are no `.hlsl` pixel shaders
in the shipped game. Every per-pixel effect (texture sampling, specular highlight, reflection) is
computed by D3D9's fixed-function texture stages, configured via `SetTextureStageState`. This is
why SA does not support per-pixel lighting, normal maps, or sub-surface scattering natively —
all of those require a pixel shader stage that SA never sets.

An ASI can attach a custom pixel shader to the device after any draw call in the pipeline:

```cpp
// After every car-paint DrawIndexedPrimitive:
g_pDevice->SetPixelShader(g_pNormalMapPS);
g_pDevice->SetTexture(1, pNormalMapTexture);  // extra texture sampler
g_pDevice->DrawIndexedPrimitive(...);          // re-draws with pixel shader
g_pDevice->SetPixelShader(NULL);              // restore fixed-function
```

This "re-draw" approach doubles the draw call count for affected materials — acceptable for
small counts (vehicle paint, a few key buildings) but prohibitive for the whole world. The
more efficient approach is to replace the pipeline (Depth 2, §3.1) so the custom pixel shader
is part of the original draw call rather than an extra pass.

---

## 6. The register map for custom shaders

Any custom vertex or pixel shader that participates in SA's pipeline must respect the register
assignments C44.2 documented. Violation breaks other materials in the same frame:

```
c0  = world-view-projection matrix column 0  (DO NOT override between SetVS and DrawIndexedPrimitive)
c1  = WVP column 1
c2  = WVP column 2
c3  = WVP column 3
c4..c7 = world matrix rows (object → world transform)
c8  = camera position in world space (available for specular highlight computation)
c9  = directional light direction
c10 = ambient + directional light colour
c11 = material specular constants
```

A custom shader that reads these in the standard order will correctly transform and light any
mesh. A custom shader that *writes* to these registers (some D3D9 shader ISAs allow this) will
corrupt the state for the next draw call — always treat c0–c11 as read-only inputs from the
game's perspective.

Custom constants can be freely assigned to **c12 and above** — SA's shaders do not use these
registers, and they persist across draw calls (D3D9 constant register state is not automatically
cleared). Set them once at the frame start:

```cpp
// Set custom lighting parameter once per frame before any render
float specularExponent[4] = { 64.0f, 0.0f, 0.0f, 0.0f };
g_pDevice->SetVertexShaderConstantF(12, specularExponent, 1);
```

---

## 7. LOD and visual quality: disabling LOD pop

C40.4 §3 covered LOD from a performance angle. For visual quality, the goal is eliminating pop.
The two strategies:

**Strategy A: Match LOD silhouettes.** Design LOD models whose silhouette matches the high-poly
at the switch distance. This requires care in the LOD mesh authoring but costs no performance.
Stock SA LODs are sometimes visually coarse at their switch distances — a custom mod can re-author
them to be silhouette-preserving.

**Strategy B: Patch `draw_dist` values.** Set both the high-poly and LOD `draw_dist` to the
same value, effectively disabling LOD — the model renders at full detail until it disappears.
Cost: at the old switch distance, polygon count doubles. This is appropriate for high-importance
models (player vehicles, mission-critical props) and unacceptable for world-fill props.

**Strategy C: Cross-fade hook.** Implement alpha cross-fading between LOD levels by hooking
the LOD switch in `CRenderer::BuildRenderList` — draw both models for 5–10 frames after the
switch, linearly fading the old model's alpha out and the new model's alpha in. This requires
two draw calls during the transition but completely eliminates visible pop. This is the approach
used by GTA V's LOD system and can be back-ported to SA via an ASI.

---

### Key takeaways

- SA's shader selection is a **pipeline function pointer per `RpAtomic`** — replacing that pointer
  is the clean per-material shader mod point (Depth 2).
- Three hook depths: **Present** for whole-frame post-processing; **pipeline dispatch** for
  per-material shader replacement; **vertex buffer modification** for per-vertex data changes.
- SA uses **fixed-function pixel processing** — any per-pixel effect (normal maps, PBR) requires
  an extra draw pass or a pipeline-level pixel shader attachment.
- Custom shader registers should be in **c12 and above** — SA's shaders own c0–c11 and read them
  as input; writing into c0–c11 corrupts the next draw call.
- LOD pop elimination options: **silhouette-preserving LOD authoring** (zero cost), **disable LOD**
  (doubles polygon count), or **alpha cross-fade hook** (two draw calls during transition, zero pop).

**Previous:** [C44.3 — Runtime assembly](03-runtime-assembly.md)
**Up:** [C44 — Shaders hub](C44-Shaders.md)
