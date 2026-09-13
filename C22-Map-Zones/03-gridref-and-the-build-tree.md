# C22.3 — `gridref.dat` and the Build Tree

> **The one-sentence version:** `gridref.dat` records which Rockstar artist owned each cell of a 10 × 10
> map grid — a production-management file that has no business in a shipped game — and the retail
> executable not only ships it but **still parses it**, with an index calculation this page recovers from
> a single `shl ecx, 5`.

[← Chapter 22 hub](C22-Map-Zones.md) · [Prev: C22.2 — The radar grid](02-the-radar-grid.md)

**Confidence:** ✅ Verified (the grid, its completeness, the record size, the index arithmetic, the fact
that the engine parses it, **and — now closed — that nothing reads the parsed table back: the reader API is
dead code**)

---

## 1. What the file says about itself

`data/gridref.dat` is 1,498 bytes of plain text, and it opens with a banner that is not addressed to
anyone who bought the game:

```
##################################################
# gridref.dat                                    #
# stores which artist deals with map grid areas. #
# These names must be the artist's user id       #
##################################################

# these names MUST!!!! be the artist's bugstar userid any questions see Alex !!!!!!!!!!

A1	GARY
A2	GARY
A3	GARY
   ⋮
```

The body is tab-separated `cell → name` pairs. "Bugstar" is Rockstar's internal bug tracker, so the second
column is a set of internal user IDs, and the eleven exclamation marks are a colleague's exasperation
preserved in a retail data file.

This chapter treats the comment as **evidence about the file's purpose**, since it is text shipped inside
the subject of study, not an outside source. What the comment cannot establish is whether the *engine*
does anything with it — §3 answers that, and the answer is not the obvious one.

## 2. The grid is complete

Stripping comments leaves exactly **100** records:

| Property | Measurement |
|---|---|
| Cell labels | `A1` … `J10` |
| Distinct letters | **10** (`A`–`J`) |
| Distinct numbers | **10** (`1`–`10`) |
| Cells present | **100 of 100** — the grid is complete, with no duplicate and no gap |
| Distinct names | **10** |

✅ *Verified:* a complete **10 × 10** grid. Against the 6,000-unit world square established in
[C22.1 §3](01-the-zone-tables.md):

```
6000 / 10 = 600.0 units per cell        exactly, no remainder
```

The ten names and their cell counts:

| Name | Cells | | Name | Cells |
|---|---:|---|---|---:|
| `STUARTM` | 34 | | `STEVEM` | 7 |
| `SCOTT` | 14 | | `NIK` | 5 |
| `ANDREWSO` | 12 | | `ADAMC` | 4 |
| `GARY` | 9 | | `JIMA` | 4 |
| `WAYLAND` | 8 | | `SIMONL` | 3 |

Ten people, a hundred cells, one of them owning a third of the map. The distribution is the sort of detail
that has no engineering consequence and is worth recording anyway, because it is a measurement of how the
world was actually built.

## 3. The engine parses it

The expectation, given §1, is that this is a stray file — the kind of thing that ends up in a shipped data
directory because nobody removed it. It is stray in the sense that it should not be there. It is **not**
stray in the sense of being ignored.

The path is in `.rdata`, upper-cased, exactly as the audio config paths were
([C20.4 §2](../C20-Audio/04-the-loaders-in-the-executable.md) — the same lesson about case-sensitive
searching applies):

```
.rdata @ 0x872954:  "%c%d %s"
        @ 0x87295C:  "DATA\GRIDREF.DAT"
```

Both are referenced exactly once, from the same function:

```
0071d500  push   0x87295c              ; "DATA\GRIDREF.DAT"
0071d505  call   0x538900              ; open
0071d50a  mov    esi, eax
0071d50d  call   0x536f80              ; read a line
0071d515  test   eax, eax
0071d517  je     0x71d587              ; end of file
0071d520  mov    cl, byte ptr [eax]
0071d522  cmp    cl, 0x23              ; '#'
0071d525  je     0x71d57a              ; …skip comment lines
```

✅ *Verified:* the retail executable opens `DATA\GRIDREF.DAT`, reads it line by line, and **skips lines
beginning with `#`** — which is why the banner is harmless. It then parses each surviving line with the
format `"%c%d %s"`: a character, a number, a string. That is precisely `A`, `1`, `GARY`.

## 4. Recovering the record size from one shift

The interesting part is what it does with the parsed cell:

```
0071d53a  push   0x872954              ; "%c%d %s"
0071d53f  push   eax
0071d540  call   0x8220ad              ; sscanf
0071d549  push   edx
0071d54a  call   0x836c58
0071d54f  movzx  ecx, byte ptr [esp + 0x1f]     ; the letter
0071d554  mov    edx, dword ptr [esp + 0x20]    ; the number
0071d558  lea    ecx, [ecx + ecx*4]             ; ecx = letter * 5
0071d55b  lea    ecx, [edx + ecx*2 - 0x28b]     ; ecx = number + letter*10 - 651
0071d562  shl    ecx, 5                         ; ecx *= 32
```

Three instructions carry the whole layout. Working the arithmetic through with `letter = 'A' = 65`:

```
index = (letter * 10 + number - 651) * 32
      = ((letter - 'A') * 10 + (number - 1)) * 32       since 65 × 10 = 650
```

Two facts fall out and both are checkable against the file:

**Ten cells per row.** The multiplier on the letter is 10, so the code expects the numeric part to run
`1 … 10` before the letter advances. The file has exactly ten numbers per letter. ✅

**A 32-byte record.** `shl ecx, 5` is a multiply by 32 — the stride of whatever array the parsed name is
stored into. A name field of 32 bytes is generous for `ANDREWSO` and unremarkable as a fixed buffer. ✅

And the two together are self-checking: feeding all 100 cell labels through the recovered expression
yields `0, 32, 64, … 3168` — the complete arithmetic sequence from `0` to `99 × 32`, with **no collision
and no gap**. A wrong multiplier or a wrong bias would fold two cells onto one index or leave holes.

✅ *Verified:* the loader maps `A1 … J10` onto **100 records of 32 bytes**, index
`((letter − 'A') × 10 + (number − 1)) × 32`.

## 5. ✅ What it is used for: nothing — the reader API is dead code

An earlier draft left this open and suggested a bug-reporting overlay. Following the table's base pointer to
its consumers — the same bounded exercise that solved [C20.5](../C20-Audio/05-eventvol.md) — gives a
cleaner and more surprising answer than the guess.

The parsed names are stored into a 100-record table at **`0xC72FB0`** (the loader's `add ecx, 0xC72FB0`,
and the same base in the getter below). Three functions form the table's intended API, and each is exactly
what a position→owner overlay would need:

| VA | What it does |
|---|---|
| `0x71D5A0` | world `(x, y)` → grid `(col, row)`: `+3000`, `× 1/600`, and a `9 −` flip on Y — the gridref twin of the radar conversion in [§C22.2](02-the-radar-grid.md) |
| `0x71D650` | validity check: both indices `< 10`, else return the default string |
| `0x71D670` | the getter: `(row, col) → &table[(col + row·10)·32]`, then `atoi` on the name — i.e. the artist's bugstar user ID as a number |

Now count the references to each of them, across every section of the executable — `.text`, the
HOODLUM-relocated code, `_TEXT_HA`, everything:

| Function | Call / jump refs | Pointer refs |
|---|---:|---:|
| Loader `0x71D4E0` | **1** (from `0x5BA392`) | 0 |
| world→cell `0x71D5A0` | **0** | 0 |
| world→cell `0x71D5E0` | **0** | 0 |
| validity `0x71D650` | **0** | 0 |
| getter `0x71D670` | **0** | 0 |

The loader **is** called — once, from an initialisation sequence at `0x5BA392` that runs a long row of
subsystem `call`s — so the table really is populated at load, and §3's "the engine parses it" stands. But
every function that would *read* the table has **zero references of any kind**. Nothing calls them, nothing
stores their address, no jump reaches them. They are compiled-in dead code.

✅ **The parsed gridref table is write-only in retail.** The engine loads `gridref.dat`, converts each cell
to a 32-byte record, fills the table at `0xC72FB0` — and then never looks at it again. The
position→owner feature the file's banner implies was evidently cut before release; what survived is the
loader, still wired into init, and its orphaned reader API. So the file did not merely outlive its purpose
in the *data* tree — its whole read path outlived its purpose in the *code*, one `call` short of being
reachable.

This is the same disposition [C20.5](../C20-Audio/05-eventvol.md) took: a negative result, fully proven, is
a result. `tools/derive_zones.py` re-checks it — the loader has a caller, the four accessors have none.

---

### Key takeaways

- ✅ `gridref.dat` is a complete **10 × 10** grid (`A1`–`J10`, 100 cells, no gap) of `cell → name` pairs,
  covering the 6,000-unit world at **600 units per cell**.
- 🧭 The names are Rockstar artists' internal bug-tracker IDs, per the file's own banner; ten people, with
  one owning 34 of the 100 cells.
- ✅ The retail executable **parses it**: `"DATA\GRIDREF.DAT"` at `0x87295C`, loader at `0x71D500`, comment
  lines (`#`) skipped, records read with `"%c%d %s"`.
- ✅ The record size and row width come from three instructions —
  `lea`/`lea`/`shl ecx, 5` gives `index = ((letter − 'A') × 10 + (number − 1)) × 32`. All 100 cells map
  onto `0 … 3168` step 32 with **no collision and no gap**.
- ⚠️ The path is **upper-case** in `.rdata`, as the audio paths were — a case-sensitive search finds
  nothing.
- ✅ **What consumes the parsed table: nothing.** The loader (`0x71D4E0`) is called once from init, so the
  100-record table at `0xC72FB0` is populated — but the four accessor functions (world→cell ×2, a validity
  check, and the getter) have **zero** call, jump and pointer references anywhere in the executable. The
  table is **write-only** in retail; the position→owner feature was cut, leaving its reader API as dead
  code.

**Continue:** [← Chapter 22 hub](C22-Map-Zones.md)
