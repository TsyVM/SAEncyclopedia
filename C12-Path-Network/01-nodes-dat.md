# C12.1 — NODES\*.DAT: the Binary Node File

> **The one-sentence version:** a 28-byte header, a 28-byte record, and four independent structural
> checks that all hold on every one of 68,237 nodes — including a field that reproduces its own
> filename.

[← Chapter 12 hub](C12-Path-Network.md) · [Next: C12.2 — The text path source →](02-text-path-source.md)

**Confidence:** ✅ Verified (header, node record) / ⏳ (post-node block)

---

## 1. The header

```c
struct PathFileHeader {      // 28 bytes
    uint32_t numNodes;
    uint32_t numVehicleNodes;
    uint32_t numPedNodes;
    uint32_t numNaviNodes;
    uint32_t numLinks;
    uint32_t stalePointer;   // +0x14  a serialised runtime address — see §1.1
    uint32_t _zero;          // +0x18  zero in all 64 files
};
```

✅ *Verified* across all 64 files, with an arithmetic check that leaves no doubt:

```
numVehicleNodes + numPedNodes == numNodes
```

Game-wide: **30,587 + 37,650 = 68,237.** True in every individual file too. A header whose own fields
sum to its own total is self-validating — if the field order were misread, the sum would not close.

### 1.1 ⚠️ The sixth dword is not reserved

A first draft of this page claimed both trailing dwords were zero. **That was wrong**, and it was wrong
because it was checked against `NODES0.DAT` alone — the one file where it happens to be zero.

Across all 64:

| Field | Result |
|---|---|
| `+0x18` | zero in **64 / 64** ✅ |
| `+0x14` | **non-zero in 63 / 64** — only `NODES0.DAT` is zero |

The non-zero values run from `0x01C18B88` to `0x028142BC`, are **all 4-byte aligned**, and carry no
relationship to any count in the file.

🟡 *Reasoned:* that is a **serialised heap pointer** — a live `CPathNode*` (or similar) captured when
the tool wrote the file out of memory, and never cleared. The alignment, the magnitude, and the absence
of any correlation with the node counts all fit; nothing else plausibly produces 4-byte-aligned values
in a 12 MB band.

It is the same class of artefact as the first bytes of each node record, which C12.1 §3 leaves
undecoded for the same reason: **parts of these files are memory dumps, not designed formats.**

**For tooling:** ignore `+0x14`. A rebuilder should write zero, and a validator must not treat a
non-zero value as corruption.

**For method:** one file is not a population. `NODES0.DAT` is the smallest and the first alphabetically,
which is exactly why it was the one inspected — and exactly why it was unrepresentative. This is the
same failure as [C11.2 §5](../C11-IDE-And-IPL/02-ipl-placements.md)'s top-six interior table.

## 2. Finding the record stride without assuming it

The node array begins immediately at offset 28. Its stride was **not** assumed — it was measured.

Scanning `NODES0.DAT` for the byte pair `FE 7F` (`0x7FFE`):

```
occurrences: 209
positions:   34, 62, 90, 118, 146, 174, 202, 230, 258, 286, ...
spacings:    28, 28, 28, 28, 28, 28, 28, 28, 28, 28, 28, 28
```

**209 occurrences — exactly `numNodes`. Spacing 28, without variation.** The first is at 34, so the
constant sits at record offset `34 − 28 = +0x06`.

That single scan establishes the record start, the stride, and a constant field, with no prior
knowledge of the format. It is the same technique as the exact-size assertions used throughout this
encyclopedia: **let a repeating structure reveal itself.**

## 3. The record

```c
struct PathNode {            // 28 bytes
    int16_t  x;              // +0x00  world units × 8
    int16_t  y;              // +0x02
    int16_t  z;              // +0x04
    uint16_t marker;         // +0x06  always 0x7FFE
    uint16_t linkId;         // +0x08  first index into the link array
    uint16_t areaId;         // +0x0A  == the file's own number
    uint16_t nodeId;         // +0x0C  == this record's index
    uint8_t  pathWidth;      // +0x0E
    uint8_t  floodFill;      // +0x0F
    uint32_t flags;          // +0x10
    uint8_t  _tail[12];      // +0x14  ⏳ not decoded
};
```

### The four proofs

| Check | Result |
|---|---|
| `marker == 0x7FFE` | **68,237 / 68,237** |
| `areaId` == the numeral in the filename | **68,237 / 68,237** |
| `nodeId` == the record's sequential index | **68,237 / 68,237** |
| Positions inside −3000 … +3000 after `÷ 8` | **68,237 / 68,237** |

✅ All four, full population, zero exceptions.

**The `areaId` check is the one that settles the layout.** `NODES12.DAT` holds 2,215 records and every
one carries `12` at `+0x0A`. `NODES37.DAT` carries 37. A field cannot accidentally reproduce its own
filename across 64 files and 68,237 records — that *is* the area field, and its position fixes
everything around it.

`nodeId` being the record index is the second lock: it makes the array self-indexing and confirms the
stride independently of the `0x7FFE` scan.

### Coordinates

`int16` scaled by ⅛ gives a range of ±4096 units at 0.125-unit precision — comfortably covering the
6000-unit world with resolution to spare. Verified extents:

```
X  −2992.0 … 2946.2
Y  −2932.5 … 2853.6
Z    −46.0 … 2023.2
```

Z reaching **2023** is worth noting: the tallest object placement in
[C11.3](../C11-IDE-And-IPL/03-what-placements-prove.md) topped out at 1382, so the path network extends
well above the placed geometry — flight paths.

## 4. The area grid

64 files over a 6000-unit world is an **8 × 8 grid of 750 × 750-unit areas**.

This is a **third** spatial partition, alongside the two in
[C5.1](../C5-CWorld/01-the-two-grids.md):

| Grid | Cells | Cell size | Purpose |
|---|---:|---:|---|
| Sectors | 14,400 | 50 | static geometry |
| Repeat sectors | 256 | 50 (wrapping) | moving entities |
| **Path areas** | **64** | **750** | **navigation data files** |

🟡 *Reasoned:* the path grid is coarse because it partitions *files*, not queries — an area is a
streaming and authoring unit, not something indexed per frame. 750 units is roughly a district.

## 5. ⏳ The post-node block

After `28 + 28 × numNodes` bytes, each file continues. That tail holds the link array, the navi
(car-path) nodes, and more.

Attempting to solve its size as a linear combination across all 64 files:

```
remaining = a·numNavi + b·numLinks + c·numNodes
```

for small integer `a`, `b`, `c` — **no exact solution exists.**

| File | nodes | navi | links | remaining |
|---|---:|---:|---:|---:|
| `NODES0` | 209 | 215 | 428 | 7,578 |
| `NODES1` | 658 | 392 | 1,364 | 17,544 |
| `NODES12` | 2,215 | 633 | 4,786 | 48,294 |
| `NODES13` | 2,410 | 490 | 5,247 | 49,980 |

✅ *Verified negative result.* The tail is **not** three fixed-size arrays. It contains at least one
additional array, a variable-length structure, or a spatial index.

This is worth recording rather than omitting, for the same reason as
[C5.3 §5](../C5-CWorld/03-repeat-sectors.md)'s missing `1/375` constant: a ruled-out hypothesis narrows
the next pass. Whoever opens this next should look for a fourth count — possibly derived rather than
stored — instead of trying harder to fit three.

⏳ Also open: the 12 trailing bytes of each node record at `+0x14`, and the meaning of `flags`,
`pathWidth` and `floodFill` beyond their positions.

## 6. Reading it

```python
import struct

def path_nodes(path):
    d = open(path, 'rb').read()
    numNodes, numVeh, numPed, numNavi, numLinks = struct.unpack_from('<5I', d, 0)
    assert numVeh + numPed == numNodes          # holds in all 64 retail files
    for i in range(numNodes):
        o = 28 + 28 * i
        x, y, z, marker, linkId, areaId, nodeId = struct.unpack_from('<3hHHHH', d, o)
        assert marker == 0x7FFE                 # holds on all 68,237 retail nodes
        assert nodeId == i
        yield (x / 8.0, y / 8.0, z / 8.0), linkId, areaId, nodeId
```

Both assertions held on every retail record, so they are safe to keep in production — they turn a
misaligned read into an immediate failure rather than plausible garbage, which is the trap
[C10.3 §2](../C10-2dEffect/03-the-light-record.md) fell into.

---

### Key takeaways

- **28-byte header, 28-byte records.** The header self-validates: `vehicle + ped == total`, exact in
  every file and **30,587 + 37,650 = 68,237** game-wide.
- ⚠️ The dword at `+0x14` is **not reserved** — it is non-zero in **63 of 64** files and looks like a
  **stale serialised pointer**. A first draft called it zero after checking only `NODES0.DAT`.
- The stride was **measured, not assumed** — scanning for `0x7FFE` gave exactly `numNodes` hits at
  perfectly regular 28-byte spacing.
- Four checks hold on **68,237 / 68,237** records; **`areaId` reproducing the filename** is the one that
  fixes the layout.
- Coordinates are `int16 ÷ 8` — ±4096 range, 0.125-unit precision. **Zero nodes outside the world grid.**
- Z reaches **2023** against geometry's 1382 — the network extends above the placed world.
- 64 files = an **8 × 8 grid of 750-unit areas**, a third partition distinct from both C5 grids.
- ⏳ The post-node tail **does not fit three fixed arrays** — a verified negative result that redirects
  the next pass.

**Continue:** [C12.2 — The text path source](02-text-path-source.md)
