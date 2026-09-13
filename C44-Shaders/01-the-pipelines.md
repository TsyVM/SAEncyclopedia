# C44.1 — The vertex-shader pipelines

San Andreas draws through RenderWare's "pipeline 2" (P2) architecture, and on Windows the concrete pipelines
are Direct3D 9 nodes. This page catalogues the ones the executable links, separates the vertex-shader
pipelines from the immediate-mode path, and pins the RenderWare version.

## The RenderWare version

Every RenderWare source file leaves an `$Id` stamp in the binary, and they all agree:

```
//RenderWare/RW36Active/rwsdk/world/pipe/p2/d3d9/d3d9vertexshader.c
//RenderWare/RW36Active/rwsdk/world/pipe/p2/d3d9/nodeD3D9AtomicAllInOne.c
//RenderWare/RW36Active/rwsdk/world/pipe/p2/d3d9/nodeD3D9WorldSectorAllInOne.c
//RenderWare/RW36Active/rwsdk/driver/d3d9/d3d9device.c
```

So the renderer is **RenderWare 3.6 (RW36), Direct3D 9**. The presence of `d3d9vertexshader.c` alone settles
the headline: a fixed-function-only build would not link a vertex-shader translation unit.
`derive_shaders.py` asserts the six pipeline ids are present (`rw_d3d9_pipelines_linked`).

## The pipelines

| Pipeline (`$Id` / string) | What it draws | Shaders? |
|---|---|---|
| `nodeD3D9WorldSectorAllInOne` | the map — world-sector geometry | **vertex shader** |
| `nodeD3D9AtomicAllInOne` | atomics: objects and vehicles | **vertex shader** |
| `nodeD3D9SkinAtomicAllInOne` (`skind3d9pipesshared.c`) | skinned atomics: **peds** | **vertex shader** (+ bone matrices) |
| `CustomEnvMapPipe` (`effectPipesD3D9.c`, matfx) | environment mapping: **vehicle reflections** | **vertex shader** |
| `im3dpipe.c` (`baim3d.c`) | immediate mode: sky, HUD, coronas | fixed-function-ish |

The first four are the programmable path — every building, prop, car and character is transformed by a vertex
shader. The last, **im3d**, is the immediate-mode path this encyclopedia already met: the
[C38](../C38-Skybox-And-Clouds/C38-Skybox-And-Clouds.md) sky gradient and the
[C23](../C23-Fonts-HUD/C23-Fonts-HUD.md) HUD sprites are submitted through `RwIm3D*`, which does not build a
per-object shader. That split is the key to the [C40.3](../C40-Render-Pipeline/03-where-the-frame-fits.md)
correction: im3d is simple, but it is a minority of the frame.

## "AllInOne" — why one pipeline per object class

RenderWare's P2 pipelines are graphs of nodes, but San Andreas uses the **`AllInOne`** variants, which
collapse the whole transform-and-render sequence into a single node per object class. There is one for world
sectors, one for generic atomics, and one for skinned atomics — three because the vertex work differs: a
world sector is static geometry, a generic atomic has a single model matrix, and a skinned atomic needs an
array of bone matrices and per-vertex blend weights ([C44.2](02-register-map-and-transform.md)). Each
`AllInOne` node owns a `.csl` shader-source library:

```
nodeD3D9WorldSectorAllInOne.csl
nodeD3D9AtomicAllInOne.csl
nodeD3D9SkinAtomicAllInOne.csl
```

`derive_shaders.py` confirms all three `.csl` names are present (`csl_shader_sources`). A `.csl` is not a
compiled shader — it is the **source template library** the pipeline draws from to build its vertex shader
([C44.3](03-runtime-assembly.md)).

## The San Andreas addition: the env-map pipe

Most of the pipeline stack is stock RenderWare, but `CustomEnvMapPipe` (with its
`CustomEnvMapPipeAtmDataPool` / `CustomEnvMapPipeMatDataPool` allocators) is San Andreas's own — the
environment-mapping pipeline that gives vehicles their reflective sheen
([C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md)'s cars). It layers a reflection lookup on top of the
atomic pipeline, and it is the clearest example of the game reaching past stock RW into custom shader work.
`derive_shaders.py` asserts it is present (`custom_envmap_pipeline`).

## Key takeaways

- The renderer is **RenderWare RW36 Direct3D 9**, and it links `d3d9vertexshader.c` — a vertex-shader build,
  not fixed-function-only.
- Four pipelines are programmable — world-sector, atomic, skin-atomic and the San Andreas `CustomEnvMapPipe`
  — and one, **im3d**, is the simple immediate-mode path ([C38](../C38-Skybox-And-Clouds/C38-Skybox-And-Clouds.md)
  sky, [C23](../C23-Fonts-HUD/C23-Fonts-HUD.md) HUD).
- Each `AllInOne` pipeline owns a `.csl` source-template library, and there are three because static, rigid
  and skinned geometry need different vertex work.

**Continue:** [C44.2 — The register map and the transform →](02-register-map-and-transform.md)
