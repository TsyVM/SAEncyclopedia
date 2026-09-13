# C9.1 — The Material List

> **The one-sentence version:** a count-prefixed index array followed by 28-byte material records, each
> carrying a colour, a textured flag and three surface-response floats — verified exact on every one of
> 258 lists and 1,139 materials.

[← Chapter 9 hub](C9-Materials-And-Textures.md) · [Next: C9.2 — Texture references →](02-texture-references.md)

**Confidence:** ✅ Verified

---

## 1. The list

```c
// MaterialList (0x08)
//   Struct (0x01):
struct MaterialListStruct {
    uint32_t numMaterials;
    int32_t  indices[numMaterials];   // -1 = "the next inline Material section"
};
//   then numMaterials × Material (0x07)
```

✅ *Verified:* the struct's declared size is exactly `4 + 4 × numMaterials` on **258 of 258** material
lists.

The index array exists so a list can *reuse* an earlier material instead of repeating it — a
non-negative entry points back at an already-defined material, `-1` means "read the next inline one."
In the sampled world geometry the entries are uniformly `-1`; reuse is a capability the exporter did
not exercise for these models.

🟡 *Reasoned:* the `-1` convention is the documented RenderWare behaviour and is consistent with every
sample, but this pass did not observe a non-negative index in the wild to confirm the reuse path.

## 2. The material record

```c
// Material (0x07)
//   Struct (0x01):  exactly 28 bytes
struct MaterialStruct {
    uint32_t flags;         // +0x00
    uint32_t colour;        // +0x04  RGBA, one byte per channel
    uint32_t unused;        // +0x08
    int32_t  isTextured;    // +0x0C  0 or 1
    float    ambient;       // +0x10
    float    specular;      // +0x14
    float    diffuse;       // +0x18
};                          // = 0x1C
```

✅ *Verified:* **1,139 of 1,139** material structs are exactly 28 bytes. The three trailing floats
account for the final 12, which is what fixes them as three separate scalars rather than one vector or
a packed value.

### `isTextured`

| Value | Count |
|---:|---:|
| 1 | 1,124 (98.7 %) |
| 0 | 15 (1.3 %) |

When it is 1, a `Texture` (`0x06`) section follows inside the material
([C9.2](02-texture-references.md)). When it is 0, the material is flat-shaded and renders with
`colour` alone — 15 of 1,139 sampled surfaces, typically untextured trim.

A reader must branch on this field rather than probing for a `Texture` section, because the material's
child list is otherwise identical in both cases.

## 3. The three surface floats

Distribution across 1,139 materials:

| `(ambient, specular, diffuse)` | Count |
|---|---:|
| `(1.0, 0.0, 1.0)` | 890 (78.1 %) |
| `(0.4, 1.0, 1.0)` | 129 |
| `(0.5, 1.0, 1.0)` | 94 |
| `(0.3, 1.0, 1.0)` | 13 |

✅ *Verified.*

The default is `(1.0, 0.0, 1.0)` — full ambient response, **no specular**, full diffuse. That is
78.1 % of world geometry: matte surfaces lit by the ambient term, consistent with
[C8.1 §2](../C8-Geometry/01-the-geometry-struct.md)'s finding that 65.5 % of geometry is prelit with no
normals. **A surface with baked vertex colours and no normals cannot compute a specular highlight**, so
a zero specular coefficient is not a stylistic choice — it is the only coherent value.

The three non-default groups all share `specular = 1.0` and `diffuse = 1.0` and vary only in ambient
(0.3 / 0.4 / 0.5). Those 236 materials are the runtime-lit minority, and they line up with the
`0x00F3`-flagged geometry from C8.1 that carries normals.

🟡 *Reasoned:* the pairing between low-ambient/specular materials and normal-bearing geometry is a
strong correlation across two independent censuses, not a per-material join — this pass counted them
separately rather than matching material to geometry record by record.

## 4. Materials per geometry

1,139 materials across 258 geometries — **4.4 per geometry** on average, which agrees with the
`BinMeshPLG` split counts in [C8.2 §3](../C8-Geometry/02-binmesh-plugin.md) (64 single-split, long tail
past nine).

That agreement is a useful cross-check: the split count and the material count are recorded in
*different sections* by *different mechanisms*, and they describe the same quantity. If a reader's
material parsing disagreed with its `BinMeshPLG` parsing, one of them would be wrong.

## 5. Reading it

```python
def materials(buf, matlist_start, matlist_end):
    """Yield (colour, isTextured, ambient, specular, diffuse) per material."""
    st = next(s for s in sections(buf, matlist_start, matlist_end) if s[0] == 0x01)
    n  = struct.unpack_from('<I', buf, st[3])[0]
    # st[1] == 4 + 4*n   <- assert this; it is exact in retail data
    for typ, size, lib, payload in sections(buf, matlist_start, matlist_end):
        if typ != 0x07:
            continue
        s = next(x for x in sections(buf, payload, payload + size) if x[0] == 0x01)
        flags, colour, _unused, textured = struct.unpack_from('<IIiI', buf, s[3])
        amb, spec, diff = struct.unpack_from('<3f', buf, s[3] + 16)
        yield colour, bool(textured), amb, spec, diff
```

The `4 + 4*n` assertion is worth keeping in production code: it is the cheapest possible check that the
material list was located correctly, and it held on every retail sample.

---

### Key takeaways

- `MaterialList` struct is **`4 + 4 × numMaterials`** — exact on **258/258**.
- The index array supports material **reuse** (`-1` = inline); retail world geometry never uses it.
- The material record is **exactly 28 bytes** on **1,139/1,139**: flags, RGBA colour, unused,
  `isTextured`, and three surface floats.
- **98.7 % of materials are textured**; the other 15 render from `colour` alone, and a reader must
  branch on the field rather than probe for the section.
- The default surface response is **`(1.0, 0.0, 1.0)` — zero specular — on 78.1 %**, which is the only
  coherent value for prelit geometry with no normals.
- **4.4 materials per geometry**, agreeing with `BinMeshPLG`'s split counts recorded independently in a
  different section.

**Continue:** [C9.2 — Texture references](02-texture-references.md)
