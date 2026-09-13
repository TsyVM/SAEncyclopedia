# C44.3 — Runtime assembly, and what is still fixed-function

The reason "extract the shaders" has no answer is that San Andreas's shaders are not stored — they are
**built each time from templates**. This page shows the runtime-assembly mechanism, explains why no HLSL
ships, and draws the honest line between what is programmable (vertex) and what stays fixed-function (pixel).

## The shaders are generated, not stored

The tell is a `printf`-style format specifier inside the shader source:

```
dcl_texcoord%u v%u
```

A stored shader would read `dcl_texcoord0 v5`; a *generated* one reads `dcl_texcoord%u v%u`, because the
pipeline fills in the numbers at runtime depending on how many texture coordinate sets the geometry actually
has. `derive_shaders.py` asserts this template is present (`runtime_shader_source_template`). So the
`AllInOne` pipeline, when it first sees a given vertex format, walks the format and **emits** a vertex-shader
assembly string — the right `dcl_*` lines for the inputs that exist, the `m4x4 oPos` transform
([C44.2](02-register-map-and-transform.md)), the skinning block if there are bone weights, the env-map lines
if it is a reflective material — then assembles and caches it.

## D3DX9 does the assembling

The tool that turns that string into a shader is compiled into the executable: the strings
`D3DX9 Shader Assembler` and `D3DX9 Shader Compiler`, the `D3DX` version markers, and
`Direct3DShaderValidatorCreate9` are all present, and so are the shader-model targets `vs.1.0`, `vs.1.1`,
`vs.2.0`. `derive_shaders.py` asserts the assembler and the models (`d3dx_assembler_and_models`). At load
time the pipeline calls the D3DX assembler on its generated source and gets back a compiled vertex shader;
the many `ps_1_1`/`ps_1_4`/`ps_2_0` version strings elsewhere in the binary are that same D3DX library's
internal tables, not shipped pixel shaders. The shader model in play is **vs 1.1** on older hardware and
**vs 2.0** where available — modest by later standards, but programmable.

This is why the disc has no `shaders` folder, no `.fx`, no `.hlsl`, and no `.vso`/`.pso` bytecode: the shader
for any object is a function of its vertex format and material, produced on demand. It is the same shape of
finding as [C43](../C43-Front-End-Menu/C43-Front-End-Menu.md)'s hardcoded menu — the thing you would look for
as an asset is instead compiled into the program's behaviour — but here the asset does not even exist as a
fixed blob in the exe; it is a *recipe*.

## What stays fixed-function

Programmable does not mean everything is a shader. The line falls between the two halves of the pipeline:

- **Vertex side — programmable.** Transform, skinning, morphing, env-map coordinate generation: all in the
  vertex shader ([C44.2](02-register-map-and-transform.md)).
- **Pixel side — largely fixed-function.** RenderWare 3.x on D3D9 does most of its per-pixel work with the
  fixed-function **texture stage states** and render states — the same `RwRenderStateSet` calls
  [C40](../C40-Render-Pipeline/C40-Render-Pipeline.md) saw each pass issue. Base texture, alpha blend, the
  material colour modulate, fog: these are stage-state configuration, not a pixel shader. Only specific
  effects (parts of the env-map composite) reach for pixel-shader instructions.

So San Andreas sits in the middle of the era's spectrum: a **programmable vertex pipeline over a
fixed-function pixel pipeline**. Calling it "fixed-function" (as [C40.3](../C40-Render-Pipeline/03-where-the-frame-fits.md)
did) undersells the vertex work; calling it a "shader engine" would oversell the pixel work. The accurate
description is the one this chapter proves.

## What this closes, and what's left

**Closes** the shader question and corrects [C40.3](../C40-Render-Pipeline/03-where-the-frame-fits.md): the
vertex pipeline is programmable RenderWare D3D9, the shaders are runtime-assembled from templates, and no HLSL
ships because none exists as an asset.

**Opens:**

- ⏳ The full generated shader **source** for one pipeline (reconstruct the complete `nodeD3D9AtomicAllInOne`
  vertex shader by following the template-emission code, not just the individual lines).
- ⏳ The **env-map** pixel work — whether the reflection composite uses a pixel shader or texture stages.
- ⏳ The **constant-register** map (which `c[n]` holds the WVP matrix, the bone matrices, the light and fog
  parameters).
- ⏳ The lighting model in the vertex shader (`D3D9lights.c` is linked) — per-vertex directional + ambient.

## Key takeaways

- The shaders are **generated at runtime** from templates (`dcl_texcoord%u v%u`) per vertex format, then
  assembled by the linked **D3DX9** assembler — so no shader ships as an asset or even a fixed blob.
- The shader models are **vs 1.1 / vs 2.0**; the many `ps_*` strings are D3DX's internal tables, not shipped
  pixel shaders.
- San Andreas is a **programmable vertex pipeline over a fixed-function pixel pipeline** — the honest middle
  ground, and the correction to [C40.3](../C40-Render-Pipeline/03-where-the-frame-fits.md).

**Continue:** [back to the C44 hub →](C44-Shaders.md) · or the rendering arc:
[C40 Render Pipeline](../C40-Render-Pipeline/C40-Render-Pipeline.md) · [C7 RenderWare Stream](../C7-RenderWare-Stream/C7-RenderWare-Stream.md).
