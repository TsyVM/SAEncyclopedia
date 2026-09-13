# C43.1 — The screens table

The front-end is one array. Everything the pause menu and the settings pages show is a row of
`aScreens`, a table of 44 fixed-size `tMenuScreen` structs baked into the executable at `0x8CE008`. This page
proves the base, the stride and the count, and reads the real screen titles straight out of the bytes.

## The stride, from the index multiply

`CMenuManager` reaches a screen by indexing `aScreens[m_nCurrentScreen]`, and the compiler emitted the struct
size as the multiply. In the menu code region the pattern

```
imul ecx, ecx, 0xE2        ; m_nCurrentScreen * sizeof(tMenuScreen)
```

appears **40 times** (`derive_menu.py`: `tmenuscreen_stride_0xE2`), so `sizeof(tMenuScreen) = 0xE2` (226
bytes). One indexing site shows the whole address computed at once:

```
0x57344D: imul  ecx, ecx, 0xE2               ; screen * 0xE2
0x573453: lea   edx, [eax + eax*8]           ; item * 9
0x573456: movzx eax, [ecx + edx*2 + 0x8CE012]; aScreens_base + screen*0xE2 + item*0x12
```

The `edx*2` on the `×9` makes `item × 0x12` — the *item* stride ([C43.2](02-the-menu-item.md)) — and the
displacement `0x8CE012` is a field inside `aScreens[0].m_aItems[0]`. Backing out the `0xA`-byte screen header
(title + parent + start), the array base is **`0x8CE008`**.

## The struct tiles

`tMenuScreen` closes with no residue:

```
m_szTitleName[8]      : +0x00,  8 bytes   (a GXT key)
m_nParentMenu         : +0x08,  1 byte    (eMenuScreen)
m_nStartEntry         : +0x09,  1 byte    (default highlighted item)
m_aItems[12]          : +0x0A, 12 × 0x12 = 0xD8
--------------------------------------------------
sizeof(tMenuScreen)   = 0x0A + 0xD8 = 0xE2
```

`0xA + 12 × 0x12 = 0xE2` (`derive_menu.py`: `tmenuscreen_tiles`) — the ten-byte header plus twelve
eighteen-byte item slots exactly fill the 226-byte struct. The `m_aItems[12]` count is why every menu screen
tops out at twelve visible options.

## Reading the table cold

The proof that `0x8CE008` and `0xE2` are right is that stepping the base by the stride yields coherent screen
titles — every one a [C19](../C19-GXT-Text/C19-GXT-Text.md) GXT key:

| # | `@` | Title key | Screen |
|--:|---|---|---|
| 0 | `0x8CE008` | `FEP_STA` | Stats |
| 1 | `0x8CE0EA` | `FEH_LOA` | Game (New/Load/Delete) |
| 2 | `0x8CE1CC` | `FEH_BRI` | Brief |
| 3 | `0x8CE2AE` | `FEH_AUD` | Audio settings |
| 4 | `0x8CE390` | `FEH_DIS` | Display settings |
| 5 | `0x8CE472` | `FEH_MAP` | Map |
| … | | | |
| 41 | `0x8D043A` | `FET_PAU` | Pause menu |

`derive_menu.py` asserts the exact titles at screens 0, 1, 3, 4 and 41 (`ascreens_titles_are_gxt`). A wrong
base or stride would desync and produce garbage at every step; the fact that all 44 line up on valid GXT keys
is the tiling proof for a struct array — the same standard C40's render lists and C42's handling array meet.

## The count is 44 — and the last two screens are empty

The array ends where the titles do. Screen 41 (`FET_PAU`) is the last titled screen; screens **42 and 43**
read as **empty titles**, and screen 44 onward is out-of-bounds garbage:

```
screen[41] = "FET_PAU"     ; SCREEN_PAUSE_MENU
screen[42] = ""            ; SCREEN_INITIAL  (the pre-game/quit screen, no title)
screen[43] = ""            ; SCREEN_EMPTY    (a deliberate empty slot)
screen[44] = <end of array @0x8D06E0>
```

So `SCREEN_COUNT = 44` (`derive_menu.py`: `screen_count_44`), and the array spans `0x8CE008 … 0x8D06E0`
(`44 × 0xE2 = 0x26D8`). The two empty trailing screens are not corruption — they are `SCREEN_INITIAL` and
`SCREEN_EMPTY`, special screens the code drives directly rather than from title/item data, exactly as
`gta-reversed`'s `eMenuScreen` enum declares them. Naming the empties is the house rule in miniature: the
exception is identified, not rounded away.

## Key takeaways

- `aScreens` is a hardcoded array at **`0x8CE008`** of **44** `tMenuScreen` structs, each **`0xE2`** bytes —
  base, stride and count all read from the exe.
- The struct **tiles**: `title[8] + parent + startEntry + 12 × 0x12-byte items = 0xE2`, which is why a screen
  shows at most 12 options.
- Stepping the table yields real GXT title keys (`FEP_STA` … `FET_PAU`); screens 42/43 are the deliberately
  empty `SCREEN_INITIAL`/`SCREEN_EMPTY`, fixing the count at 44.

**Continue:** [C43.2 — The menu item →](02-the-menu-item.md)
