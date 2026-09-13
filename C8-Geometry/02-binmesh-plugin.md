# C8.2 — BinMeshPLG: the Real Index Buffer

> **The one-sentence version:** every geometry carries a second, redundant copy of its topology —
> pre-split by material and pre-stripped — and that copy, not the struct's triangle array, is what the
> renderer draws.

[← C8.1 — The geometry struct](01-the-geometry-struct.md) · [Chapter 8 hub](C8-Geometry.md) ·
[Next: C8.3 — Identifying plugins by size →](03-identifying-plugins-by-size.md)

**Confidence:** ✅ Verified

---

## 1. The layout

Extension section `0x50E`, inside every geometry's `Extension` (`0x03`) child.

```c
struct BinMeshHeader {
    uint32_t flags;        // 0 = triangle list, 1 = tristrip
    uint32_t numSplits;    // one per material
    uint32_t totalIndices; // sum over all splits
};

struct BinMeshSplit {      // × numSplits
    uint32_t numIndices;
    uint32_t materialIndex;
    uint32_t indices[numIndices];
};
```

✅ *Verified.* The walk — header, then `numSplits` splits, each `8 + 4 × numIndices` bytes — landed
**exactly** on the section end for all **258 of 258** geometries, and the per-split index counts summed
to `totalIndices` in every case. Two independent consistency checks, both clean.

## 2. Why it exists

The geometry struct already contains triangles ([C8.1 §4](01-the-geometry-struct.md)). `BinMeshPLG`
contains the same topology again, transformed for the hardware:

| | Struct triangles | BinMeshPLG |
|---|---|---|
| Form | 8-byte records with a material id per triangle | flat `uint32` index runs |
| Grouping | interleaved, any material order | **grouped into one run per material** |
| Topology | triangle list | **tristrip** (255 of 258) |
| Purpose | authoring / editing | **the draw call** |

A renderer wants one contiguous index run per material so it can issue one draw per material with no
sorting. That is precisely what a split is. The struct's triangle array is the source-of-truth form; the
plugin is the baked form.

**Consequence for tools:** a tool that edits geometry must rebuild `BinMeshPLG`, not just the struct
triangles. Editing only the struct produces a file that parses perfectly and renders the *old* mesh —
another instance of the silent-failure pattern this encyclopedia keeps running into
([C6.2 §4](../C6-Collision/02-header-and-bounds.md),
[C7.1 §3](../C7-RenderWare-Stream/01-the-section-stream.md)).

## 3. What the sample shows

| Measurement | Value |
|---|---:|
| Geometries with a `BinMeshPLG` | **258 / 258 — all of them** |
| Total indices | 184,468 |
| `flags == 1` (tristrip) | 255 |
| `flags == 0` (triangle list) | 3 |

Split counts:

| Splits | Geometries |
|---:|---:|
| 1 | 64 |
| 2 | 56 |
| 3 | 29 |
| 4 | 22 |
| 5 | 17 |
| 6 | 14 |
| 7 | 13 |
| 8 | 8 |
| 9+ | 35 |

✅ *Verified.*

**One split means one material.** A quarter of world geometry is single-material — a wall, a road
segment, a simple prop. The long tail runs past nine, and those are the composite buildings.

The split count is therefore a cheap proxy for a model's material complexity, readable without
decoding the material list at all.

## 4. Indices are 32-bit

`uint32` per index, for meshes averaging 433 vertices ([C8.1 §6](01-the-geometry-struct.md)) — where 16
bits would comfortably suffice.

🟡 *Reasoned:* this is RenderWare's platform-neutral form rather than a considered choice for this
content. The D3D9 pipeline presumably narrows to 16-bit index buffers at upload; the file format keeps
the wider type because it must serve platforms and content where it is needed.

The cost is real: 184,468 indices × 4 bytes = 738 KB across 258 geometries, roughly half of which is
zero bytes. In a game with a 13.18 MiB streaming budget
([C2.3](../C2-CStreaming/03-memory-budget-and-stream-ini.md)) that is not nothing — though it is paid
in streamed bytes, not resident memory, since the buffer is transformed at load.

⏳ **Open:** whether the loader converts to 16-bit at upload. Answering it means reading the geometry
upload path, which was not attempted.

## 5. Tristrip and the three exceptions

255 of 258 are tristripped. The three triangle-list geometries are worth flagging as a reader
requirement rather than a curiosity: **a reader must honour the flag**, because interpreting a triangle
list as a strip produces a mesh that is geometrically wrong but structurally valid — it parses, it
renders, and it looks like corruption rather than a bug.

🟡 *Reasoned:* the three exceptions are likely geometry that stripped badly — meshes where the
stripifier found no useful runs and the exporter fell back. Confirming that would need the models
inspected individually.

---

### Key takeaways

- `BinMeshPLG` (`0x50E`) is present on **every** geometry and holds the **real index buffer**.
- Layout verified two ways: the walk lands exactly on the section end on **258/258**, and per-split
  counts sum to `totalIndices` every time.
- It is a **second copy** of the topology — material-split and tristripped — because the renderer wants
  one draw per material.
- **Editing the struct triangles without rebuilding this plugin renders the old mesh**, silently.
- **255 of 258 are tristrips**; the flag must be honoured or the mesh is wrong-but-valid.
- Split count is a free proxy for material complexity: **64 of 258 geometries are single-material**.
- Indices are **32-bit** — roughly 738 KB across the sample, about half of it zero bytes.

**Continue:** [C8.3 — Identifying plugins by size](03-identifying-plugins-by-size.md)
