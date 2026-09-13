# C19.1 — The Container

> **The one-sentence version:** a four-byte header, a `TABL` directory whose size divides by 12 into
> exactly 127 entries, and 127 `TKEY`/`TDAT` pairs that — with nothing but 4-byte alignment padding
> between them — account for **every one of the 738,256 bytes**.

[← Chapter 19 hub](C19-GXT-Text.md) · [Next: C19.2 — The key hash →](02-the-key-hash.md)

**Confidence:** ✅ Verified (structure, exact tiling, character width) / ⏳ (one header word)

---

## 1. The header is four bytes

```
offset 0:  04 00  08 00
           ^^^^^  ^^^^^
           word=4   bitsPerChar = 8
```

Two little-endian `uint16`s. The second is **provably the character width**: decoding the string data as
16-bit UTF-16 (the scheme GTA III and Vice City use) produces garbage, while decoding it as 8-bit produces
clean text on every sampled entry.

✅ *Verified:* `bitsPerChar = 8`. **San Andreas moved its text to one byte per character**, which is why
the file needs a codepage for accents ([C19.3](03-two-languages.md)) — a trade the 16-bit predecessors did
not have to make.

⏳ The **first word is `4` on both language files** and its meaning is not derived. It is plausibly a
version or format tag, but this chapter records only what it can prove: the word is constant across the two
files it has, and the byte after it is the character width.

## 2. `TABL`: the directory divides by 12

At offset 4:

```
'TABL'  size = 0x05F4 = 1,524
```

`1,524 / 12 = 127`, with no remainder — so the directory is **127 records of 12 bytes**, each an 8-byte
table name and a `uint32` file offset:

| Field | Width |
|---|---:|
| `char name[8]` | 8 |
| `uint32 offset` | 4 |

✅ **The remainder is the proof.** A record width guessed one byte wrong leaves `1,524 / 13` or `/ 11` with
a remainder; only 12 divides cleanly. It is the same arithmetic that fixed the segment records in
[C18.1](../C18-SCM-Script/01-the-segment-chain.md) and the `CPool` element size in
[C4.5](../C4-Entities-And-Pools/05-the-cpool-object.md) — **the count either divides or it does not.**

Entry 0 is `MAIN`, offset **1,536**. The header plus the `TABL` block occupy `4 + 8 + 127 × 12 = 1,536`
bytes exactly, so **`MAIN` begins on the first byte after the directory** — no gap, no header slack.

## 3. A table is a name, a `TKEY`, and a `TDAT`

Every table past `MAIN` opens with its own 8-byte name, then two tagged blocks:

```
"AMBULAE\0"                          ← 8-byte name (MAIN omits this)
'TKEY'  size   →  N × { uint32 tdatOffset ; uint32 keyHash }
'TDAT'  size   →  the string bytes, 8-bit, NUL-terminated
```

`MAIN` is the exception that proves the rule: its directory offset points **straight at its `TKEY`**, with
no name prefix, because the loader already knows it by position. Every other table repeats its name at its
own offset — the same **name-reproduces-its-own-index** pattern seen in `areaId`
([C12.1 §3](../C12-Path-Network/01-nodes-dat.md)) and `packageName`
([C17.1 §3](../C17-IFP-Animation/01-anp3-container.md)): a field that restates what the directory already
says cannot be at the wrong offset, and here it confirms the 8-byte name width a second time.

`MAIN` is by far the largest table — **5,428 keys, 147,945 bytes of text** — holding all the
mission-independent strings: place names, menus, stats, pickups. The 126 named tables are mission text,
and the largest of them are the story set-pieces:

| Table | Keys |
|---|---:|
| `MAIN` | 5,428 |
| `CAT` | 459 |
| `RIOT4` | 309 |
| `LAFIN1` | 243 |
| `CASINO4` | 214 |

## 4. The file tiles to the byte

Summing every block — header, directory, and each table's name + `TKEY` + `TDAT` — comes up **195 bytes
short** of the 738,256-byte file. Those 195 bytes are not lost; they are **alignment**:

| Check | Result |
|---|---:|
| Table start offsets divisible by 4 | **127 / 127** ✅ |
| Bytes in the gaps | all `0x00` ✅ |
| Gap size vs `(4 − blockEnd mod 4) mod 4` | matches on every table ✅ |
| header + `TABL` + Σ(blocks + pad) | **738,256** ✅ |

✅ **Every table begins on a 4-byte boundary**, and the 0-to-3 `00` bytes after each `TDAT` are exactly the
padding that makes the next one align. Counting them, the file is fully tiled — the same standard of
evidence as the mission table partitioning `main.scm` in
[C18.2 §2](../C18-SCM-Script/02-mission-and-external-tables.md): no gap that is not accounted for, no
overlap.

⚠️ **A reader that sums block sizes without rounding up to 4 will drift**, landing one to three bytes early
on each table and eventually reading a `TKEY` tag as text. The alignment is not decorative; it is part of
how the offsets in the directory stay correct.

## 5. Reading it

```python
def gxt(d):
    ver, bits = struct.unpack_from('<HH', d, 0)          # 4, 8
    assert d[4:8] == b'TABL'
    n = struct.unpack_from('<I', d, 8)[0] // 12           # 127
    tables = [(d[12+12*i:20+12*i].split(b'\0')[0],
               struct.unpack_from('<I', d, 20+12*i)[0]) for i in range(n)]
    for name, off in tables:
        if name != b'MAIN': off += 8                      # skip the repeated name
        assert d[off:off+4] == b'TKEY'
        nk = struct.unpack_from('<I', d, off+4)[0] // 8
        keys = [struct.unpack_from('<Ii', d, off+8+8*k) for k in range(nk)]  # (tdatOff, hash)
        tdat_off = off + 8 + nk*8
        assert d[tdat_off:tdat_off+4] == b'TDAT'
```

The loop needs no length field it is not given: the directory sizes `TABL`, each `TKEY` sizes itself, and
`TDAT` follows. ✅ On retail `american.gxt` it yields 127 tables and 16,588 keys.

---

### Key takeaways

- The header is **four bytes**: a constant word `4` (⏳ meaning open) and `bitsPerChar = 8`, the latter
  **proven** because 16-bit decoding fails and 8-bit decoding is clean.
- `TABL` size **`1,524 = 127 × 12`** fixes the directory at 127 records of `name[8] + offset[4]`; `MAIN`
  starts at 1,536, exactly where the directory ends.
- Each table is **`[name[8]] + TKEY + TDAT`**; `MAIN` omits the name because the loader knows it by
  position, and the others **reproduce their own name** — a self-confirming field.
- ✅ The file **tiles to the byte** once **4-byte table alignment** (195 `00` pad bytes) is counted; every
  table start is divisible by 4.
- ⚠️ Summing block sizes **without** rounding to 4 drifts by 1–3 bytes per table and eventually reads a tag
  as text.

**Continue:** [C19.2 — The key hash](02-the-key-hash.md)
