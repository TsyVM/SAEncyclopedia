# C8.4 — Modding geometry and plugins

## Why the geometry struct matters for mod tools

C8.1–C8.3 decoded `RpGeometry`'s struct, the `BinMesh` plugin, and the plugin-size identification method. For modding, the geometry struct is the layer that determines whether a tool can round-trip a model without losing data.

## Vertex format and what it costs

`RpGeometry` stores vertex data in parallel arrays (matching C47's parallel-array design philosophy):
- `positions[]` — required; XYZ floats per vertex
- `normals[]` — present if `rpGEOMETRYNORMALS` flag set
- `prelit_colors[]` — present if `rpGEOMETRYPRELIT` flag set (vertex colors)
- `texcoords[N][]` — N sets of UV coords; N controlled by `numTexCoords`

The `flags` field in the geometry header tells tools exactly which arrays are present. A tool that reads the flags correctly will produce output that round-trips perfectly; one that assumes all arrays are always present will write zeros into absent slots, changing the model's appearance.

**Prelit vertex colors** are important for map objects: many SA world props use vertex colors for baked ambient occlusion and lighting. Stripping them produces a model that looks flat and overlit. The `rpGEOMETRYPRELIT` flag must be preserved.

## The BinMesh plugin (C8.2) and draw call structure

The `BinMesh` plugin (type `0x50E` = 1294) stores the triangle list as an array of `MaterialSplit` structures. Each split has:
- A material index (into the geometry's material list)
- A triangle count
- The actual index data

This one-split-per-material arrangement maps directly to one D3D9 `DrawIndexedPrimitive` call per split. The render pipeline (C40) walks the BinMesh splits in order, setting the material state (texture + alpha mode) before each draw. The total number of draw calls for a model = the number of BinMesh splits.

**Modding implication:** merging materials (combining multiple materials into one atlas) reduces draw call count and improves performance. The tradeoff is UV overlap/waste in the atlas. This is why many optimization-focused mod replacements use a single-material "baked" approach.

## Triangle soup limit

The maximum safe index count per `BinMesh` split is bounded by the D3D9 index buffer the streaming system allocates (C1). High-polygon replacements that exceed this limit cause silent truncation or a crash in the geometry instancing step. The community has observed practical limits around 32,768 vertices per geometry — the D3D9 16-bit index limit when the geometry is instanced as an indexed vertex buffer.

A geometry that exceeds 65,535 vertices must be split into multiple geometries (multiple atomics in the DFF), each referencing its own `RpGeometry`. This is why high-poly vehicle mods use multiple body parts rather than a single mesh.

## Plugin payloads and their preservation requirement

C8.3 established the plugin identification method (by size). For a mod tool's purposes, the critical plugins to preserve are:

| Plugin | Type ID | If stripped |
|---|---|---|
| `BinMesh` | `0x50E` | Model cannot be rendered (no index data) |
| `skin` | `0x116` | Skinned models lose bone weighting (deform to root) |
| `hanim` | `0x11E` | Hierarchical animation breaks (no bone controller) |
| `matfx` | `0x120` | Reflections, dual-UV effects disabled |
| `2dfx` | `0x100` | Lights and particle emitters on the model disappear |

The safe default for any plugin not listed above is to preserve it verbatim. The plugin ID is guaranteed unique (Criterion assigned IDs), so an unknown plugin can be safely forwarded without parsing.

## Normals and their effect on lighting

SA's vertex shader pipelines (C44) use normals for diffuse lighting calculations. A geometry with the `rpGEOMETRYNORMALS` flag set will be lit by the directional light; one without the flag is rendered as a fully-lit (unlit) object, which makes it appear to emit light uniformly.

Many SA world props deliberately omit normals to save memory (they're static, pre-lit by the vertex color bake). Vehicle and ped geometries always carry normals — the dynamic lighting response is part of what makes them look like separate objects from the world.

**Recalculating normals** after editing vertex positions is required if smooth lighting is desired. Tools that export from DCC (Blender, 3ds Max) and import back into the DFF must recompute normals in the correct space (object-local, not world-space).

## The texture coordinate connection

SA geometry uses up to two UV sets (`numTexCoords = 1` or `2`). A second UV set is used by the `matfx` plugin for reflection/spec mapping. If the material has a `matfx` entry but the geometry only has one UV set, the reflection map samples from the same UV as the diffuse — a common source of incorrect reflections in third-party vehicle mods.

**Previous:** [C8.3 — Identifying plugins by size](03-identifying-plugins-by-size.md)  
**Up:** [C8 — Geometry](C8-Geometry.md)
