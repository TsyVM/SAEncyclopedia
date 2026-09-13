# C8.1 — The Geometry Struct

> **The one-sentence version:** sixteen bytes of counts followed by a payload whose entire layout is
> computed from them — verified by the only check that can verify a conditional format, which is that
> the computed size lands exactly on the declared one, 258 times out of 258.

[← Chapter 8 hub](C8-Geometry.md) · [Next: C8.2 — BinMeshPLG →](02-binmesh-plugin.md)

**Confidence:** ✅ Verified

---

## 1. The header

```c
struct RpGeometryStruct {     // 16 bytes, at the head of the Struct (0x01) child
    uint16_t flags;           // +0x00  format bits — see §2
    uint8_t  numUVSets;       // +0x02
    uint8_t  nativeFlags;     // +0x03
    uint32_t numTriangles;    // +0x04
    uint32_t numVertices;     // +0x08
    uint32_t numMorphTargets; // +0x0C
};
```

✅ *Verified.* The first dword is read as a packed word-plus-two-bytes: the low 16 bits are the format
flags, byte 2 is the UV-set count, byte 3 is platform-native flags.

## 2. The flags

| Bit | Name | Effect on layout |
|---|---|---|
| `0x0001` | TRISTRIP | authoring hint; the real topology is in `BinMeshPLG` ([C8.2](02-binmesh-plugin.md)) |
| `0x0002` | POSITIONS | — |
| `0x0004` | TEXTURED | one UV set |
| `0x0008` | PRELIT | **adds** `4 × numVertices` colour bytes |
| `0x0010` | NORMALS | — (normals are per morph target) |
| `0x0020` | LIGHT | — |
| `0x0040` | MODULATEMATERIALCOLOR | — |
| `0x0080` | TEXTURED2 | two UV sets |

Census over the 258 sampled geometries:

| Flags | Count | Decoded |
|---|---:|---|
| `0x002F` | 169 | TRISTRIP, POSITIONS, TEXTURED, PRELIT, LIGHT |
| `0x00F3` | 35 | TRISTRIP, POSITIONS, NORMALS, LIGHT, MODULATE, TEXTURED2 |
| `0x006F` | 16 | + MODULATE |
| `0x0037` | 14 | TRISTRIP, POSITIONS, TEXTURED, NORMALS, LIGHT |
| `0x00B3` | 7 | TRISTRIP, POSITIONS, NORMALS, LIGHT, TEXTURED2 |
| `0x004F` | 5 | TRISTRIP, POSITIONS, TEXTURED, PRELIT, MODULATE |
| `0x0077` | 5 | + NORMALS, MODULATE |
| `0x000F` | 4 | no LIGHT |
| `0x0036` | 3 | **no TRISTRIP** |

✅ *Verified.*

Two patterns worth naming. **`0x002F` covers 65.5 % of world geometry** (169 of 258) — prelit,
textured, no normals.
Static map geometry carries baked vertex colours instead of normals, because it is never dynamically
lit. And the second most common, `0x00F3`, is the inverse: normals and two UV sets, no prelit — the
signature of geometry that *is* lit at runtime.

**`numUVSets` is authoritative over the flags.** Observed: 216 geometries with 1 set, 42 with 2. A
reader should use the byte and fall back to deriving from `TEXTURED`/`TEXTURED2` only when it is zero.

## 3. The payload

Everything after the 16-byte header is conditional:

```
offset = structStart + 16

if flags & PRELIT:
    offset += 4 * numVertices              # BGRA per vertex

offset += 8 * numVertices * numUVSets      # two floats per vertex per set

offset += 8 * numTriangles                 # see §4

for m in range(numMorphTargets):
    offset += 16                           # bounding sphere: centre[3], radius
    hasVertices, hasNormals = read two u32
    offset += 8
    if hasVertices: offset += 12 * numVertices
    if hasNormals:  offset += 12 * numVertices

assert offset == structStart + structSize   # <- the verification
```

✅ **That assertion held on all 258 geometries.** Not approximately — exactly.

This is the only honest way to verify a conditional format. Each individual field could be
misinterpreted in a way that happens to parse; the probability that a wrong layout reproduces the
declared size across 258 sections with vertex counts from tens to thousands is negligible.

## 4. The triangle record is 8 bytes

```c
struct RpTriangle {
    uint16_t vertex2;
    uint16_t vertex1;
    uint16_t materialId;
    uint16_t vertex3;
};
```

🟡 *Reasoned:* the **size** is ✅ verified at 8 bytes by the walk. The **field order** — with the second
vertex first and the material index sandwiched in the middle — is the documented RenderWare ordering
and is consistent with the data, but this pass did not independently prove the ordering. It is flagged
because it is exactly the kind of detail that parses either way
([C6.2 §2](../C6-Collision/02-header-and-bounds.md) is the cautionary precedent).

Note also that these triangles are **not what gets drawn** — see [C8.2](02-binmesh-plugin.md).

## 5. Morph targets: always one

All 258 geometries report `numMorphTargets == 1`.

The morph-target block is where **positions and normals actually live** — not in the struct header.
That is the RenderWare model: a geometry is a topology plus N sets of vertex positions, and animation
by morphing swaps between them. San Andreas uses none of it for world geometry.

🟡 *Reasoned:* the loop is therefore effectively dead code in practice for this data set, but a reader
must still implement it, because the size arithmetic depends on it and a hand-authored or modded file
could legitimately carry more.

Each target's bounding sphere is `centre[3], radius` — 16 bytes, at the *morph target* level rather
than the geometry level, which is why a geometry with multiple targets has multiple bounding spheres.

## 6. Sizing

Across the sample: 111,772 vertices and 76,558 triangles in 258 geometries — averaging **433 vertices
and 297 triangles** per geometry.

For a typical `0x002F` geometry (prelit, one UV set, one morph target with positions and normals) the
payload works out to `4 + 8 + 12 + 12 = 36` bytes per vertex plus 8 per triangle. At the sample average
that is roughly 18 KB per geometry — consistent with the 4–32 KB IMG slots observed in
[C7.1 §3](../C7-RenderWare-Stream/01-the-section-stream.md).

---

### Key takeaways

- **16-byte header**: packed flags/UV-count/native byte, then triangle, vertex and morph-target counts.
- The payload layout is **entirely conditional** on those values — there is no fixed offset past byte 16.
- Verified by the **exact-size assertion on 258 of 258** geometries; a conditional format cannot be
  verified any other way.
- **`0x002F` is 65.5 % of world geometry** (169/258) — prelit, textured, no normals: baked lighting for
  static geometry. The second-commonest flag set is its runtime-lit inverse.
- **`numUVSets` is authoritative**; derive from flags only if it is zero.
- Triangle records are **8 bytes** (✅); the field *order* is 🟡 conventional, not independently proved.
- **All 258 geometries have exactly one morph target**, and positions/normals live inside it, not in the
  header.

**Continue:** [C8.2 — BinMeshPLG: the real index buffer](02-binmesh-plugin.md)
