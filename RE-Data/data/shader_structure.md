# Shaders: RenderWare D3D9 Vertex-Shader Pipelines
*Source: `shader_structure.json`*
- **$schema:** shader_structure.v1
- **Generated:** tools/derive_shaders.py

## Finding
GTA:SA 1.0 PC is NOT fixed-function: it renders through RenderWare (RW36) D3D9 vertex-shader pipelines whose VS assembly is generated at runtime from embedded source templates and assembled with D3DX9. No shipped HLSL; pixel processing is largely fixed-function texture-stage-state.

**Corrects:** C40.3's 'no per-object shader system to extract'

## Vertex-input register map
| Register | Semantic |
| --- | --- |
| v0 | position0 |
| v1 | blendweight0 |
| v2 | blendindices0 |
| v3 | normal0 |
| v4 | color0 |
| v5 | texcoord0 |
| v6 | texcoord1 |
| v13 | tangent0 |
| v14 | position1 |
| v15 | normal1 |

## Pipelines
| Pipeline | Role |
| --- | --- |
| nodeD3D9WorldSectorAllInOne | map / world sector geometry |
| nodeD3D9AtomicAllInOne | atomics: objects & vehicles |
| nodeD3D9SkinAtomicAllInOne | skinned atomics: peds (blendweight/indices) |
| CustomEnvMapPipe | San Andreas env-map: vehicle reflections |
| matfx effectPipesD3D9 | material effects |
| im3dpipe | immediate mode: sky (C38) / HUD (C23) / coronas |

## Checks
| Check | Result |
| --- | --- |
| vertex_input_register_map | PASS |
| skinning_is_vertex_shader | PASS |
| position_transform_m4x4 | PASS |
| runtime_shader_source_template | PASS |
| rw_d3d9_pipelines_linked | PASS |
| csl_shader_sources | PASS |
| custom_envmap_pipeline | PASS |
| d3dx_assembler_and_models | PASS |
