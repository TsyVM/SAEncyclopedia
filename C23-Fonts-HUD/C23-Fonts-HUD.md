# Chapter 23 — Fonts, the Glyph Atlas and the HUD

> **Goal of this chapter:** decode how San Andreas draws text — the `data/fonts.dat` width tables, the
> `models/fonts.txd` glyph atlas, and the character-to-atlas mapping baked into the executable — and pin
> each claim down by arithmetic that closes. The single strongest result is a **triple confirmation**: the
> per-font record is `208 + 1 + 1 = 210` bytes, and the stride `0xD2` appears *twice* in `gta_sa.exe`
> (once where the parser writes the table, once where the renderer reads it), with the `26 × 8` loop that
> fills it visible in the parser itself.

**Subsystem category:** UI / rendering
**Depends on:** [C9 — Materials and Textures](../C9-Materials-And-Textures/C9-Materials-And-Textures.md)
(the atlas and HUD sprites are RenderWare `TextureNative` sections) ·
[C19 — GXT text](../C19-GXT-Text/C19-GXT-Text.md) (the character codes the atlas indexes are the ones GXT
stores)
**Ties:** [C9](../C9-Materials-And-Textures/C9-Materials-And-Textures.md), [C19](../C19-GXT-Text/C19-GXT-Text.md), [C20](../C20-Audio/C20-Audio.md), [C21](../C21-Particles/C21-Particles.md), [C22](../C22-Map-Zones/C22-Map-Zones.md)
**RE status:** Verified
**Confidence:** ✅ Verified (the fonts.dat grammar, the 210-byte record stride, the width lookup, the
16 × 16 UV grid, the atlas census, and — from decoding the atlas — the grid and the `index = ASCII − 0x20`
character mapping) / ⏳ (the two absent mask atlases; the HUD sprite geometry; the exact purpose of the
secondary alphabet in rows 13–15)

---

## Deep-dive pages

- [C23.1 — `fonts.dat` and the 210-byte record](01-fonts-dat-and-the-record.md): a plain-text width table
  whose `208 + 1 + 1` layout is confirmed three ways — the `26 × 8` fill loop in the parser, and the
  `imul …, 0xD2` stride in both the parser and the renderer.
- [C23.2 — The glyph atlas and its UV grid](02-the-glyph-atlas.md): two 512 × 512 DXT3 textures, a
  16 × 16 grid of 32-pixel cells, and the `and 0xF` / `shr 4` split plus the `1/16` UV scale that the
  executable uses to place each glyph.
- [C23.3 — The HUD texture dictionary](03-the-hud-txd.md): 69 individually-named sprites in `hud.txd` —
  radar blips, weapon crosshairs, the map disc — a *named-sprite* dictionary rather than a grid, and the
  two font mask atlases the engine asks for but the retail tree never ships.

---

## 23.1 The result first

| Claim | Evidence |
|---|---|
| `fonts.dat` is **plain text** | zero structural bytes; `#` comments and bracketed section tags ✅ |
| It declares **2 fonts** | `[TOTAL_FONTS] 2`, and **2** `[FONT_ID]` blocks are found ✅ |
| Each font is **208 proportional widths** | `26 × 8` — the parser's outer loop is `mov edi, 0x1A`, inner `mov esi, 8` ✅ |
| The in-memory record is **210 bytes** | `208` widths + `1` replacement-space char + `1` unproportional width ✅ |
| The stride is **`0xD2`** | `imul …, 0xD2` at the parser's PROP store **and** at the renderer's width lookup ✅ |
| Widths are stored as **bytes** | the fill loop copies 8 `int`s down to 8 bytes; values observed `5 … 37` ✅ |
| Width lookup is `PROP[fontId × 210 + glyph]` | `movzx eax, [ecx + edx + 0xC718B0]` at VA `0x719700` ✅ |
| `UNPROP` is the **monospace fallback** | the same function reads `[… + 0xC71981]` when the proportional flag is clear ✅ |
| The atlas is **2 × 512² DXT3** | `fonts.txd` holds `font1` and `font2`, both 512 × 512 DXT3 ✅ |
| Glyphs tile a **16 × 16 grid** | `col = glyph & 0x0F`, `row = glyph >> 4`, UV scaled by `1/16` at VA `0x718B30` ✅ |
| A glyph cell is **32 × 32 px** | `512 / 16 = 32` exactly; decoding the atlas confirms the grid ✅ |
| Character mapping | **glyph index = ASCII − 0x20** for the printable block, read off the decoded atlas ✅ |
| The atlas exceeds the width table | all 16 rows are populated (~256 glyphs); the width table covers only the first **208** ✅ |
| `hud.txd` is **69 named sprites** | header count `69` equals the sections walked; radar blips + crosshairs ✅ |
| Two **mask atlases are requested, not shipped** | the exe names `font1m` / `font2m`; neither is in `fonts.txd` ✅ |

## 23.2 How the text system fits together

Three files cooperate to put a glyph on screen, and each is pinned to the others by the executable:

```
              file                    what it provides            proved against the exe
fonts.dat     data/fonts.dat          per-glyph advance widths     parser 0x7187C0, renderer 0x7196F4
glyph atlas   models/fonts.txd        the pixels for each glyph     UV grid 0x718B30
char remap    (compiled in)           code -> atlas glyph index     0x718770 / 0x7192C0
hud.txd       models/hud.txd          the non-text HUD sprites      loaded via 0x5BA69F
```

The width table and the atlas are two views of the same 208-glyph alphabet. `fonts.dat` says *how far to
advance the pen* after each glyph; the atlas says *what to draw*; and the executable's character-remap
turns a raw character code (as stored by [C19](../C19-GXT-Text/C19-GXT-Text.md)'s GXT) into the index that
selects both. Because the index feeds the width table (`glyph` in `PROP[fontId × 210 + glyph]`) *and* the
atlas cell (`col = glyph & 0x0F`, `row = glyph >> 4`), the two tables cannot disagree about which glyph is
which — the same integer addresses both.

## 23.3 Why the `0xD2` stride is the load-bearing result

The record width is confirmed the way the project trusts most — the same constant showing up in two
independent places that would both break if it were wrong. The parser at `0x7187C0` computes a font's
storage base as `fontId × 0xD2 + 0xC718B0` before writing 208 width bytes into it; the renderer at
`0x7196F4` computes `fontId × 0xD2 + 0xC718B0` before reading one back. `0xD2` is `210`, and `210` is
`208 + 1 + 1` — the 208 proportional widths plus the single `REPLACEMENT_SPACE_CHAR` byte (stored at
`+208`) plus the single `UNPROP` byte (stored at `+209`). The count `208` is not assumed either: it is the
product of the parser's own two loop bounds, `mov edi, 0x1A` (26 rows) and `mov esi, 8` (8 per row). Every
number in `208 + 1 + 1 = 210` is read from the machine code, not fitted to the file.

## 23.4 What this chapter does not claim

✅ The **character mapping is now read off the decoded atlas** (see
[C23.2 §3](02-the-glyph-atlas.md)): the printable block is `glyph index = ASCII − 0x20`, and rows 6+ hold
accented-Latin glyphs and a secondary alphabet. Decoding also **corrected** an earlier arithmetic guess —
the atlas populates all 16 rows (~256 glyphs), not the 208 the width table covers.

⏳ The **exact purpose of the secondary alphabet** in rows 13–15 (the ~48 glyphs with no proportional
width) is not pinned down — they are placed by fixed-width or special-purpose HUD code rather than the
proportional text path, but which code and for what display is not traced here.

⏳ **Two mask atlases are missing.** The loader asks the texture dictionary for `font1m` and `font2m`
alongside `font1` and `font2`, but `fonts.txd` ships only the latter two. This is the same shape as
[C19](../C19-GXT-Text/C19-GXT-Text.md)'s five named-but-two-shipped languages,
[C21](../C21-Particles/C21-Particles.md)'s three unused particle types and
[C22](../C22-Map-Zones/C22-Map-Zones.md)'s cut `NAVIG.ZON`: a capability compiled into the engine with no
data behind it. Whether the mask atlases ever existed, or what the `m` suffix would have carried (a
separate alpha/outline channel is the obvious reading), is not derivable from the shipped files.

⏳ **The HUD sprite geometry** is not decoded here. `hud.txd` is a flat dictionary of named sprites, not a
grid; placing each one on screen is a per-call concern in the HUD code rather than a table, and is left for
a HUD-drawing chapter.

---

### Key takeaways

- ✅ `fonts.dat` is a **plain-text** width table declaring **2 fonts**, each **208** proportional widths in
  a `26 × 8` block, confirmed by the parser's own loop bounds.
- ✅ The in-memory record is **210 bytes** = `208 + 1 (replacement space) + 1 (unproportional)`, and the
  stride **`0xD2`** appears in **both** the parser (write) and the renderer (read) — the project's
  strongest form of proof.
- ✅ The glyph atlas is **two 512 × 512 DXT3** textures; glyphs tile a **16 × 16 grid of 32-pixel cells**,
  with `col = glyph & 0x0F`, `row = glyph >> 4` and a `1/16` UV scale read from VA `0x718B30`.
- ✅ Width lookup is `PROP[fontId × 210 + glyph]`, with `UNPROP` as the monospace fallback when the
  proportional flag is clear.
- ✅ `hud.txd` is **69 named sprites** (radar blips, crosshairs, the map disc) — a named-sprite dictionary,
  not an atlas grid.
- ✅ **Decoding the DXT3 atlases** confirms the grid visually and gives the character mapping
  (`index = ASCII − 0x20`); it also **corrected** a wrong "rows 13–15 unused" inference — all 16 rows are
  populated, while the width table covers only the first 208 glyphs.
- ⏳ The engine requests **two mask atlases** (`font1m`, `font2m`) that the retail tree never ships — a
  capability without data.
- ⏳ The exact purpose of the secondary alphabet (rows 13–15) and the HUD sprite placement are left open.

**Continue:** [C23.1 — `fonts.dat` and the 210-byte record](01-fonts-dat-and-the-record.md)

## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Loaded / consumed by:** [C9](../C9-Materials-And-Textures/C9-Materials-And-Textures.md), [C19](../C19-GXT-Text/C19-GXT-Text.md), [C20](../C20-Audio/C20-Audio.md), [C21](../C21-Particles/C21-Particles.md), [C22](../C22-Map-Zones/C22-Map-Zones.md)
- **Known bugs / gotchas:** hud.txd count read at wrong offset gives 21 not 69 (parse trap); mask atlases requested-not-shipped.
- **Modding:** fonts.dat + fonts.txd/hud.txd are HUD-mod assets; the 210-byte stride is the contract.
- **Performance:** glyph atlas sampled per drawn char.
