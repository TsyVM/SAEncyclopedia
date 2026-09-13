# C23.1 — `fonts.dat` and the 210-byte record

> **The one-sentence version:** `data/fonts.dat` is a plain-text table of per-glyph advance widths — two
> fonts, 208 widths each — and the executable both fills and reads that table at a stride of `0xD2`
> (`208 + 1 + 1`), a number every digit of which is read from the machine code rather than fitted to the
> file.

[Chapter 23 hub](C23-Fonts-HUD.md) · next: [C23.2 — The glyph atlas](02-the-glyph-atlas.md)

**Confidence:** ✅ Verified (grammar, record layout, stride, width lookup, monospace fallback; the glyph
byte is used directly as the atlas index — character mapping settled in C23.2)

---

## 1. The file is text

`data/fonts.dat` is 3,576 bytes and, apart from a handful of high-bytes used as inline annotation
characters in the trailing comments, it is plain ASCII. It follows the same "declared counts close"
discipline the project used for [C21](../C21-Particles/C21-Particles.md)'s `effects.fxp` and
[C22](../C22-Map-Zones/C22-Map-Zones.md)'s zone files: a `#` begins a comment (also used to annotate each
row with the characters it covers), and bracketed tags introduce sections.

```
[TOTAL_FONTS]
2

[FONT_ID]
0
[PROP]
12  13  13  28  28  28  28  8     #  0    ! " £ $ % & '
17  17  30  28  28  12  9   21     #  8  ( ) * + , - . /
...  (26 rows of 8) ...
[UNPROP]
27
[REPLACEMENT_SPACE_CHAR]
10
```

The tags are exactly the strings the executable compares against — `[TOTAL_FONTS]`, `[FONT_ID]`, `[PROP]`,
`[UNPROP]`, `[REPLACEMENT_SPACE_CHAR]` — all present in `.rdata` at `0x8728A0 … 0x8728FC`, and the file it
opens, `DATA\FONTS.DAT`, at `0x87290C`. The parser is one function at **VA `0x7187C0`**.

## 2. Two fonts, 208 widths each

The file declares `[TOTAL_FONTS] 2`, and exactly two `[FONT_ID]` blocks follow — font `0` (annotated
`GOTHIC`) and font `1` (annotated `GTA HEADER`). The count closes.

Each `[PROP]` block is **26 rows of 8 values = 208 widths**. This is not counted off the page and hoped;
it is the parser's own loop structure. The `[PROP]` branch sets up an outer counter and an inner counter:

```asm
007188EB  mov  edi, 0x1A          ; outer: 26 rows
...
00718932  mov  esi, 8             ; inner: 8 values per row
00718937  mov  dl, [ecx]          ; read one parsed int (low byte)
00718939  add  ecx, 4             ;   advance over the int
0071893C  mov  [eax], dl          ; store it as ONE byte
0071893E  inc  eax
0071893F  dec  esi
00718940  jne  0x718937           ; ... 8 times
00718942  dec  edi
00718945  jne  0x7188F0           ; ... 26 times
```

`26 × 8 = 208`. Two independent facts fall out of this loop at once. First, the widths are **stored as
bytes** — the `sscanf` parses each field as an `int` into a stack slot, and the copy takes only the low
byte (`mov dl, [ecx]` / `add ecx, 4`). The shipped values run `5 … 37`, comfortably inside a byte, so
nothing is lost. Second, the per-line format string is literally eight conversions —
`"%d  %d  %d  %d  %d  %d  %d  %d"` at `0x8728AC` — which is why the file is laid out 8-to-a-row.

## 3. The record is 210 bytes, and the exe says so twice

Before the fill loop, the `[PROP]` branch computes where this font's 208 bytes go:

```asm
007188DF  imul esi, esi, 0xD2      ; esi = fontId * 210
007188E5  add  esi, 0xC718B0       ; PROP base for this font
```

The two singleton fields land just past the 208 widths, at the same stride:

```asm
; [REPLACEMENT_SPACE_CHAR] branch
007188B7  imul edx, edx, 0xD2
007188BD  mov  [edx + 0xC71980], al     ; 0xC71980 = 0xC718B0 + 208
; [UNPROP] branch
0071897D  imul edx, edx, 0xD2
00718983  mov  [edx + 0xC71981], al     ; 0xC71981 = 0xC718B0 + 209
```

So the in-memory record for one font is:

| Offset | Field | Bytes |
|---:|---|---:|
| `+0` | `PROP[208]` — proportional advance widths | 208 |
| `+208` | `REPLACEMENT_SPACE_CHAR` | 1 |
| `+209` | `UNPROP` — the monospace advance width | 1 |
| | **stride** | **210 = `0xD2`** |

`0xD2 = 210 = 208 + 1 + 1`. Every term is read from the machine code: `208` from the `26 × 8` loop, the
two `+1`s from the two store offsets, and the total from the `imul …, 0xD2` that spaces the fonts. Note
also that `[TOTAL_FONTS]` is *parsed but not used as a bound* — the loader places each font's data by its
`FONT_ID` (`fontId × 210`), exactly the direct-indexing pattern seen in
[C20.5](../C20-Audio/05-eventvol.md)'s event-indexed `EventVol.dat` and C22's letter-indexed gridref.

## 4. The consumer confirms the stride independently

A record width is only as trustworthy as the code that *reads* it agreeing with the code that *wrote* it.
The width lookup, at **VA `0x7196F4`**, recomputes the identical address:

```asm
007196F4  movzx ecx, cl            ; cl = font id
007196F7  imul  ecx, ecx, 0xD2     ; * 210   <-- same stride
007196FD  movzx edx, al            ; al = glyph index
00719700  movzx eax, [ecx + edx + 0xC718B0]   ; PROP[fontId*210 + glyph]
```

The advance width of glyph `g` in font `f` is `PROP[f × 210 + g]`. The same `0xD2` and the same base
`0xC718B0` that the parser wrote through are the ones the renderer reads through; a wrong stride would
mis-address one of the two and the text would break. This is the C20/C18.6 standard of proof — the same
constant recovered from two independent sites.

## 5. `UNPROP` is the monospace fallback

The very next lines show what the second singleton field is for. The lookup takes a boolean "proportional"
flag; when it is clear, the function reads a *single* fixed width instead of the per-glyph one:

```asm
00719722  movzx edx, cl
00719725  imul  edx, edx, 0xD2
0071972B  movzx eax, [edx + 0xC71981]   ; UNPROP, the monospace width
```

So `UNPROP` is the advance used when a font is drawn non-proportionally (fixed pitch), and the per-glyph
`PROP` table is used otherwise. Font `0` uses `27`, font `1` uses `20`. In both branches the width is then
converted to a float, a per-call spacing term is added, and the result is multiplied by the caller's font
scale — i.e. the advance is `(width + spacing) × scale`.

## 6. The character remap

The `al` passed to the lookup is a *glyph index*, not a raw character code. A remap at `0x718770` /
`0x7192C0` turns the incoming byte into that index. The direct path is nearly the identity: codes
`0x00 … 0x9B` pass through, `0x91` is special-cased to glyph `0x40`, and anything above `0x9B` folds to
glyph `0`. A secondary switch at `0x7192C0` — selected by a per-font render-state value — maps a set of
control and high codes up into the **extended glyph rows** (for example `add al, 0x7A` over one input
range, `mov al, 0xCC/0xCD/0xCE/0xCF/0xD0` for individual codes). Index `0xD0` (208) lands on the
`REPLACEMENT_SPACE_CHAR` slot — a tidy consistency check on the `+208` offset.

What each glyph index *draws* is settled in [C23.2 §3](02-the-glyph-atlas.md) by decoding the atlas: the
printable block is `glyph index = ASCII − 0x20`, and the extended rows hold accented Latin and a secondary
alphabet. So the direct path is the identity precisely because the game stores glyph indices, which for the
basic set are ASCII minus the 32 omitted control codes — the byte the lookup receives is already the cell
number.

---

### Key takeaways

- ✅ `fonts.dat` is **plain text**: `#` comments, bracketed tags, `[TOTAL_FONTS] 2` closing with 2
  `[FONT_ID]` blocks.
- ✅ Each font is **208 proportional widths** — the parser's own `mov edi, 0x1A` (26) × `mov esi, 8` (8) —
  stored as **bytes**.
- ✅ The record is **210 = `0xD2` = 208 + 1 + 1**: widths, then `REPLACEMENT_SPACE_CHAR` at `+208`, then
  `UNPROP` at `+209`. The stride `0xD2` appears in **both** the parser (write) and the renderer (read).
- ✅ Width lookup is `PROP[fontId × 210 + glyph]` at VA `0x719700`; `UNPROP` is the **monospace fallback**.
- ✅ `[TOTAL_FONTS]` is parsed but the loader indexes storage by `FONT_ID` directly — direct-indexing, as
  in C20.5 and C22.
- ✅ The incoming byte is used **directly** as the glyph index (no runtime shift); what each index draws is
  read off the decoded atlas in [C23.2 §3](02-the-glyph-atlas.md) — `index = ASCII − 0x20` for the
  printable block, accented Latin and a secondary alphabet above it.

**Continue:** [C23.2 — The glyph atlas and its UV grid](02-the-glyph-atlas.md)
