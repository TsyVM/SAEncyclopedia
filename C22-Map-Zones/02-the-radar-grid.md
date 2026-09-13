# C22.2 — The Radar Grid

> **The one-sentence version:** `gta3.img` holds exactly 144 `radarNN.txd` tiles numbered 0 to 143 with no
> gap, the executable loads them in a loop bounded by `0x90`, and `6000 / 12 = 500` — three independent
> statements of the same 12 × 12 grid.

[← Chapter 22 hub](C22-Map-Zones.md) · [Prev: C22.1 — The zone tables](01-the-zone-tables.md) ·
[Next: C22.3 — `gridref.dat` →](03-gridref-and-the-build-tree.md)

**Confidence:** ✅ Verified (tile count, contiguous numbering, naming scheme, grid arithmetic, the
executable's loop bound, **and — new — the tile orientation**)

---

## 1. Counting the tiles

The radar textures are not loose files — they live inside `gta3.img`, the IMG VER2 archive
[C1.1](../C1-Streaming/01-img-ver2-archive-model.md) documents. Walking its directory gives 16,297
entries, of which 151 match `radar*`. Filtering to the strict pattern `radar<digits>.txd` leaves:

```
144 tiles:  radar00.txd, radar01.txd, … radar142.txd, radar143.txd
numbering:  0 … 143, contiguous, no gap and no duplicate
```

The seven `radar*` entries that are *not* tiles are models rather than textures — `radar_bit_01.dff`
through `radar_bit_05.dff`, plus `radarmast1_lawn.dff` and `radarmast1_lawn01.dff` — and are excluded by
the pattern rather than by judgement.

✅ *Verified:* **144 radar tiles**, numbered `0 … 143` with no gap.

144 is a square, and the square root is 12. Combined with the world span
[C22.1 §3](01-the-zone-tables.md) established from `map.zon`:

```
12 × 12  =  144 tiles
6000 / 12  =  500.0 units per tile          exactly, no remainder
```

✅ *Verified:* the radar grid is **12 × 12 at 500 units per tile**, tiling the 6,000-unit world square
with no gap and no overlap.

## 2. The executable agrees, twice

Counting archive entries proves what shipped. It does not prove what the engine *expects* — a game that
loaded 100 tiles and ignored 44 would produce the same directory listing. The executable settles both
halves.

**The naming scheme** is a format string at VA `0x866B98`:

```
"radar%02d"
```

referenced once, from `0x588015`. That is the whole naming rule, and it explains a detail that would
otherwise look inconsistent: `%02d` pads to a *minimum* of two digits, so tiles 0–99 are `radar00` …
`radar99` and tiles 100–143 are `radar100` … `radar143`. The names are not fixed-width, and a tool that
assumed two digits throughout would miss 44 of them.

**The count** is the loop that uses it:

```
00588010  ...                                  ; loop head
00588015  push   0x866b98                      ; "radar%02d"
0058801a  push   eax
0058801b  call   0x821bb5                      ; sprintf
00588020  lea    ecx, [esp + 0x10]
00588024  push   ecx
00588025  call   0x731850                      ; look the name up / load it
0058802a  add    esp, 0x10
0058802d  mov    dword ptr [esi*4 + 0xba8478], eax    ; store the handle
00588034  inc    esi
00588035  cmp    esi, 0x90                     ; 0x90 = 144
0058803b  jl     0x588010
```

`cmp esi, 0x90` — the loop runs for indices `0 … 143` and stops. ✅ **The executable expects exactly 144
tiles**, and stores their handles in a 144-entry array of dwords based at `0xBA8478`.

Two sources with no common failure mode — an archive built by an exporter and a loop emitted by a compiler
— agree on 144. That is the same standard of evidence [C20.4](../C20-Audio/04-the-loaders-in-the-executable.md)
applied to the audio record widths.

## 3. Three grids, three cell sizes

It is worth stating plainly that the radar grid does **not** align with either zone table or with the
gridref of [C22.3](03-gridref-and-the-build-tree.md):

| Grid | Cells | Cell size | Divides 6,000? |
|---|---:|---:|:--:|
| Radar | 12 × 12 = 144 | **500.0** | ✅ |
| `gridref.dat` | 10 × 10 = 100 | **600.0** | ✅ |
| `info.zon` | 378 zones | irregular | — |

All three divide the world square, but at different granularities, and 500 and 600 share no useful
relationship — a radar tile is neither a whole gridref cell nor a clean fraction of one. This is not a
discrepancy to reconcile. The radar grid is sized by texture memory, the gridref by how the art team
carved up the work, and the zones by where places actually are. They were never meant to line up.

## 4. ✅ The orientation — closed at the draw site

An earlier draft left this open. The reasoning then was correct — the evidence is not at the load site,
which only counts — so the thread to pull was the `0.002f` (`1 / 500`) at the *use* site, exactly the
[C20.5](../C20-Audio/05-eventvol.md) lesson that meaning lives where a value is consumed. Pulling it
settles the orientation completely.

The world-to-tile conversion is a small function at **VA `0x5858D0`**. It takes a world position and
produces a tile column and row:

```asm
005858D5  fld   [esi]                 ; worldX
005858D8  fadd  [0x859A94]            ; + 3000.0        (shift origin to the SW corner)
005858E1  fmul  [0x858F44]            ; * 0.002  (=1/500)  -> COLUMN
        ... ftol -> edi = col
005858F4  fld   [esi+4]               ; worldY
005858F7  fadd  [0x859A94]            ; + 3000.0
005858FF  fmul  [0x858F44]            ; * 0.002
00585905  fsubr [0x858FBC]            ; 11.0 - (that)   -> ROW   (the Y axis is FLIPPED)
        ... ftol -> eax = row
```

Two facts fall straight out of the signs. The column is `(worldX + 3000) / 500`, so it **increases with
X** — column 0 is the west edge, column 11 the east. The row is `11 − (worldY + 3000) / 500`; the `fsubr`
subtracts from 11, so the row **increases as Y decreases** — row 0 is the north edge, row 11 the south.
The `+3000` is the same world half-span the chapter pinned from `map.zon`, moving the origin to a corner;
the `1/500` is the tile size; and the `11 −` is the north–south flip.

The tile index is then `col + row × 12`, visible as the address arithmetic at the draw sites (`0x584B7B`
and `0x584BCA`):

```asm
00584B7B  lea   ecx, [ecx + ecx*2]         ; row * 3
00584B7E  lea   eax, [eax + ecx*4]         ; col + (row*3)*4 = col + row*12
00584B81  mov   eax, [eax*4 + 0xBA8478]    ; handle = tiles[index]
```

Feeding the four world corners through this arithmetic lands each on a definite tile:

| World corner | (X, Y) | col, row | Tile |
|---|---|---:|---:|
| **North-west** | (−3000, +3000) | 0, 0 | **0** |
| North-east | (+3000, +3000) | 11, 0 | 11 |
| South-west | (−3000, −3000) | 0, 11 | 132 |
| South-east | (+3000, −3000) | 11, 11 | 143 |

✅ **Tile 0 is the north-west corner**, and the numbering runs **west-to-east across a row, then top-to-
bottom (north-to-south) row by row** — row-major over a 12-wide grid. This is not inferred from the
picture; it is the executable's own coordinate arithmetic, and `derive_zones.py` re-checks both the
constants (`+3000`, `1/500`, the `11 −` flip) and the four corner mappings.

## 5. What is not derived

⏳ **What a tile contains.** Each `radarNN.txd` is a RenderWare texture dictionary, so decoding its
contents is [C9](../C9-Materials-And-Textures/C9-Materials-And-Textures.md)'s subject rather than this
chapter's. Nothing here inspects the pixels.

---

### Key takeaways

- ✅ `gta3.img` holds exactly **144** `radar<digits>.txd` tiles, numbered **0 … 143 with no gap**; the
  seven other `radar*` entries are `.dff` models and are excluded by pattern, not by judgement.
- ✅ The grid is **12 × 12 at exactly 500 units**, dividing the 6,000-unit world square established from
  `map.zon` with no remainder.
- ✅ The executable confirms both independently: the naming rule is `"radar%02d"` at VA `0x866B98`, and the
  loader loop at `0x588015` is bounded by **`cmp esi, 0x90`** — 144 — storing handles in an array at
  `0xBA8478`.
- ⚠️ `%02d` pads to a **minimum** of two digits: tiles 100–143 have three-digit names. A fixed-width
  assumption loses 44 tiles.
- 🧭 Radar (500), gridref (600) and zones (irregular) divide the same world square at different
  granularities and are **not** meant to align.
- ✅ **Tile orientation is closed:** the world-to-tile conversion at VA `0x5858D0` gives
  `col = (X + 3000) / 500` (west→east) and `row = 11 − (Y + 3000) / 500` (north→south), index
  `col + row × 12`. **Tile 0 is the north-west corner**; numbering runs west→east then north→south. The
  four world corners map to tiles 0 (NW), 11 (NE), 132 (SW), 143 (SE).

**Continue:** [C22.3 — `gridref.dat` and the build tree](03-gridref-and-the-build-tree.md)
