# C9.3 — TextureNative: the Pixel Payload

> **The one-sentence version:** a 76-byte header carrying two 32-character names, a raster format, a
> D3D format and the dimensions — censused over **every texture in the game**, showing San Andreas is
> **96.1 % DXT** with **78.2 % shipping no mipmaps**, and closing the `0x8000` flag with a perfect
> correlation.

[← C9.2 — Texture references](02-texture-references.md) · [Chapter 9 hub](C9-Materials-And-Textures.md)

**Confidence:** ✅ Verified (header, full-population census) / ⏳ (per-level offsets)

---

## 1. Population, not sample

Every figure on this page is a **complete census**, not an extrapolation:

| Scope | Value |
|---|---:|
| TXD files parsed | **3,974 — all of them** |
| `TextureNative` sections decoded | **32,157** |
| Archives covered | `gta3`, `gta_int`, `player`, `cutscene` |

This matters. An earlier draft of this page reported figures from a 60-TXD sample and they did not
reproduce when re-run with a different draw — sampled statistics are only meaningful alongside the
sampling procedure, and it is cheaper here to just read all of them. Percentages below are exact over
the whole game.

It also corrects [C7.3 §5](../C7-RenderWare-Stream/03-clumps-and-txd.md), which extrapolated "roughly
35,000 textures" from a small sample. The true figure is **32,157** — the extrapolation was about 10 %
high, which is exactly the error bar a 🟡 estimate deserved.

## 2. The header

```c
// TextureNative (0x15)
//   Struct (0x01):
struct TextureNativeStruct {
    uint32_t platform;         // +0x00  9 = D3D9  (C7.3 §4)
    uint16_t filterFlags;      // +0x04
    uint16_t addressingUV;     // +0x06
    char     name[32];         // +0x08
    char     maskName[32];     // +0x28
    uint32_t rasterFormat;     // +0x48
    uint32_t d3dFormat;        // +0x4C  FourCC or D3DFORMAT enum
    uint16_t width;            // +0x50
    uint16_t height;           // +0x52
    uint8_t  depth;            // +0x54
    uint8_t  numLevels;        // +0x55
    uint8_t  rasterType;       // +0x56
    uint8_t  compressionFlags; // +0x57
    // then per level: uint32 size, followed by size bytes
};
```

✅ *Verified* — the header parsed cleanly on all 32,157 sections.

The names are **32 bytes each**, against COL's 22 ([C6.2 §1](../C6-Collision/02-header-and-bounds.md)).
Different subsystem, different convention: a shared fixed-width-name helper across formats is a
liability.

## 3. Compression — 96.1 % DXT

| `d3dFormat` | Count | Share |
|---|---:|---:|
| **`DXT1`** | 28,807 | 89.6 % |
| **`DXT3`** | 2,098 | 6.5 % |
| `0x16` — `X8R8G8B8` | 1,015 | 3.2 % |
| `0x15` — `A8R8G8B8` | 237 | 0.7 % |
| **DXT total** | **30,905** | **96.1 %** |

✅ *Verified.*

The DXT1/DXT3 split is the alpha story. DXT1 carries at most 1-bit alpha; DXT3 carries 4-bit explicit
alpha. So the 2,098 DXT3 textures are the ones needing *graded* transparency — glass, fences, foliage
edges — and everything opaque or hard-masked is DXT1.

### The depth field cross-checks it exactly

| `depth` | Count |
|---:|---:|
| 16 | **30,905** |
| 32 | **1,252** |

`30,905` is precisely the DXT total. `1,252 = 1,015 + 237` is precisely the uncompressed total. Two
independent header fields agreeing to the unit across 32,157 records is what validates both readings.

## 4. ✅ The `0x8000` raster-format bit — closed

An earlier draft left this open. The full census settles it:

| `rasterFormat` | Count |
|---|---:|
| `0x0200` | 21,491 |
| `0x8200` | 7,008 |
| `0x0300` | 2,098 |
| `0x0600` | 1,015 |
| `0x0100` | 308 |
| `0x0500` | 237 |

Correlating the `0x8000` bit against `numLevels > 1`:

| `0x8000` set | has mips | Count |
|---|---|---:|
| no | no | **25,149** |
| yes | yes | **7,008** |
| no | yes | **0** |
| yes | no | **0** |

✅ **The correlation is perfect in both directions across all 32,157 textures.** `0x8000` is the
**mipmap-present flag**. Not one exception.

Note also `0x0300` = 2,098 = exactly the DXT3 count, and `0x0600` = 1,015 = exactly the `X8R8G8B8`
count. The raster format and the D3D format are not independent fields; the low bits encode the same
information the `d3dFormat` does.

## 5. Mip chains — most textures have none

| `numLevels` | Count |
|---:|---:|
| **1** | **25,149 (78.2 %)** |
| 9 | 5,415 |
| 8 | 799 |
| 10 | 679 |
| 7 | 104 |
| 6 | 9 |
| 11 | 2 |

✅ *Verified.* **78.2 % of San Andreas textures ship with a single mip level.**

A 9-level chain implies a 256×256 base, 10 implies 512×512, 11 implies 1024×1024 — and exactly 2
textures in the game are 1024². The level counts and the dimension census (§6) agree.

**Consequence for texture replacement:** adding a chain to one of the 25,149 single-level textures
increases its size by ~33 % and its streaming cost with it. Dropping a chain from one of the 7,008 will
shimmer at distance. `numLevels` must be preserved, and the `0x8000` bit must be kept consistent with
it — §4 shows the shipped data never disagrees, so the loader may well rely on that.

## 6. Dimensions

| Size | Count |
|---|---:|
| 128 × 128 | 10,642 |
| 256 × 256 | 7,988 |
| 64 × 64 | 5,191 |
| 32 × 32 | 2,191 |
| 128 × 256 | 1,156 |
| 128 × 64 | 951 |
| 256 × 128 | 930 |
| 64 × 128 | 809 |

Power-of-two throughout; 128² and 256² are 58 % of the game between them. Both orientations of every
non-square size are present in similar numbers — the pipeline did not normalise aspect, so artists
authored whatever the surface needed.

## 7. What is not decoded

⏳ **Open:** the **per-level payload walk**. Each mip level is prefixed by a `uint32` size followed by
that many bytes; palettised formats (`0x0100`/`0x0500`, 545 textures) carry a palette first. This page
decodes the header and censuses every field over the whole population; it does not walk the level data.

That is bounded and tractable, and would be verified exactly as
[C8.1 §3](../C8-Geometry/01-the-geometry-struct.md) was: sum header plus every level, assert it lands
on the section end, over all 32,157 sections.

---

### Key takeaways

- **Full-population census: 32,157 textures across all 3,974 TXDs** — no sampling, fully reproducible.
- This corrects [C7.3 §5](../C7-RenderWare-Stream/03-clumps-and-txd.md)'s extrapolated "~35,000" by
  about 10 %.
- **96.1 % DXT** — 28,807 DXT1, 2,098 DXT3. DXT3 is exactly the graded-alpha content.
- The **`depth` field cross-checks the format exactly**: 30,905 sixteen-bit = the DXT total;
  1,252 thirty-two-bit = the uncompressed total.
- ✅ **`0x8000` is the mipmap-present flag** — perfect correlation with `numLevels > 1`, zero exceptions
  in 32,157 records. Previously open, now closed.
- **78.2 % ship a single mip level**; exactly 2 textures in the game are 1024².
- Replacement must preserve `numLevels` **and** the `0x8000` bit — the shipped data never disagrees.
- ⏳ Per-level payload walk remains open; palettised formats account for 545 textures.

**Continue:** [Chapter 9 hub](C9-Materials-And-Textures.md) · next chapter: `C10 — 2dEffect & the Map's Lighting`
