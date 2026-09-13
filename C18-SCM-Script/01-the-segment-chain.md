# C18.1 — The Segment Chain

> **The one-sentence version:** six `GOTO` instructions carve `main.scm` into six segments, every one of
> which accounts for its length exactly — and the largest of them, 43,800 bytes of global variables, is
> entirely zero.

[← Chapter 18 hub](C18-SCM-Script.md) ·
[Next: C18.2 — The mission and external tables →](02-mission-and-external-tables.md)

**Confidence:** ✅ Verified (structure, all six sizes) / ⏳ (three constants)

---

## 1. The file opens with a jump

```
offset 0:  02 00  01  20 ab 00 00
           ^^^^^  ^^  ^^^^^^^^^^^
           opcode  |  int32 = 43808
           0x0002  parameter type 0x01
```

`0x0002` is `GOTO`. Parameter type `0x01` means "the next four bytes are a little-endian int32". The
whole instruction is **seven bytes**, and the target is the start of the next instruction — which is
another `GOTO`.

Six of these run in sequence. Each one's target is the following jump, and the last one's target is the
first real instruction of the program.

| Jump at | Target | Segment body | ID byte |
|---:|---:|---|---:|
| 0 | 43,808 | 7 … 43,808 | `0x73` |
| 43,808 | 53,156 | 43,815 … 53,156 | `0x00` |
| 53,156 | 53,720 | 53,163 … 53,720 | `0x01` |
| 53,720 | 55,948 | 53,727 … 55,948 | `0x02` |
| 55,948 | 55,960 | 55,955 … 55,960 | `0x03` |
| 55,960 | 55,976 | 55,967 … 55,976 | `0x04` |

✅ *Verified.*

**The header is expressed in the language it introduces.** Nothing in the file says "this is a segment
table" — an interpreter that has never heard of segments simply executes six jumps and arrives at the
code. The structure exists only in the loader's willingness to look at what it skipped over.

🟡 *Reasoned:* this is the same design instinct as RenderWare's section stream
([C7.1](../C7-RenderWare-Stream/01-the-section-stream.md)), where a reader that does not understand a
section can still skip it correctly. Both formats make ignorance survivable.

## 2. Every size comes out exact

The one byte immediately after each jump is a **segment ID**, and the rest is a record array. Predicting
each segment's length from its own declared counts:

| Segment | ID | Formula | Predicted | Actual | |
|---|---:|---|---:|---:|---|
| 1 globals | `0x73` | `1 + 43,800` | 43,801 | **43,801** | ✅ |
| 2 models | `0` | `1 + 4 + 389 × 24` | 9,341 | **9,341** | ✅ |
| 3 missions | `1` | `1 + 16 + 135 × 4` | 557 | **557** | ✅ |
| 4 externals | `2` | `1 + 8 + 79 × 28` | 2,221 | **2,221** | ✅ |
| 5 unknown | `3` | `1 + 4` | 5 | **5** | ✅ |
| 6 trailer | `4` | `1 + 8` | 9 | **9** | ✅ |

✅ **Six for six.** Every byte between two jumps is accounted for.

This is the strongest form of structural evidence available in a format with no magic numbers: a record
width guessed one byte wrong leaves a remainder, and no remainder appears anywhere. It is the same test
that fixed the `CPool` element sizes in
[C4.5](../C4-Entities-And-Pools/05-the-cpool-object.md) — **the arithmetic either closes or it does not.**

Note that segment 1's ID byte is proved *by subtraction*: 43,808 − 7 = 43,801, and segment 6 independently
declares the globals block to be 43,800 bytes. The difference is one byte, so there is exactly one byte
of header — and it is not part of the variable space.

## 3. Segment 1: 43,800 bytes of nothing

| Measurement | Value |
|---|---:|
| Globals block | **43,800 bytes** |
| Global variables (`/4`) | **10,950** |
| **Non-zero bytes in the block** | **0** |

✅ Verified over the full range `[8, 43808)`.

**The entire global variable space ships zeroed.** Not sparsely populated, not mostly zero — literally
every byte.

🟡 *Reasoned:* the compiler reserves the block at its final size and leaves initialisation to the program.
That reading is supported directly by the code: eight of the 79 external scripts open with a run of
`SET_VAR_INT` instructions writing to globals ([C18.3 §4](03-the-code-stream.md)), and the main script's
opening sequence does the same at length.

**For tooling this is convenient.** A mod that needs new globals can extend the block without disturbing
any existing value, because there are no existing values — only the block's declared size in segment 6
has to move with it.

### Global references are byte offsets, not indices

Parameter type `0x02` introduces a global variable reference as a `uint16`. Every such value observed at a
known-good instruction position is **divisible by 4 and less than 43,800**:

```
05 00 | 02 c4 95 | ...      $38340   38340 / 4 = 9585   ✅
04 00 | 02 90 9d | ...      $40336   40336 / 4 = 10084  ✅
04 00 | 02 3c 9c | ...      $39996   39996 / 4 =  9999  ✅
```

✅ Verified on every sampled reference. **The number in the instruction is a byte offset into segment 1**,
which is why the divisibility holds and why the bound is 43,800 rather than 10,950.

That matters for a decompiler: a tool printing `$38340` and a tool printing `$9585` are both correct and
they disagree, so the convention has to be stated.

## 4. Segments 2 through 6

**Segment 2 — the model list.** A `uint32` count of 389, then 389 fixed 24-byte names. Index 0 is empty;
the remaining 388 are model names the scripts request. All 388 appear in the IDE files
([hub §18.4](C18-SCM-Script.md)).

**Segment 3 — the mission table.** Four dwords then 135 offsets. Covered in
[C18.2 §1](02-mission-and-external-tables.md).

**Segment 4 — the external script table.** Two dwords then 79 twenty-eight-byte records. Covered in
[C18.2 §3](02-mission-and-external-tables.md).

**Segment 5 — a single dword, value `0`.** Five bytes total. ⏳ A segment that exists to declare nothing
is worth noticing rather than skipping: 🟡 *reasoned*, it is a count of a record type San Andreas does not
use, kept so the loader's segment walk stays uniform.

**Segment 6 — the trailer.** Two dwords: **43,800** and **569**. The first is the globals block size and
matches segment 1 exactly ✅. The second is ⏳ underived.

## 5. ⚠️ `aaa.scm` is a copy of segment 6

The alphabetically first entry in `script.img` is `aaa.scm`, eight bytes long. Those eight bytes are:

```
18 ab 00 00  39 02 00 00      →   43800, 569
```

**Byte for byte identical to segment 6's payload.** ✅ Verified — and it is the *only* one of the 79
external scripts that contains that pair anywhere.

Its record in segment 4 is stranger still: `("AAA", base = 0, size = 8)`. Every other external script has a
base address in the virtual space above `main.scm` ([C18.2 §3](02-mission-and-external-tables.md)); this
one is mapped to offset **0**.

🟡 *Reasoned:* `aaa.scm` is a **build artefact**, not a script. The compiler emitted the trailer record it
writes at the end of `main.scm` into a stub file, and the stub got swept into the archive with everything
else. The name — first in sort order, three characters, no meaning — reads the same way.

This is the fourth accidental artefact this encyclopedia has found in shipped data, after the stale heap
pointer in [C12.1 §1.1](../C12-Path-Network/01-nodes-dat.md), the `3dsmax5` string in
[C17.1 §4](../C17-IFP-Animation/01-anp3-container.md), and the uninitialised name-field tails in the same
page. **Build systems leak, and what leaks is informative** — this one tells us the trailer is produced by
the compiler as a unit, not composed inline.

## 6. Reading it

```python
def scm_segments(b):
    off, segs = 0, []
    while True:
        assert b[off:off+2] == b'\x02\x00' and b[off+2] == 0x01   # GOTO int32
        target = struct.unpack_from('<i', b, off+3)[0]
        segs.append((b[off+7], off+8, target))                    # id, data start, end
        off = target
        if b[off:off+2] != b'\x02\x00':                           # first real instruction
            return segs, off
```

The loop terminates on the first non-`GOTO` — which is exactly how the game must do it, since the segment
*count* is written down nowhere. ✅ On retail `main.scm` it yields six segments and a code offset of
**55,976**.

⚠️ **Do not hard-code six.** The count is discovered, not declared, and a modified `main.scm` may carry
more.

---

### Key takeaways

- The file is **six chained `GOTO` instructions**; the header is written in the instruction set it
  prefixes, so a naive interpreter still reaches the code.
- ✅ **All six segment sizes are predicted exactly** by count × width — no remainder anywhere.
- Segment 1 is **43,800 bytes / 10,950 globals and is entirely zero** — nothing is initialised in the file.
- **Global references are byte offsets**, not indices: every observed value is divisible by 4 and below
  43,800.
- ⚠️ **`aaa.scm` is byte-identical to segment 6** and mapped to virtual offset 0 — a compiler artefact
  that shipped, and the only external script containing that byte pair.
- The **segment count is not declared**; a reader must stop on the first non-`GOTO`.
- ⏳ Segment 5 declares a single `0`; segment 6's second dword (**569**) and segment 1's ID byte
  (**`0x73`**) are unnamed.

**Continue:** [C18.2 — The mission and external tables](02-mission-and-external-tables.md)
