# Chapter 44 — Shaders: The RenderWare D3D9 Vertex-Shader Pipelines

> **Goal of this chapter:** answer the "shaders" question honestly, and correct an overstatement.
> [C40.3](../C40-Render-Pipeline/03-where-the-frame-fits.md) claimed San Andreas has "no per-object shader
> system to extract." That is **wrong**, and this chapter says so with the bytes: GTA:SA 1.0 PC renders the
> world, vehicles and peds through **RenderWare (RW36) Direct3D 9 vertex-shader pipelines**, whose
> vertex-shader assembly is **generated at runtime** from embedded source templates and assembled with
> **D3DX9**. There is no shipped HLSL and pixel processing is largely fixed-function, but vertex processing is
> genuinely programmable. This chapter catalogues the pipelines, reads the fixed vertex-input register map out
> of the executable, and shows the position-transform and skinning instructions that prove it.

**Subsystem category:** Rendering — the D3D9 shader pipelines under RenderWare
**Depends on:** [C7](../C7-RenderWare-Stream/C7-RenderWare-Stream.md) (the RenderWare clumps/atomics these
draw), [C40](../C40-Render-Pipeline/C40-Render-Pipeline.md) (the render pass that submits them) ·
corroborated by the RenderWare `$Id` source stamps in the executable
**Ties:** [C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md) (vehicles, whose reflections use the env-map
pipe), [C17](../C17-IFP-Animation/C17-IFP-Animation.md) (skeletal animation the skin pipeline transforms),
[C38](../C38-Skybox-And-Clouds/C38-Skybox-And-Clouds.md)/[C23](../C23-Fonts-HUD/C23-Fonts-HUD.md) (the
im3d path)
**RE status:** Documented
**Confidence:** ✅ for the pipelines being linked, the runtime-assembly mechanism, the register map, and the
transform/skinning instructions (`derive_shaders.py`, 8/8) · this is a **catalogue + correction**, not a
struct-tiling chapter — the proof standard is presence of the embedded shader source, stated as such
**Data artifact:** [`RE-Data/data/shader_structure.json`](../RE-Data/data/shader_structure.json) — generated
by [`tools/derive_shaders.py`](../tools/derive_shaders.py)

---

## Deep-dive pages

- [C44.1 — The vertex-shader pipelines](01-the-pipelines.md): the RW36 D3D9 `AllInOne` pipelines
  (world-sector, atomic, skin-atomic), the San Andreas `CustomEnvMapPipe`, and the `im3d` path — and the
  correction of [C40.3](../C40-Render-Pipeline/03-where-the-frame-fits.md).
- [C44.2 — The register map and the transform](02-register-map-and-transform.md): the fixed vertex-input
  semantic→register table (`v0` = position … `v15` = normal1), the `m4x4 oPos` world-view-projection
  transform, and the `blendweight`/`blendindices` that make skinning a vertex-shader job.
- [C44.3 — Runtime assembly, and what is still fixed-function](03-runtime-assembly.md): how the pipelines
  *build* shader source from templates (`dcl_texcoord%u v%u`) and assemble it with D3DX9 each time, why there
  is no HLSL to extract, and what pixel-side work stays fixed-function.
- [C44.4 — Custom pipeline and visual modding](04-custom-pipeline-and-visual-modding.md): **three hook depths** — Depth 1 (Present vtable slot 17, full D3D9 device access, post-frame effects), Depth 2 (pipeline dispatch via `RpAtomicSetPipeline` / `RwD3D9SubmitNode` hook, per-object vertex replacement), Depth 3 (vertex buffer modification for pre-transform injection); the **register map constraint table** (c0–c11 read-only engine constants, c12+ free for custom shaders); custom pixel shader re-draw pattern (second submit after Present with custom `IDirect3DPixelShader9`); LOD pop elimination strategies (draw-dist inflation, fade-crossover, alpha dither); visual mod capability table by hook depth; the ENB/graphics mod injection point.

---

## 44.0 The result first

| Claim | Value | Tier | Evidence |
|---|---|:--:|---|
| GTA:SA 1.0 PC uses **vertex shaders** | yes | ✅ | RW D3D9 VS pipeline `$Id`s + embedded VS source |
| Renderer | **RenderWare RW36 D3D9** | ✅ | `//RenderWare/RW36Active/…/d3d9/…` stamps |
| Vertex-shader pipelines | world / atomic / skin / env-map / im3d | ✅ | `nodeD3D9*AllInOne`, `CustomEnvMapPipe`, `im3dpipe` |
| Shader source is **runtime-assembled** | via D3DX9 | ✅ | `dcl_texcoord%u v%u` template + `D3DX9 Shader Assembler` |
| Vertex-input register map | `v0`=pos … `v15`=normal1 (10) | ✅ | `dcl_*` declarations @`0x8D7340` |
| Skinning is a **vertex-shader** feature | blendweight/indices | ✅ | `dcl_blendweight0 v1`, `dcl_blendindices0 v2` |
| Position transform | `m4x4 oPos.xyzw, v0, c[0]` | ✅ | plain + skinned (`r3`) forms present |
| Shipped HLSL files | **none** | ✅ (negative) | no `.fx`/`.hlsl`; source is RW assembly, built in code |
| Automated checks | **8 / 8** | — | `tools/derive_shaders.py` refuses to write otherwise |

## 44.1 The correction, stated plainly

The rendering arc reached a wrong conclusion and this chapter fixes it in the open, per the house rule that
corrections are written up, not silently patched. [C40.3](../C40-Render-Pipeline/03-where-the-frame-fits.md)
said San Andreas is "immediate-mode fixed-function RenderWare … no per-object shader system to extract." Two
things were conflated. The **im3d path** — the sky ([C38](../C38-Skybox-And-Clouds/C38-Skybox-And-Clouds.md)),
the HUD ([C23](../C23-Fonts-HUD/C23-Fonts-HUD.md)), coronas — *is* the simple immediate-mode path, and that
part was right. But the **world/atomic/skin path** — every building, vehicle and ped — goes through
RenderWare's D3D9 `AllInOne` pipelines, which run **vertex shaders**. The evidence was in the executable the
whole time: RenderWare's `d3d9vertexshader.c` is linked, the vertex-shader source declarations
(`dcl_position0 v0`, `m4x4 oPos…`) are embedded, and the D3DX9 assembler is compiled in to turn them into
shaders. So the accurate statement is: **vertex processing is programmable (VS 1.1/2.0), pixel processing is
largely fixed-function, and no HLSL ships because the shaders are RenderWare assembly built at runtime.**

## 44.2 What "a shader" is in San Andreas

There is no `shaders/` folder and no `.fx` file because San Andreas's shaders are not assets — they are
**generated in code**. Each RW D3D9 pipeline (world sector, atomic, skinned atomic, env-map) builds a short
vertex-shader assembly string from templates that depend on the geometry's vertex format (how many texture
coordinates, whether it has bone weights, whether it needs a tangent), then hands that string to
`D3DXAssembleShader` and caches the compiled vertex shader. The register map is fixed — position is always
`v0`, bone weights always `v1`, the first texcoord always `v5` — so the shaders are small, uniform, and
composed rather than authored. That is why the "extract the shaders" instinct has no target: the shader for a
given object does not exist on disk or even in the exe as bytecode; it exists as a *recipe* (the templates
and the register map this chapter reads) that the pipeline cooks per vertex format at load time.

---

## Key takeaways

- GTA:SA 1.0 PC is **not** fixed-function: the world/vehicle/ped pipelines are RenderWare **D3D9
  vertex-shader** pipelines — this **corrects** [C40.3](../C40-Render-Pipeline/03-where-the-frame-fits.md).
- The shaders are **RenderWare assembly generated at runtime** from templates and assembled with D3DX9, which
  is why no HLSL ships — there is nothing to extract, only a recipe (templates + register map).
- Vertex processing is programmable (transform, skinning, env-map); pixel processing stays largely
  fixed-function — the honest middle ground between "fixed-function" and "modern shader engine."

**Continue:** [C44.1 — The vertex-shader pipelines →](01-the-pipelines.md)

## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Loaded / consumed by:** [C7](../C7-RenderWare-Stream/C7-RenderWare-Stream.md), [C17](../C17-IFP-Animation/C17-IFP-Animation.md), [C23](../C23-Fonts-HUD/C23-Fonts-HUD.md), [C38](../C38-Skybox-And-Clouds/C38-Skybox-And-Clouds.md), [C40](../C40-Render-Pipeline/C40-Render-Pipeline.md), [C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md)
- **Known bugs / gotchas:** the shaders are runtime-assembled (no HLSL to extract); corrected C40.3's 'no shaders'.
- **Modding:** the RW D3D9 pipelines (AllInOne/EnvMap) are where ENB/graphics mods hook.
- **Performance:** vertex shaders assembled once per vertex format, cached.
