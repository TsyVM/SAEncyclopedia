# C23.2 — The glyph atlas and its UV grid

> **The one-sentence version:** the glyphs live in two 512 × 512 DXT3 textures, laid out on a 16 × 16 grid
> of 32-pixel cells, and the executable addresses a cell by splitting the glyph index into
> `col = index & 0x0F`, `row = index >> 4` and scaling each by `1/16`.

[← C23.1 — The record](01-fonts-dat-and-the-record.md) · [Chapter 23 hub](C23-Fonts-HUD.md) · next:
[C23.3 — The HUD dictionary](03-the-hud-txd.md)

**Confidence:** ✅ Verified (the atlas census via C9's TextureNative header; the 16 × 16 UV grid and cell
size via the executable; the grid and character mapping confirmed by decoding the DXT3 textures) / ⏳ (the
two absent mask atlases)

---

## 1. Two textures, censused with C9's header

`models/fonts.txd` is a RenderWare texture dictionary, read with the exact `TextureNative` header
[C9.3](../C9-Materials-And-Textures/03-texturenative-payload.md) censused over the whole game. Its
dictionary struct declares **2 textures**, and both parse cleanly:

| Texture | Size | `d3dFormat` | Levels | Raster format |
|---|---|---|---:|---|
| `font2` | 512 × 512 | `DXT3` | 1 | `0x0300` |
| `font1` | 512 × 512 | `DXT3` | 1 | `0x0300` |

Both are `DXT3`, which C9.3 identified as exactly the graded-alpha format — the right choice for anti-
aliased glyph edges that need more than DXT1's one-bit alpha. Both ship a single mip level, and the
`0x0300` raster format is the DXT3 code C9.3 saw on all 2,098 DXT3 textures in the game. Nothing here is a
special case; the font atlas is an ordinary pair of textures read by the ordinary header.

Note the naming: the *textures* are `font1` and `font2`, while `fonts.dat`'s *fonts* are id `0` and `1`.
They are two different numbering schemes for the same two typefaces, and the chapter keeps them distinct
rather than assuming an off-by-one.

## 2. A glyph is a cell on a 16 × 16 grid

The atlas is 512 pixels square and holds the 208-glyph alphabet. The grid geometry is not inferred from
`512 / 208`; it is read from the draw code at **VA `0x718B30`**, which turns a glyph index into a texture
cell:

```asm
00718B35  shr  cl, 4               ; row = glyphIndex >> 4
00718B3A  movzx edx, cl
00718B3D  and  al, 0xF             ; col = glyphIndex & 0x0F
00718B3F  movzx ecx, al
...
00718B4E  fild [esp+0xC]           ; col -> float
00718B56  fild [esp+0xC]           ; row -> float
00718B5A  fmul [0x858620]          ; * (1/16)   <- 0x858620 = 0.0625
...
00718BD7  fadd [0x858620]          ; + (1/16)   -> far edge of the cell
```

The glyph index splits into a low nibble (column, `0 … 15`) and the high bits (row), and each is scaled by
`1/16` to give a texture coordinate; the opposite corner is one `1/16` step further. That is a **16-column
by 16-row grid** with each cell exactly `1/16` of the texture on a side:

```
16 columns × 16 rows  =  256 cells
512 px / 16           =  32 px per cell
glyph index  ->  (index & 0x0F, index >> 4)
```

So each glyph occupies a **32 × 32-pixel** cell.

## 3. Decoding the atlas confirms the grid — and corrects a wrong inference

`tools/decode_font_atlas.py` decodes both DXT3 textures to RGBA (a from-scratch DXT3 decoder over C9.3's
`TextureNative` header) and overlays the 16 × 16 / 32-pixel grid the executable's math implies. The grid
lines land exactly on the glyph cells in both atlases — a **visual confirmation** of the geometry derived
in §2. The renders are saved beside this page as `atlas_font1.png` and `atlas_font2.png`.

Reading the cells against the grid also pins the **character mapping**. The printable-ASCII block occupies
the first six rows with

```
glyph index = ASCII codepoint − 0x20
```

so index 0 is the space (`0x20`), index 1 is `!`, row 1 is `0 … 9 : ; < = > ?`, index 33 is `A`
(`0x41 − 0x20`), and so on through the lowercase block. This is why the text pipeline can store a glyph
index directly: for the basic set it is just the ASCII code minus the 32 control codes the atlas omits.
Rows 6 and below hold the **extended block** — accented-Latin capitals and lowercase, then a *second*
alphabet in a plainer weight (clearest in the blackletter `font2`, whose lower rows are ordinary sans
digits and letters).

The render **corrected a mistake**. An earlier draft reasoned `208 = 13 × 16` and concluded the alphabet
fills rows 0–12 and "rows 13–15 are unused space". Decoding the texture shows the opposite: **all 16 rows
are populated** — `font1` lights 254 of 256 cells, `font2` 240 — so the atlas carries roughly 256 glyphs,
while `fonts.dat`'s proportional-width table covers only the first **208** (indices 0–207, rows 0–12). The
~48 glyphs in rows 13–15 exist in the texture but have **no proportional width**; they are the tail of the
secondary alphabet and are placed by fixed-width or special-purpose HUD code rather than the proportional
path. `derive_fonts.py` now asserts both halves — the printable rows *and* rows 13–15 are lit — so the
correction cannot silently regress. It is a small worked example of the house rule that a render, not an
arithmetic guess, is what settles what a texture actually contains.

## 4. Width table and atlas share one index

The proportional-width table (208 entries) and the atlas (256 cells) are addressed by the **same** integer.
`fonts.dat` stores widths in 26 file-rows of 8; the first 208 atlas cells are the same 13 atlas-rows of 16:

```
fonts.dat :  26 rows × 8  = 208 widths        (file layout, 8-to-a-line)
atlas     :  the first 13 rows × 16 = 208 cells map 1:1 to those widths
index     :  the SAME integer g selects a width AND a cell:
             PROP[fontId × 210 + g]   and   (g & 0x0F, g >> 4)
```

There is no re-mapping between the two: the glyph index `g` that indexes `PROP[fontId × 210 + g]`
(C23.1 §4) is the identical `g` the draw code splits into `(g & 0x0F, g >> 4)`. A single integer addresses
both the "how far to advance" and the "what to draw" for indices 0–207, which is why the width table and
the atlas can never fall out of step over the range the width table covers. The atlas simply continues past
that range (§3) with glyphs the proportional path does not measure.

## 5. Two mask atlases the engine asks for and never gets

The font/HUD string block at `0x86A620` lists the texture names the loader requests from the dictionary:

```
ps2btns  font1  font1m  font2  font2m  MODELS\FONTS.TXD  fonts  MODELS\HUD.TXD
```

`fonts.txd` supplies `font1` and `font2`. It does **not** supply `font1m` or `font2m` — the two `m`-suffix
names are requested but absent from the shipped dictionary. The obvious reading of the suffix is a separate
mask channel (an outline or alpha companion to each atlas), but nothing in the shipped files confirms what
they would carry, or whether they ever existed. This is filed exactly as the chapter's siblings file their
own capabilities-without-data: [C19](../C19-GXT-Text/C19-GXT-Text.md)'s five named languages with two
shipped, [C21](../C21-Particles/C21-Particles.md)'s three compiled-but-unused particle types,
[C22](../C22-Map-Zones/C22-Map-Zones.md)'s cut `NAVIG.ZON`. The engine's reach exceeds the retail data's
grasp, and the gap is recorded, not guessed away.

---

### Key takeaways

- ✅ `fonts.txd` is **two 512 × 512 DXT3** textures, `font1` and `font2`, read by C9.3's `TextureNative`
  header — DXT3 being exactly the graded-alpha format glyph edges need.
- ✅ Glyphs tile a **16 × 16 grid of 32-pixel cells**; the executable splits the index into
  `col = index & 0x0F`, `row = index >> 4` and scales by `1/16` (VA `0x718B30`).
- ✅ **Decoding the DXT3 atlases confirms the grid visually** and pins the character mapping:
  **glyph index = ASCII − 0x20** for the printable block (rows 0–5); rows 6+ hold accented Latin and a
  secondary alphabet.
- ✅ **Correction from the render:** the atlas populates **all 16 rows** (~256 glyphs), while the width
  table covers only the first **208** (rows 0–12). The lower ~48 glyphs have no proportional width — an
  earlier "rows 13–15 unused" inference was wrong, and `derive_fonts.py` now guards against it regressing.
- ✅ The 208 widths map 1:1 to the first 208 atlas cells; the *same* index `g` addresses both
  `PROP[fontId × 210 + g]` and the cell `(g & 0x0F, g >> 4)`.
- ⏳ The engine requests two mask atlases, **`font1m` and `font2m`**, that `fonts.txd` never ships — a
  capability without data, like C19's languages and C21's particle types.

**Continue:** [C23.3 — The HUD texture dictionary](03-the-hud-txd.md)
