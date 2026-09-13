# Chapter 8 — Geometry & the Binary Mesh

> **Goal of this chapter:** decode the mesh payload itself — the variable-layout geometry struct whose
> shape is dictated by a flags word, the plugin that carries the real index buffer, and a method for
> identifying unknown plugins without reading a line of code.

**Subsystem category:** Rendering / asset format
**Depends on:** [C7 — RenderWare: the Stream Format](../C7-RenderWare-Stream/C7-RenderWare-Stream.md)
**Ties:** [C1](../C1-Streaming/C1-Streaming.md), [C2](../C2-CStreaming/C2-CStreaming.md), [C5](../C5-CWorld/C5-CWorld.md), [C6](../C6-Collision/C6-Collision.md), [C7](../C7-RenderWare-Stream/C7-RenderWare-Stream.md), [C9](../C9-Materials-And-Textures/C9-Materials-And-Textures.md), [C10](../C10-2dEffect/C10-2dEffect.md)
**RE status:** Documented
**Confidence:** ✅ Verified

---

## Deep-dive pages

- [C8.1 — The geometry struct](01-the-geometry-struct.md): a payload whose layout is computed from
  flags, verified exactly on every sample.
- [C8.2 — BinMeshPLG: the real index buffer](02-binmesh-plugin.md): why the triangle array in the
  struct is not what the GPU draws.
- [C8.3 — Identifying plugins by size](03-identifying-plugins-by-size.md): closing two unknown section
  IDs with arithmetic instead of disassembly.
- [C8.4 — Modding geometry and plugins](04-modding-geometry-and-plugins.md): vertex format flags and
  round-trip correctness; BinMesh split→draw-call mapping; 65,535-vertex limit; plugin preservation
  table; normals and prelit vertex colors for world props.

---

## 8.1 The sample

Everything in this chapter was verified against **200 randomly sampled DFFs** from `gta3.img`,
yielding:

| Measurement | Value |
|---|---:|
| Geometry sections | **258** |
| Triangles | 76,558 |
| Vertices | 111,772 |
| **Payload walks that land exactly on the section end** | **258 / 258** |

✅ The last row is the chapter's foundation. A geometry section's layout is not fixed — it is *computed*
from a flags word, a vertex count, a triangle count and a morph-target count
([C8.1](01-the-geometry-struct.md)). If any of that were misread, the computed size would diverge from
the section's declared size. It diverged on **zero** of 258.

## 8.2 The layout is conditional

Unlike the collision header ([C6.2](../C6-Collision/02-header-and-bounds.md)) or the IMG directory
entry ([C1.1](../C1-Streaming/01-img-ver2-archive-model.md)), a geometry section has **no fixed
layout**. Four counts and eight flag bits determine which arrays are present and how long each is:

```
struct                     16 bytes            always
prelit colours             4 × numVertices     if PRELIT
texture coords             8 × numVertices × numUVSets
triangles                  8 × numTriangles
per morph target:
    bounding sphere        16 bytes
    hasVertices/hasNormals 8 bytes
    positions              12 × numVertices    if hasVertices
    normals                12 × numVertices    if hasNormals
```

A reader that guesses wrong does not fail — it reads the next section's bytes as vertex data. The
exact-walk check is what makes the reading verified rather than plausible.

## 8.3 Three things the sample settles

**Every geometry has exactly one morph target.** All 258. San Andreas ships no morph-animated world
geometry, so the loop that looks general is, in practice, always one iteration.

**Tristrips are the norm.** 255 of 258 binary meshes are flagged tristrip, 3 are triangle lists
([C8.2](02-binmesh-plugin.md)).

**The struct's triangle array is not the draw call.** Every geometry also carries a `BinMeshPLG`
extension holding the actual index buffer, split by material — 184,468 indices across the sample. The
struct's triangles are the authoring-side data; the plugin is what the renderer consumes.

## 8.4 Plugins, and a method

Five distinct extension IDs appear inside geometry sections:

| ID | Occurrences | Status |
|---|---:|---|
| `0x50E` BinMeshPLG | 258 / 258 | ✅ decoded ([C8.2](02-binmesh-plugin.md)) |
| `0x253F2FD` | 258 | ✅ structure closed ([C8.3](03-identifying-plugins-by-size.md)) |
| `0x253F2F9` | 188 | ✅ structure closed ([C8.3](03-identifying-plugins-by-size.md)) |
| `0x253F2F8` | 33 | ⏳ open |
| `0x116` | 3 | ⏳ open |

[C8.3](03-identifying-plugins-by-size.md) closes two of them by correlating payload size against the
geometry's own parameters — `0x253F2F9` is exactly `4 + 4 × numVertices` on all 188 occurrences, which
identifies its *shape* beyond doubt without touching the executable.

That generalises: **a plugin's payload size, regressed against the counts its parent already declares,
is often enough to determine its structure.** It is the same style of argument as the
`center == midpoint(min, max)` test in [C6.2 §2](../C6-Collision/02-header-and-bounds.md) — let the
data constrain the answer.

## 8.5 Dependencies

```
Geometry (0x0F)                              [C8]
  ├── inside GeometryList inside Clump       [C7.3]
  ├── references MaterialList                [C7.3, not yet decoded]
  ├── BinMeshPLG supplies the index buffer   [C8.2]
  └── consumed by the D3D9 world pipeline    [C0.3, C7.3]
```

## 8.6 Scope

⏳ **Still open:** the material list (texture references, colours, surface properties), the
`TextureNative` pixel payloads, `0x253F2F8`, `0x116`, and the `2dEffect` records from
[C7.3 §2](../C7-RenderWare-Stream/03-clumps-and-txd.md). Vertex *semantics* — coordinate handedness,
units, winding order — are also not established here; this chapter decodes sizes and offsets, not
conventions.

---

### Key takeaways

- Verified against **200 DFFs / 258 geometries**, with the computed payload size matching the declared
  section size on **258 of 258** — the check that validates the whole layout.
- Geometry has **no fixed layout**: four counts and eight flag bits decide which arrays exist. A wrong
  reading silently consumes the next section.
- **Every geometry has exactly one morph target**; **255 of 258** meshes are tristripped.
- **The struct's triangle array is not what is drawn** — `BinMeshPLG` carries the real, material-split
  index buffer.
- Two unknown plugin IDs were closed by **size correlation**, not disassembly.
- Materials, pixel data, two plugins and vertex conventions remain **open**.

**Next:** [C8.1 — The geometry struct](01-the-geometry-struct.md)

## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Loaded / consumed by:** [C1](../C1-Streaming/C1-Streaming.md), [C2](../C2-CStreaming/C2-CStreaming.md), [C5](../C5-CWorld/C5-CWorld.md), [C6](../C6-Collision/C6-Collision.md), [C7](../C7-RenderWare-Stream/C7-RenderWare-Stream.md), [C9](../C9-Materials-And-Textures/C9-Materials-And-Textures.md)
- **Known bugs / gotchas:** degenerate geometry can NaN the pipeline.
- **Modding:** geometry/material edits flow through here into the D3D9 pipelines (C44).
- **Performance:** vertex/index buffers instanced once; drawn many times.
