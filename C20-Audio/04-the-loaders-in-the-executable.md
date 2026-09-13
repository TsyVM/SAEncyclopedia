# C20.4 — The Loaders in the Executable

> **The one-sentence version:** six string references lead to six loaders, and each one divides the file
> size by its record width using a compiler-generated reciprocal-multiply — so every width this chapter
> derived from arithmetic on the data is confirmed a second time by the instruction that computes it.

[← Chapter 20 hub](C20-Audio.md) · [Prev: C20.3 — The stream pack](03-the-stream-pack.md)

**Confidence:** ✅ Verified (all six loaders, all six widths, the `EventVol` read size)

---

## 1. Why bother, when the arithmetic already closed

[C20.1](01-the-config-tables.md) fixed every record width by division with no remainder, and
[C20.2](02-the-sfx-bank.md) and [C20.3](03-the-stream-pack.md) showed the containers tile to the byte. On
the encyclopedia's own standard that is proof. So this page is not needed to *establish* the widths.

It is worth doing anyway, for the reason [C18.6](../C18-SCM-Script/06-the-dispatch-table.md) set
out in the other direction. There, the executable was the weak oracle and closure was the strong test.
Here the relationship is inverted: the data closed first, and the executable serves as an **independent
witness** produced by a different process entirely — a compiler in 2004, rather than an exporter. When two
sources with no common failure mode agree on a number, the number is not an artefact of how it was
measured.

There is also a practical payoff. `EventVol.dat` does not divide ([C20.1 §4](01-the-config-tables.md)) and
the data alone says nothing about how much of it matters. The executable answers that outright.

## 2. Finding the loaders

The path strings are present in `.rdata` in **upper case**, which is worth stating because a
case-sensitive search for the on-disk filenames finds nothing and invites the wrong conclusion that the
strings are packed away:

```
.rdata @ 0x85F118:  "AUDIO\CONFIG\BANKLKUP.DAT"
                    "AUDIO\SFX\"
        @ 0x85F140:  "AUDIO\CONFIG\PAKFILES.DAT"
        @ 0x85F15C:  "AUDIO\CONFIG\BANKSLOT.DAT"
        @ 0x85F184:  "AUDIO\CONFIG\STRMPAKS.DAT"
        @ 0x85F1A0:  "AUDIO\CONFIG\TRAKLKUP.DAT"
                    "AUDIO\STREAMS\"
        @ 0x86A440:  "AUDIO\CONFIG\EVENTVOL.DAT"
```

Each has exactly **one** cross-reference, and each is a `push imm32` immediately before the open call —
so there is no ambiguity about which function loads which file. (Section layout comes from the PE header;
`gta_sa.exe` has two `.text` sections and the `.HOODLUM` section documented in
[C0.1](../C0-Binary-Identity/01-the-hoodlum-layer.md). None of the six loaders is relocated, so no resolution
through `hoodlum_relocation_map.json` was needed.)

| File | String VA | Loader reference |
|---|---|---|
| `BANKLKUP.DAT` | `0x85F118` | `0x4DFBD7` |
| `PAKFILES.DAT` | `0x85F140` | `0x4DFC7D` |
| `BANKSLOT.DAT` | `0x85F15C` | `0x4E0597` |
| `STRMPAKS.DAT` | `0x85F184` | `0x4E0982` |
| `TRAKLKUP.DAT` | `0x85F1A0` | `0x4E0A02` |
| `EVENTVOL.DAT` | `0x86A440` | `0x5B9D68` |

## 3. The division idiom

None of these loaders contains a `div` instruction. Dividing by a compile-time constant is expensive, so
the compiler emits a **reciprocal multiply**: multiply by a magic constant, keep the high 32 bits, shift.
Recognising the idiom is the whole skill here.

`BankLkup`, at `0x4DFC10`, having just read the file size into `esi`:

```
004dfc10  mov   eax, 0xAAAAAAAB      ; magic
004dfc15  mul   esi                  ; edx:eax = size × magic
004dfc17  shr   edx, 3               ; edx = size / 12
004dfc1a  movsx eax, dx              ; the count is taken as a 16-bit value…
004dfc24  mov   word ptr [ebx+0xE], dx   ; …and stored as a uint16 field
004dfc1d  lea   eax, [eax + eax*2]   ; ×3
004dfc20  shl   eax, 2               ; ×4  →  count × 12
004dfc23  push  eax
004dfc28  call  <allocate>           ; allocate count × 12 bytes
```

`mulhi(n, 0xAAAAAAAB) >> 3` is unsigned division by 12 — verified exhaustively over `n` in `[0, 5000)`
and `[200000, 205000)`, which brackets the real file. The allocation then multiplies straight back up by
12 via `lea`/`shl`. **The executable computes the record count as `filesize / 12` and allocates
`count × 12`.** ✅

The same idiom, three more times:

| Loader | Instructions | Divisor |
|---|---|---:|
| `PakFiles` @ `0x4DFCC9` | `mov eax, 0x4EC4EC4F` · `mul esi` · `shr edx, 4` · `imul eax, eax, 0x34` | **52** |
| `TrakLkup` @ `0x4E0A4B` | `mov eax, 0xAAAAAAAB` · `mul edi` · `shr edx, 3` | **12** |
| `StrmPaks` @ `0x4E09D6` | `shr eax, 4` | **16** |

`PakFiles` is the prettiest of the four: `mulhi(n, 0x4EC4EC4F) >> 4` is division by 52 — again verified
exhaustively — and the very next instruction, `imul eax, eax, 0x34`, multiplies by 52 (`0x34`) to size the
allocation. The divisor and the multiplier are the same number, written two different ways, four
instructions apart. `StrmPaks` needs no magic at all because 16 is a power of two, so its width shows up
as a bare `shr eax, 4`.

`BankSlot` is the one file with a header, and its loader reads accordingly — a two-byte count first
(`cmp ebp, 2`, then `lea ecx, [ebp-2]` for the remainder), then:

```
004e05e2  imul  eax, eax, 0x12D4     ; 0x12D4 = 4,820
```

**Slot count × 4,820**, exactly the structure [C20.1 §3](01-the-config-tables.md) derived from
`216,902 − 2 = 45 × 4,820`. ✅

## 4. `EventVol`: a fixed length, and a byte nobody reads

`EventVol.dat` has no divisor and no count. Its loader does not look for one — it reads a hard-coded
length and rejects the file if the read comes up short:

```
005b9d68  push  0x86A440             ; "AUDIO\CONFIG\EVENTVOL.DAT"
005b9d72  call  <open>
005b9d86  push  0xB159               ; 45,401 bytes
005b9d8b  push  edx                  ; destination
005b9d8c  push  edi                  ; handle
005b9d8d  call  <read>
005b9d95  cmp   eax, 0xB159          ; exact match required
005b9d9b  je    0x5b9dab             ; …otherwise the load fails
```

The file on disk is **45,402** bytes. The game reads **45,401**. ✅ *Verified:* the final byte of
`EventVol.dat` is never read by the retail executable.

That is a genuinely useful negative. It rules out any candidate record width that requires all 45,402
bytes — including 6, 14, 42, 46 and 47, every "nice" divisor the file size offers. Whatever the layout is,
it is a structure of 45,401 bytes and the trailing byte is padding. The next step is to look at the
**use** sites rather than the load site: the buffer's base pointer is stored at `0xBD00F8`.

> **Followed up in [C20.5](05-eventvol.md).** The 89 references to that pointer settle it — there is no
> row width, because there are no rows. The file is a flat `int8[45,401]` indexed directly by audio event
> ID, and the absence of any divide arithmetic in this loader is itself the clue: a one-byte element needs
> none.

## 5. The two-source summary

| Structure | From the data | From the executable | Agree |
|---|---|---|:--:|
| `PakFiles` record | `468 / 9 = 52`, no residue | `mulhi(·, 0x4EC4EC4F) >> 4`; `imul …, 0x34` | ✅ |
| `StrmPaks` record | `272 / 17 = 16`, no residue | `shr eax, 4` | ✅ |
| `BankLkup` record | `8,520 / 710 = 12`, no residue | `mulhi(·, 0xAAAAAAAB) >> 3`; `lea`+`shl` ×12 | ✅ |
| `TrakLkup` record | `23,064 / 1,922 = 12`, no residue | `mulhi(·, 0xAAAAAAAB) >> 3` | ✅ |
| `BankSlot` record | `(216,902 − 2) / 45 = 4,820` | `imul eax, eax, 0x12D4` | ✅ |
| `EventVol` extent | — (does not divide) | `push 0xB159`; `cmp eax, 0xB159` | n/a |

Six for six, with no disagreements to explain away.

---

### Key takeaways

- ✅ All six config filenames appear **upper-case** in `.rdata`, each with exactly one `push imm32`
  cross-reference — six loaders, unambiguously identified. None is HOODLUM-relocated.
- ✅ Every record width in the chapter is confirmed a second time by the loader's **reciprocal-multiply
  divide**: 52, 16, 12, 12 and a literal `imul …, 0x12D4` for 4,820.
- 🧭 The idiom to recognise is `mov eax, <magic>` · `mul` · `shr edx, k` — division by a constant with no
  `div` instruction. The magic constants here were checked exhaustively, not eyeballed.
- ✅ `EventVol.dat` is read as a fixed **45,401**-byte block and the read length is enforced; the file's
  last byte is never read. This **eliminates** every clean divisor of 45,402 as a candidate row width.
- 🧭 Two sources with no common failure mode — a 2004 compiler and a 2004 exporter — agree on all six
  numbers. That is a stronger position than either alone.

**Continue:** [C20.5 — `EventVol.dat`](05-eventvol.md)
