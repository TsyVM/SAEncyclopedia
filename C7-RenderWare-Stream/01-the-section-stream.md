# C7.1 — The Section Stream

> **The one-sentence version:** twelve bytes of header, a size that excludes itself, and unbounded
> recursion — the entire RenderWare container in one rule, plus the eleven retail files that break the
> assumption everyone makes about it.

[← Chapter 7 hub](C7-RenderWare-Stream.md) · [Next: C7.2 — Version encoding →](02-version-encoding.md)

**Confidence:** ✅ Verified

---

## 1. The header

```c
struct RwSectionHeader {
    uint32_t type;       // +0x00  section type ID
    uint32_t size;       // +0x04  payload bytes, EXCLUDING this 12-byte header
    uint32_t libraryID;  // +0x08  packed version + build  (C7.2)
};                       // next sibling at (this + 12 + size)
```

✅ *Verified.* Little-endian throughout.

Two properties make the format navigable without knowing any section's meaning:

- **Every section is self-sizing.** You can skip a section you do not understand.
- **Every section carries its own version.** Not just the file — each nested section repeats the
  library ID ([C7.2](02-version-encoding.md)).

The consequence is that a RenderWare reader can be written as a generic tree walker first and a
semantic parser second, which is exactly how the census in this chapter was produced.

## 2. Recursion

Container sections hold child sections; leaf sections hold raw data. Nothing in the header says which a
section is — that is knowledge about the type ID.

A verified walk of a typical clump, by depth:

```
depth 0   Clump (0x10)
depth 1     Struct (0x01)          — counts
depth 1     FrameList (0x0E)
depth 2       Struct               — frame matrices
depth 2       Extension (0x03)
depth 3         Frame / HAnimPLG   — plugin data
depth 1     GeometryList (0x1A)
depth 2       Struct
depth 2       Geometry (0x0F)
depth 3         Struct
depth 3         MaterialList (0x08)
depth 3         Extension (0x03)
depth 1     Atomic (0x14)  × N
depth 1     Extension (0x03)
```

✅ *Verified* by walking 60 randomly sampled DFFs to depth 3. Observed counts over that sample:

| Depth | Section | Occurrences |
|---:|---|---:|
| 0 | Clump | 60 |
| 1 | Struct / FrameList / GeometryList / Extension | 60 each |
| 1 | Atomic | 61 |
| 2 | Struct | 181 |
| 2 | Extension | 155 |
| 2 | Geometry | 61 |
| 3 | `2dEffect` (`0x253F2FE`) | 92 |
| 3 | `0x11E` | 34 |
| 3 | `HAnimPLG` (`0x1F`) | 1 |

Note **61 Atomics across 60 clumps** — nearly all models are a single atomic, with one sample carrying
two. And **92 `2dEffect` sections in 60 files** — those are the placed light, particle and effect
markers attached to world geometry, which is where map lighting comes from.

🟡 The `0x11E` sections (34 occurrences) were not identified. Recorded rather than guessed.

## 3. ⚠️ Eleven files have two roots

The universal assumption — *a RenderWare file has one root section* — is false in retail data.

Checking every `.dff`/`.txd` in `gta3.img` for whether `12 + rootSize` accounts for the payload:

| Result | Count |
|---|---:|
| Single root, fits | 15,694 |
| **Does not fit** | **11** |

All eleven are `.dff` files whose **first** root is type `0x2B` (`UVAnimDict`), followed by a second
root section of type `0x10` (`Clump`):

| File | Root 1 (UVAnimDict) | Root 2 (Clump) | Consumed / slot |
|---|---:|---:|---|
| `sfw_waterfall.dff` | 788 | 2,173 | 2,985 / 4,096 |
| `ltseld01_lawn.dff` | 1,332 | 5,009 | 6,365 / 8,192 |
| `ltsrec01_lawn.dff` | 1,332 | 3,361 | 4,717 / 6,144 |
| `ltsreg01_lawn.dff` | 2,648 | 3,897 | 6,569 / 8,192 |
| `vgsn_scrollsgn01.dff` | 724 | 3,024 | 3,772 / 4,096 |
| `vgsn_frntneon_nt.dff` | 13,652 | 18,560 | 32,236 / 32,768 |

✅ *Verified.* After both roots the remaining bytes are sector padding
([C1.1 §4.1](../C1-Streaming/01-img-ver2-archive-model.md)).

The subject matter is consistent and explains the design: a **waterfall**, three **lawns**, a
**scrolling sign**, **neon**. All are surfaces whose texture coordinates animate, so the file carries a
UV-animation dictionary that must be registered *before* the clump that references it. Order matters,
which is why it is a second root rather than an extension inside the clump.

**Reader requirement:** loop over root sections until the buffer is consumed; do not read one and stop.
A single-root reader does not error on these files — it returns the UVAnimDict and reports success, and
the model silently never appears.

## 4. A generic walker

```python
import struct

def sections(buf, start=0, end=None):
    """Yield (type, size, libraryID, payload_offset) for siblings in [start, end)."""
    end = len(buf) if end is None else end
    off = start
    while off + 12 <= end:
        typ, size, lib = struct.unpack_from('<III', buf, off)
        if size == 0 or off + 12 + size > end:
            break                      # padding or truncation -> stop
        yield typ, size, lib, off + 12
        off += 12 + size

def roots(buf):
    return list(sections(buf))         # NOT next(sections(buf)) — see §3
```

The `off + 12 + size > end` guard is what makes this safe against the sector padding every IMG-extracted
file carries: padding decodes as a nonsense header with an oversized `size`, and the walk stops cleanly
instead of running away.

## 5. Type IDs observed

Confirmed present in retail data by the census:

| ID | Name |
|---|---|
| `0x01` | Struct |
| `0x03` | Extension |
| `0x08` | MaterialList |
| `0x0E` | FrameList |
| `0x0F` | Geometry |
| `0x10` | **Clump** — DFF root |
| `0x14` | Atomic |
| `0x15` | TextureNative |
| `0x16` | **TextureDictionary** — TXD root |
| `0x1A` | GeometryList |
| `0x1F` | HAnimPLG |
| `0x2B` | **UVAnimDict** — second DFF root |
| `0x253F2FE` | 2dEffect |
| `0x11E` | ⏳ unidentified |

Plugin IDs in the `0x253F2xx` range are RenderWare's vendor-extension space; `2dEffect` is the one that
appears in bulk.

---

### Key takeaways

- **12-byte header** — `type`, `size` (excluding the header), `libraryID` — and every section is
  self-sizing and self-versioning.
- The format is walkable **without understanding any type**, which is how this chapter's census was
  built.
- Clump structure verified to depth 3: `Struct`, `FrameList`, `GeometryList`, `Atomic`×N, `Extension`.
- **61 atomics across 60 clumps** — models are overwhelmingly single-atomic; **92 `2dEffect` sections**
  carry world lighting and effect placement.
- ⚠️ **Eleven retail DFFs have two root sections** (UVAnimDict then Clump) — animated-UV surfaces. A
  single-root reader drops them **silently**.
- Guard the walk with `off + 12 + size > end` so sector padding terminates it cleanly.

**Continue:** [C7.2 — Version encoding & the RW 3.6 confirmation](02-version-encoding.md)
