# C44.2 — The register map and the transform

A vertex shader is only meaningful against the vertex format it reads. RenderWare's D3D9 pipelines fix that
format as a **semantic→register map** — position is always in register `v0`, bone weights always in `v1`, and
so on — and that map is embedded in the executable as the shader's declaration block. This page reads it out
and shows the two instructions that matter: the transform and the skinning.

## The vertex-input register map

At `0x8D7340` in `.data` sits the block of vertex-declaration strings the pipelines emit. Read together they
are the complete input layout of a San Andreas vertex shader:

| Register | Declaration | Semantic |
|---|---|---|
| `v0` | `dcl_position0 v0` | object-space position |
| `v1` | `dcl_blendweight0 v1` | skinning bone weights |
| `v2` | `dcl_blendindices0 v2` | skinning bone indices |
| `v3` | `dcl_normal0 v3` | vertex normal |
| `v4` | `dcl_color0 v4` | prelit / vertex colour |
| `v5` | `dcl_texcoord0 v5` | texture coordinates 0 |
| `v6` | `dcl_texcoord1 v6` | texture coordinates 1 |
| `v13` | `dcl_tangent0 v13` | tangent (for env/bump) |
| `v14` | `dcl_position1 v14` | second position (morph) |
| `v15` | `dcl_normal1 v15` | second normal (morph) |

`derive_shaders.py` asserts all ten declarations are present (`vertex_input_register_map`). This is the shader
equivalent of a struct layout: it is the fixed contract between the vertex buffer
([C7](../C7-RenderWare-Stream/C7-RenderWare-Stream.md)'s geometry) and the vertex program. Because it is
fixed, the shaders can be tiny — they always know where to find each input.

Two entries are worth calling out. `v14`/`v15` (`position1`/`normal1`) are the **morph-target** channel —
RenderWare's vertex morphing, used for things like the interpolated deformation of some models. `v13`
(`tangent0`) is what the env-map pipe ([C44.1](01-the-pipelines.md)) needs to compute a reflection direction.
Their presence in the register map is what tells you those features exist in the vertex path at all.

## The transform: `m4x4 oPos`

The one instruction every one of these shaders must contain is the projection of the object-space position
into clip space. It is embedded verbatim:

```
m4x4 oPos.xyzw, v0, c[0]
```

`m4x4` is the 4×4 matrix multiply: it transforms input position `v0` by the matrix in constant registers
`c[0]`–`c[3]` (the combined world-view-projection matrix the pipeline uploads) and writes the clip-space
result to the output position register `oPos`. That single line *is* the vertex transform — the thing a
fixed-function T&L unit would otherwise do — now expressed as shader code. Finding it as a literal string in
the executable is the proof that the transform is programmable here, not fixed-function.

## Skinning is a vertex-shader job

The skinned-atomic pipeline ([C44.1](01-the-pipelines.md)) proves peds are transformed on the GPU. Its
declaration block adds the two skinning inputs — `dcl_blendweight0 v1` and `dcl_blendindices0 v2` — and its
transform reads a *different* register:

```
m4x4 oPos.xyzw, r3, c[0]      ; r3 = the skinned position
```

Here the position handed to the projection is not `v0` but `r3` — a temporary the shader has already filled by
blending the vertex against its bone matrices, weighted by `v1` and indexed by `v2`. In other words: the
shader looks up each vertex's up-to-four bones (indices in `v2`), fetches their matrices from constant
registers, transforms the vertex by each, blends the results by the weights in `v1` into `r3`, and *then*
projects `r3`. That is hardware skeletal skinning, and it is why the skin pipeline is a separate `.csl` from
the rigid atomic one. It also connects to [C17](../C17-IFP-Animation/C17-IFP-Animation.md): the IFP animation
system poses the skeleton, the pipeline uploads those bone matrices as shader constants, and this shader
applies them per vertex.

`derive_shaders.py` asserts both the skinning declarations and the two `m4x4 oPos` forms
(`skinning_is_vertex_shader`, `position_transform_m4x4`).

## Key takeaways

- The vertex-input register map is fixed and embedded at `0x8D7340`: `v0` position, `v1`/`v2` skinning,
  `v3` normal, `v4` colour, `v5`/`v6` texcoords, `v13` tangent, `v14`/`v15` morph — the shader's input
  contract.
- Every shader contains `m4x4 oPos.xyzw, v0, c[0]` — the world-view-projection transform expressed as shader
  code, the programmable replacement for fixed-function T&L.
- Peds are **skinned in the vertex shader**: the skin pipeline blends bones by `v1`/`v2` into `r3` then
  projects `r3` — hardware skinning tied to [C17](../C17-IFP-Animation/C17-IFP-Animation.md)'s poses.

**Continue:** [C44.3 — Runtime assembly, and what is still fixed-function →](03-runtime-assembly.md)
