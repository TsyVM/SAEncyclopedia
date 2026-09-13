# Chapter 22 — Zones, the Radar Grid and the Map Gridref

> **Goal of this chapter:** decode how San Andreas divides its 6,000-unit world — 378 named zones, six
> island regions, a 144-tile radar grid and a 100-cell development gridref — and pin each one down by
> arithmetic that closes. The strongest single result is a **cross-subsystem check**: every one of the
> **169** zone text keys hashes onto a key stored in [Chapter 19](../C19-GXT-Text/C19-GXT-Text.md)'s
> `american.gxt`. **169 / 169**, no misses.

**Subsystem category:** World / map
**Depends on:** [C11 — IDE and IPL](../C11-IDE-And-IPL/C11-IDE-And-IPL.md) (the zone files are loaded
through the IPL path) · [C19 — GXT text](../C19-GXT-Text/C19-GXT-Text.md) (zone labels are GXT keys) ·
[C1.1 — The IMG VER2 archive](../C1-Streaming/01-img-ver2-archive-model.md) (radar tiles live in
`gta3.img`)
**Ties:** [C1](../C1-Streaming/C1-Streaming.md), [C9](../C9-Materials-And-Textures/C9-Materials-And-Textures.md), [C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md), [C18](../C18-SCM-Script/C18-SCM-Script.md), [C19](../C19-GXT-Text/C19-GXT-Text.md), [C20](../C20-Audio/C20-Audio.md)
**RE status:** Verified
**Confidence:** ✅ Verified (both zone grammars, the GXT resolution, the radar grid arithmetic and **tile
orientation**, the gridref index arithmetic) / ⏳ (the type and island enumerations)

---

## Deep-dive pages

- [C22.1 — The zone tables](01-the-zone-tables.md): a ten-field record whose field order is confirmed by
  the `scanf` format compiled into the executable, 378 info zones inside a 6,000-unit world square, and
  the **169 / 169** GXT cross-check.
- [C22.2 — The radar grid](02-the-radar-grid.md): 144 tiles in `gta3.img`, numbered without a gap, tiling
  the world at exactly 500 units — the executable's loop bound of `0x90` that says so independently, and
  the world-to-tile conversion that fixes **tile 0 as the north-west corner**.
- [C22.3 — `gridref.dat` and the build tree](03-gridref-and-the-build-tree.md): a complete 10 × 10 grid
  mapping map cells to the Rockstar artists who owned them, shipped in retail and **still parsed by the
  engine** — into a table that, it turns out, **nothing ever reads** (the reader API is dead code).
- [C22.4 — Zone runtime and gameplay integration](04-zone-runtime-and-gameplay-integration.md): per-frame
  zone lookup callers; GXT zone-name connection; gang territory zones; radar grid O(1) tile lookup;
  interior zone ID conventions.

---

## 22.1 The result first

| Claim | Evidence |
|---|---|
| The zone record is **10 fields** | `378 / 378` and `6 / 6` records parse; the exe's format string has ten conversions ✅ |
| Field order `name type x1 y1 z1 x2 y2 z2 island textKey` | `"%s %d %f %f %f %f %f %f %d %s"` at VA `0x868D04` ✅ |
| The world is a **6,000-unit square** | `map.zon` spans exactly `−3000 … +3000` on **both** X and Y ✅ |
| Every zone box is well-formed | `min < max` on all three axes in **384 / 384** records ✅ |
| Zone labels are **GXT keys** | **169 / 169** distinct `info.zon` text keys hash onto stored keys ✅ |
| The radar grid is **12 × 12 = 144** | 144 `radarNN.txd` in `gta3.img`, numbered 0–143 with no gap; exe loop bound `0x90` ✅ |
| A radar tile covers **500 units** | `6000 / 12 = 500` exactly ✅ |
| **Tile 0 is the north-west corner** | world→tile at VA `0x5858D0`: `col=(X+3000)/500`, `row=11−(Y+3000)/500`, index `col+row×12` ✅ |
| `gridref.dat` is a complete **10 × 10** grid | all 100 cells `A1`–`J10` present; **600** units per cell ✅ |
| A gridref record is **32 bytes** | `shl ecx, 5` in the loader at `0x71D53A`; 100 cells map to `0 … 3168` step 32 with no collision ✅ |

## 22.2 Three grids over one world

San Andreas overlays three independent subdivisions on the same 6,000 × 6,000 world square, and each
exists for a different consumer:

```
                     cell size     cells      consumer
map.zon      6 regions      —          6      which island you are on (LA / SF / Vegas)
info.zon   378 zones        —        378      the place name shown on screen; navigation
radar grid   12 × 12      500.0      144      the minimap texture streamed per tile
gridref.dat  10 × 10      600.0      100      which artist owned that patch of map (build only)
```

The three that the *engine* uses do not share a cell size, and there is no reason they should — the radar
grid is a texture-streaming concern, the zones are a gameplay and UI concern, and they were sized
independently. What they do share is the world square, and that is the number the chapter pins down first:
`map.zon`'s six regions between them span exactly `−3000` to `+3000` on X and Y. Every other arithmetic
result in the chapter divides that span.

## 22.3 Why the GXT check matters

[C19](../C19-GXT-Text/C19-GXT-Text.md) derived the GXT key hash — CRC-32 with the final complement
omitted, on the uppercased key — and confirmed it by hashing the text keys named in
[C18](../C18-SCM-Script/C18-SCM-Script.md)'s mission scripts: 1,180 of 1,186 resolved, and the six
exceptions were listed.

The zone tables provide a second, entirely independent population of keys. They were authored by different
people for a different purpose, they live in a different file format, and they reach the text system by a
different code path. Hashing them is therefore a real test rather than a restatement — and it comes back
**169 / 169, with no exceptions to name**. That is a cleaner result than C19's own, and it confirms three
things at once: the hash function, the GXT container walk, and the identity of the zone record's tenth
field.

The one text key that does *not* resolve is `map.zon`'s, and that is the correct outcome: all six of its
records carry the literal `UNUSED`, which is a sentinel rather than a label. It hashes to nothing because
nothing is meant to be displayed for an island region. Recorded as a confirmation, not a miss.

## 22.4 What this chapter does not claim

⏳ The **`type` field** is `0` on all 378 info zones and `3` on all six map zones. Two values across two
files is not an enumeration one can decode; the field's position and width are certain and its meaning is
not derived.

⏳ The **`island` field** is `1` on every info zone, and on the map zones takes `1`, `2` and `3` — and
those group exactly as their names do (`LA01`/`LA02` → 1, `SF01`–`SF03` → 2, `Vegas` → 3). 🟡 That the
field identifies the three cities is a supported reading, since the correspondence with the names is
perfect and the names are themselves data in the file. It is not promoted to ✅ because three samples
cannot establish an enumeration.

✅ **Radar tile orientation is now closed** — see [C22.2 §4](02-the-radar-grid.md). The world-to-tile
conversion at VA `0x5858D0` gives `col = (X + 3000) / 500` and `row = 11 − (Y + 3000) / 500` with index
`col + row × 12`, so **tile 0 is the north-west corner** and numbering runs west→east then north→south.
Settling it meant reading the arithmetic at the draw site, not the load site — the same move that solved
[C20.5](../C20-Audio/05-eventvol.md).

---

### Key takeaways

- ✅ Zones are a **ten-field comma-separated record** in a `zone` … `end` section, and the field order is
  confirmed by the executable's own `scanf` format at VA `0x868D04`.
- ✅ The world is a **6,000-unit square**, established by `map.zon` spanning `−3000 … +3000` exactly on
  both axes; every other grid in the chapter divides it.
- ✅ **169 / 169** zone text keys resolve against C19's GXT — an independent population of keys confirming
  the hash, the container walk and the field's identity in one test.
- ✅ The radar grid is **12 × 12 = 144 tiles at 500 units**, confirmed both by counting `gta3.img` and by
  the executable's loop bound of `0x90`; **tile 0 is the north-west corner** (world→tile at VA `0x5858D0`).
- ✅ `gridref.dat` is a complete **10 × 10** grid of **32-byte** records, the arithmetic recovered from a
  `shl ecx, 5` in its loader — which runs at init, though **nothing ever reads the table it fills** (the
  four accessor functions are unreferenced dead code; the table is write-only in retail).
- 🧭 Three grids, three cell sizes, one world square — sized independently because their consumers are
  independent.
- ⏳ The `type` and `island` enumerations are left open; the radar tile orientation is now closed
  (tile 0 = NW).

**Continue:** [C22.1 — The zone tables](01-the-zone-tables.md)

## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Loaded / consumed by:** [C1](../C1-Streaming/C1-Streaming.md), [C9](../C9-Materials-And-Textures/C9-Materials-And-Textures.md), [C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md), [C18](../C18-SCM-Script/C18-SCM-Script.md), [C19](../C19-GXT-Text/C19-GXT-Text.md), [C20](../C20-Audio/C20-Audio.md)
- **Known bugs / gotchas:** dead gridref reader; radar tile 0=NW must not be assumed; NAVIG.ZON cut.
- **Modding:** info.zon/map.zon define zones; radar tiles in gta3.img are map-mod assets.
- **Performance:** zone lookup per position; radar tiled 12x12.
