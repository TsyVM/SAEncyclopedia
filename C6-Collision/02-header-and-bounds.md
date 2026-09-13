# C6.2 — The Header, the Bounds, and a Broken File

> **The one-sentence version:** a 32-byte header followed by 40 bytes of bounds whose **field order
> changed between v1 and v2** — proved by arithmetic rather than assumed — and one retail file whose
> chunk chain is off by a byte.

[← C6.1 — The COL container & version dispatch](01-col-container-and-versions.md) ·
[Chapter 6 hub](C6-Collision.md) · [Next: C6.3 — CColStore & the binding →](03-colstore-and-binding.md)

**Confidence:** ✅ Verified

---

## 1. The header

```c
struct ColHeader {       // 32 bytes, all versions
    char     magic[4];   // +0x00
    uint32_t fileSize;   // +0x04  bytes after this field
    char     name[22];   // +0x08  NUL-padded
    uint16_t modelId;    // +0x1E
};                       // +0x20 -> bounds
```

✅ *Verified* against both the data and the loader. The loader's own arithmetic confirms the field
widths:

```
005384d8: mov  esi, ebp
005384da: mov  ecx, 5
005384df: lea  edi, [esp + 0x20]
005384e3: rep movsd                    ; 5 dwords = 20 bytes
005384e5: add  ebp, 0x16               ; advance 22
005384e8: movsw                        ; + 2 bytes  = 22 total -> name[22]
005384ea: movzx eax, word ptr [ebp]    ; modelId (u16)
005384ee: sub  edx, 0x18               ; 24 bytes consumed (22 + 2)
005384f1: add  ebp, 2
```

`rep movsd` × 5 plus one `movsw` is exactly 22 bytes — the name — and the `sub edx, 0x18` accounts for
the name plus the `u16` that follows. The 22-byte name is an odd width, and it is confirmed twice: by
the copy sequence and by the `add ebp, 0x16`.

## 2. The bounds — and the reordering

40 bytes follow the header. **The field order is not the same in v1 and v2+.**

### `COLL` (v1)

```c
struct ColBoundsV1 {
    float radius;      // +0x20
    float center[3];   // +0x24
    float min[3];      // +0x30
    float max[3];      // +0x3C
};
```

### `COL2` / `COL3` / `COL4`

```c
struct ColBoundsV2 {
    float min[3];      // +0x20
    float max[3];      // +0x2C
    float center[3];   // +0x38
    float radius;      // +0x44
};
```

✅ *Verified — and this is the part worth showing*, because the two layouts contain the same four
quantities and a reader that assumes the wrong order still parses without error.

The proof is arithmetic: **`center` must be the midpoint of `min` and `max`.** Testing that invariant
against the candidate orderings picks the right one unambiguously.

`peds.col` → `cop` (`COLL`), floats read from `+0x20`:

```
2.37, -0.01, -0.14, 0.20, -0.97, -2.36, -0.42, 0.95, 2.08, 0.82
```

Under the v1 ordering: `radius = 2.37`, `center = (-0.01, -0.14, 0.20)`,
`min = (-0.97, -2.36, -0.42)`, `max = (0.95, 2.08, 0.82)`.

```
midpoint(min, max) = (-0.01, -0.14, 0.20)   ==  center   ✓ exact
```

`levelmap_1.col` → `lib_street09` (`COL2`):

```
-59.534, -83.976, -18.911, 59.546, 83.989, 18.952, 0.006, 0.006, 0.020, 103.703
```

Under the v2 ordering: `min = (-59.534, -83.976, -18.911)`, `max = (59.546, 83.989, 18.952)`,
`center = (0.006, 0.006, 0.020)`, `radius = 103.703`.

```
midpoint(min, max) = (0.006, 0.006, 0.020)  ==  center   ✓ exact
```

Both hold to the printed precision, and the same test **fails** if the orderings are swapped. Three
`COL2` chunks were checked and all three match.

This is a technique worth naming: **when a struct's fields are internally consistent, the consistency
relation identifies the layout.** No disassembly was needed for the ordering — the data proved it.

## 3. `radius` is not the bounding-sphere radius of the AABB

For `lib_street09`, `|max − center|` = √(59.54² + 83.98² + 18.93²) ≈ **104.7**, against a stored radius
of **103.703**. Close, but not equal, and stored radius is *smaller*.

🟡 *Reasoned:* the radius is fitted to the actual geometry, not derived from the AABB corner — a tighter
sphere is possible whenever the mesh does not reach the box corners, which is almost always. So the two
bounds volumes are computed independently from the mesh, and a rebuilder that derives radius from the
AABB will produce a slightly conservative (larger) sphere. Harmless for correctness, mildly wasteful
for broad-phase rejection.

## 4. ⚠️ A genuinely broken file in retail data

Walking `models/coll/peds.col` by `off += 8 + fileSize` **fails**. It reads 8 models and then hits
garbage.

Scanning for magic bytes instead finds **30** chunks. Comparing the two:

| Chunk | Name | `fileSize` | Declared end | Next magic | Δ |
|---|---|---:|---:|---:|---:|
| 0 | `cop` | 732 | 740 | 740 | 0 |
| 1 | `playert` | 844 | 1592 | 1592 | 0 |
| … | … | … | … | … | 0 |
| 7 | `male01` | 732 | **6196** | **6195** | **−1** |
| 8 | `male02` | 720 | 6923 | 6923 | 0 |
| … | … | … | … | … | 0 |

✅ *Verified.* **`male01`'s `fileSize` is one byte too large.** Every other chunk in the file is exact.
The chain desynchronises at that point and never recovers, which is why a strict walker sees 8 of 30
models.

Consequences:

- **A strict COL walker silently truncates `peds.col` to 8 models.** It does not error — the byte at the
  desynchronised offset simply is not `COL`, so the loop exits normally.
- The game evidently copes, so the engine's own reader must tolerate this. ⏳ **Open:** how. It may
  resync on the magic, or it may not use `fileSize` for iteration at all. Reading the archive-level loop
  rather than the per-chunk parser would settle it, and would be worth doing before anyone writes a COL
  tool that claims retail compatibility.
- **Any tool that rebuilds `peds.col` will "fix" it**, changing the file even with no semantic edit.
- **This chapter fell into it.** The first draft of the [C6 hub](C6-Collision.md) reported 14 loose
  collision models; the real figure is 36. The census had been taken with a strict walker. The bug is
  quiet enough to catch the person documenting it.

Per-file, strict walk versus magic scan:

| File | Strict walk | Magic scan | Undercount |
|---|---:|---:|---:|
| `peds.col` | 8 | **30** | 22 |
| `vehicles.col` | 1 | 1 | 0 |
| `weapons.col` | 5 | 5 | 0 |

🟡 *Reasoned:* this is an authoring-tool bug frozen into the 2003 build of the file, not a deliberate
encoding. `peds.col` is the oldest collision file in the game ([C6.1 §4](01-col-container-and-versions.md)).

**Recommended reader behaviour:** follow `fileSize`, but if the next four bytes are not a valid magic,
scan forward (and back a few bytes) for one before giving up. That recovers all 30 models and remains
correct on well-formed files.

---

### Key takeaways

- **32-byte header** — `magic[4]`, `fileSize`, `name[22]`, `modelId` — with the odd 22-byte name width
  confirmed twice by the loader's copy sequence.
- **The bounds order changed between v1 and v2+**, and both layouts hold the same four quantities, so a
  wrong assumption parses cleanly and yields nonsense.
- Layout identified by the **`center == midpoint(min, max)` invariant**, exact in every sample —
  *consistency relations identify layouts without disassembly*.
- **`radius` is fitted to the mesh, not derived from the AABB** — always slightly tighter.
- ⚠️ **`peds.col` contains a real off-by-one**: `male01`'s `fileSize` overruns by 1 byte, so a strict
  walker sees 8 of 30 models and reports no error.
- Robust readers should **resync on the magic** rather than trusting `fileSize` blindly.

**Continue:** [C6.3 — CColStore & the model binding](03-colstore-and-binding.md)
