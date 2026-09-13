# C23.3 — The HUD texture dictionary

> **The one-sentence version:** `models/hud.txd` is not an atlas — it is 69 individually-named RenderWare
> sprites, 62 of them 16 × 16 radar blips, and the executable loads it right beside `fonts.txd` through the
> same font/HUD setup path.

[← C23.2 — The glyph atlas](02-the-glyph-atlas.md) · [Chapter 23 hub](C23-Fonts-HUD.md)

**Confidence:** ✅ Verified (the dictionary census via C9's TextureNative header; the load-site pairing
with `fonts.txd`) / ⏳ (per-sprite screen geometry)

---

## 1. A named-sprite dictionary, not a grid

Where `fonts.txd` is a grid the engine indexes by arithmetic, `hud.txd` is the opposite design: a flat
dictionary of **69** textures, each fetched **by name** when the HUD needs it. Read with the same
`TextureNative` header from [C9.3](../C9-Materials-And-Textures/03-texturenative-payload.md), the header's
declared count (69) equals the number of sections walked, and the sizes break down as:

| Size | Format | Count | What they are |
|---|---|---:|---|
| 16 × 16 | DXT1 | 58 | radar blips (opaque icons) |
| 16 × 16 | DXT3 | 4 | radar blips needing graded alpha |
| 64 × 64 | DXT3 | 4 | `radardisc`, `radarRingPlane`, crosshairs |
| 128 × 128 | DXT3 | 1 | `skipicon` |
| 32 × 32 | DXT1 | 1 | `siterocket` (a crosshair) |
| 32 × 32 | DXT3 | 1 | one blip |

**62 of the 69** are `radar_*` blips — the minimap icons for mission markers, shops, pickups, and
characters (`radar_ammu`, `radar_barbers`, `radar_bulldozer`, `radar_WOOZIE`, …). The remaining seven are
the non-blip HUD sprites: `radardisc` and `radarRingPlane` (the minimap frame and its altitude ring),
`siteM16` and `siterocket` (weapon crosshairs), `fist` and `arrow` (melee / direction icons), and
`skipicon` (the "skip cutscene" prompt).

The DXT1/DXT3 split follows C9.3's rule exactly: the 58 opaque blips are DXT1 (one-bit alpha is enough for
a hard-edged icon), and the sprites needing graded transparency — the disc, the ring, the crosshairs, and
four soft-edged blips — are DXT3. Nothing here departs from the game-wide texture conventions.

## 2. It loads beside the fonts

`hud.txd` belongs in this chapter because the engine treats it as part of the same UI-texture setup. The
string block at `0x86A620` holds both dictionary paths back to back —

```
MODELS\FONTS.TXD   fonts   MODELS\HUD.TXD   ps2btns
```

— and the load site at `0x5BA69F` (which references `MODELS\FONTS.TXD` and the dictionary name `fonts`)
sits in the same routine that brings the HUD textures in. A third name in the block, `ps2btns`, is the
console button-glyph set; on the PC build it is a name without a corresponding texture in these
dictionaries, another small capability-without-data of the kind catalogued in
[C23.2 §4](02-the-glyph-atlas.md#4-two-mask-atlases-the-engine-asks-for-and-never-gets).

## 3. Why the geometry stops here

The one thing `hud.txd` does *not* provide is where each sprite is drawn. Because it is a named dictionary
rather than a grid, there is no cell arithmetic to recover — each blip's screen position is computed per
frame from a world coordinate projected onto the minimap, and each fixed sprite (disc, crosshair, skip
icon) is positioned by a literal in the HUD code. That is a drawing-side concern, not a table, and it
belongs to a HUD-rendering chapter rather than this format-focused one. What this page establishes is the
**inventory** — 69 named sprites, their formats and sizes, and their pairing with the font atlas at load
time — proven by the same census header as every other texture in the game.

---

### Key takeaways

- ✅ `hud.txd` is **69 named sprites**, not an atlas grid; the header count matches the sections walked.
- ✅ **62** are `radar_*` minimap blips (58 DXT1 + 4 DXT3 at 16 × 16); the other **7** are the disc, ring,
  two crosshairs, `fist`, `arrow` and `skipicon`.
- ✅ The DXT1/DXT3 split matches C9.3's game-wide rule — opaque icons DXT1, graded-alpha sprites DXT3.
- ✅ It is loaded beside `fonts.txd` in the same UI-texture routine (`0x5BA69F`); the block's third name
  `ps2btns` is a console button set with no PC texture behind it.
- ⏳ Per-sprite screen geometry is a HUD-drawing concern and is left for a future chapter.

**Continue:** [Chapter 23 hub](C23-Fonts-HUD.md) · next chapter: a HUD-drawing or fonts-glyph-semantics
follow-up (see the hub's open items)
