# C6.1 — The COL Container & Version Dispatch

> **The one-sentence version:** a COL archive is a bare concatenation of self-sizing chunks with no
> directory, and the loader recognises four versions by FourCC — one of which the shipped game never
> uses.

[← Chapter 6 hub](C6-Collision.md) · [Next: C6.2 — The header and bounds →](02-header-and-bounds.md)

**Confidence:** ✅ Verified

---

## 1. The container

```c
struct ColChunk {
    char     magic[4];   // 'COLL' | 'COL2' | 'COL3' | 'COL4'
    uint32_t fileSize;   // bytes following this field
    // ... fileSize bytes of header + payload
};                       // next chunk begins at (this + 8 + fileSize)
```

There is **no directory and no count**. An archive is walked by following `fileSize` from chunk to
chunk until the buffer runs out — the same "self-sizing chunk" model as EAGL containers, and the
opposite of the IMG format's explicit directory ([C1.1](../C1-Streaming/01-img-ver2-archive-model.md)).

The consequence is that a COL archive **cannot be indexed without walking it**, and a single bad
`fileSize` desynchronises everything after it ([C6.2 §4](02-header-and-bounds.md) — which is not
hypothetical).

## 2. The version dispatch

At `0x00538480`, and it is a compare chain rather than a table:

```
00538480: cmp  eax, 0x344c4f43        ; 'COL4'
00538485: ...
0053848d: ja   0x5384c1               ; above -> could only be 'COLL'
0053848f: je   0x5384b7               ; equal -> version 3
00538491: cmp  eax, 0x324c4f43        ; 'COL2'
00538496: je   0x5384ad               ;      -> version 1
00538498: cmp  eax, 0x334c4f43        ; 'COL3'
0053849d: jne  0x53865d               ;      -> reject
005384a3: mov  dword ptr [esp+0x10], 2
005384ab: jmp  0x5384d4
005384ad: mov  dword ptr [esp+0x10], 1
005384b7: mov  dword ptr [esp+0x10], 3
005384c1: cmp  eax, 0x4c4c4f43        ; 'COLL'
005384c6: jne  0x53865d               ;      -> reject
005384cc: mov  dword ptr [esp+0x10], 0
```

✅ *Verified.*

| FourCC | Little-endian | Internal version |
|---|---|---:|
| `COLL` | `0x4C4C4F43` | **0** |
| `COL2` | `0x324C4F43` | **1** |
| `COL3` | `0x334C4F43` | **2** |
| `COL4` | `0x344C4F43` | **3** |

Note the off-by-one that trips people up: the file says `COL2`, the engine calls it **version 1**. The
internal numbering is zero-based over the FourCC sequence, so `COL3` — the version 97 % of retail
collision uses — is internally `2`.

### The `ja` is doing real work

The chain tests `COL4` first and branches on **above**, not just equal. Because `'COLL'` (`0x4C…`) is
numerically greater than `'COL4'` (`0x34…`), a single unsigned comparison separates the oldest format
from the three newer ones before any equality test. It is a two-instruction way to split a four-way
dispatch, and it is why `COLL` is handled in the tail rather than in sequence.

Anything that matches none of the four jumps to `0x0053865D` — **rejection, not best-effort**. A
corrupt or unknown magic does not fall through to a default parser.

## 3. `COL4` is supported and unused

The loader accepts `COL4` and assigns it internal version 3. Surveying every collision chunk shipped
with the game:

| Version | Count |
|---|---:|
| `COLL` | 36 |
| `COL2` | 885 |
| `COL3` | 9,270 |
| **`COL4`** | **0** |

✅ *Verified* across all 251 streamed archives and the three loose files — the loose files counted by
**magic scan**, not by following `fileSize`, for the reason in
[C6.2 §4](02-header-and-bounds.md). A strict walk reports 14 and does not error.

🟡 *Reasoned:* `COL4` is a format the engine was prepared for and the shipped content never adopted —
plausibly a late-development addition, or a format used by a tool chain whose output did not reach
retail. What it changes relative to `COL3` cannot be determined from this build's data, because there
is nothing to read.

⏳ **Open:** what the `COL4` code path does differently. The dispatch target is known
(`version = 3`), so the question is answerable by reading the branches that test that variable — a
tractable next step, and the only route available given the absence of sample data.

## 4. Where the archives live

| Location | Archives | Form |
|---|---:|---|
| `models/gta3.img` | 216 | streamed |
| `models/gta_int.img` | 35 | streamed |
| `models/coll/` | 3 | loose on disk |

The three loose files — `peds.col` (30 models), `vehicles.col` (1), `weapons.col` (5) — are the only
`COLL` v1 content in the game and hold **36** models between them. 🟡 *Reasoned:* they are loaded once at startup rather than
streamed, which is consistent with their subject matter (pedestrian, vehicle and weapon collision is
always needed) and with their being the only files never converted to a newer version.

Their file dates support the reading: `peds.col` is stamped 2003, a year before the streamed content.

---

### Key takeaways

- A COL archive is a **directory-less concatenation of self-sizing chunks**: `magic[4]`, `fileSize`,
  payload, repeat.
- **Four versions dispatched by FourCC** at `0x00538480`; unknown magic is **rejected**, not
  best-effort parsed.
- The internal numbering is **zero-based**: `COL2` → 1, `COL3` → 2, `COL4` → 3.
- The chain uses a single **unsigned `ja`** to split `COLL` from the rest, because `'COLL'` sorts above
  `'COL4'`.
- **`COL4` is supported and completely unused** — 0 chunks in retail; what it adds is open and can only
  be answered from code.
- The three **loose `COLL` files** hold all **36** v1 models and predate the streamed content by a year.
  Counting them requires a **magic scan**; a strict `fileSize` walk silently returns 14.

**Continue:** [C6.2 — The header, the bounds, and a broken file](02-header-and-bounds.md)
