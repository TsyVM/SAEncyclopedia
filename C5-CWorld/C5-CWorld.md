# Chapter 5 — CWorld & Spatial Partitioning

> **Goal of this chapter:** decode how San Andreas divides 36 km² of map into a grid it can query in
> constant time — two superimposed grids at different resolutions, the exact arithmetic that maps a
> world coordinate to a cell, and why the pointer-node pool is the size it is.

**Subsystem category:** World
**Depends on:** [C4 — Entities & the Pool Allocator](../C4-Entities-And-Pools/C4-Entities-And-Pools.md)
**Ties:** [C0](../C0-Binary-Identity/C0-Binary-Identity.md), [X1](../X1-SDK-Cross-Reference/X1-SDK-Cross-Reference.md), [C1](../C1-Streaming/C1-Streaming.md), [C2](../C2-CStreaming/C2-CStreaming.md), [C4](../C4-Entities-And-Pools/C4-Entities-And-Pools.md), [C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md)
**RE status:** Documented
**Confidence:** ✅ Verified (grid geometry) / ⏳ (array bases)

---

## Deep-dive pages

- [C5.1 — The two grids](01-the-two-grids.md): why the world is partitioned twice.
- [C5.2 — Sector index arithmetic](02-sector-index-arithmetic.md): the formula, verified three ways.
- [C5.3 — Repeat sectors](03-repeat-sectors.md): the mask, and a useful negative result.
- [C5.4 — The PtrNode coupling](04-ptrnode-coupling.md): why 70,000, and why limit adjusters break.
- [C5.5 — Open: the sector arrays](05-open-the-sector-arrays.md): the boundary as it stood.
- [C5.6 — The sector arrays, closed](06-the-sector-arrays-closed.md): **both bases found**, and a
  correction to C5.1/C5.3 — the repeat grid is a wrapping hash, not a coarse partition.

> ⚠️ Read [C5.6](06-the-sector-arrays-closed.md) before relying on C5.1 §1 or C5.3 §2.

---

## 5.1 Two grids, not one

San Andreas partitions the world twice, at two resolutions, for two different jobs.

| Grid | Base | Cells | Record | Covers | Purpose |
|---|---|---|---|---|---|
| **Sectors** | `0x00B7D0B8` | 120 × 120 = 14,400 | 8 B — 2 lists | −3000 … +3000, clamped | static geometry |
| **Repeat sectors** | `0x00B992B8` | 16 × 16 = 256 | 12 B — 3 lists | **wraps every 800 units** | moving entities |

✅ Verified, including both bases — the fine array ends exactly where the repeat array begins
([C5.6](06-the-sector-arrays-closed.md)). The 2-list / 3-list record widths confirm the static/dynamic
split.

The repeat grid **does not** partition the map into coarse cells; it hashes position by masking the
sector index, so 56.25 world cells share each of the 256 buckets. C5.1 §1 and C5.3 §2 originally said
otherwise and are corrected in [C5.6 §4](06-the-sector-arrays-closed.md).

## 5.2 The sector index

The conversion from a world X or Y to a sector index is three instructions:

```
004090c2: fld   dword ptr [esp + 8]
004090c6: fmul  dword ptr [0x858b38]     ; × 0.02        (= 1/50)
004090d7: fadd  dword ptr [0x858b34]     ; + 60.0
004090dd: fstp  qword ptr [esp]
004090e0: call  0x8219f0                 ; floor
```

✅ *Verified*, including the constants read out of `.rdata`:

| Address | Value |
|---|---|
| `0x00858B38` | `0.019999999552965164` (float `1/50`) |
| `0x00858B34` | `60.0` |
| `0x00858B40` | `50.0` |

```
sectorIndex = floor(coord × 0.02 + 60)
```

Checking the endpoints: `−3000 × 0.02 + 60 = 0` and `+3000 × 0.02 + 60 = 120`. **The grid is exactly
120 cells of 50 units spanning −3000 … +3000** — the constants define the world extent, they are not
derived from it.

The row stride is confirmed independently by the multiply used to flatten the 2-D index:

```
imul reg, reg, 0x78          ; × 120
```

found in `0x004090A0`, `0x00409210`, `0x0041A820`, `0x00546670`, `0x0054BA60`. ✅

## 5.3 The repeat-sector index

The coarse grid uses a mask rather than a multiply:

```
00409116: and   ebp, 0xf                 ; index & 15
```

**16 × 16.** ✅ And `6000 / 16 = 375` — which appears in `.rdata` as a literal float:

```
0x0086555C = 375.0
```

The cell size exists as a constant *and* falls out of the world extent divided by the mask. Two
independent derivations agreeing is what promotes this from "plausible" to verified.

🟡 *Reasoned:* the split of duties — fine grid for static geometry, coarse grid for moving entities —
is the standard reading and is consistent with the coarse grid being 256 cells (cheap to sweep every
frame) against the fine grid's 14,400. This pass did not read the code that populates each, so the
*geometry* is ✅ and the *purpose* is 🟡.

## 5.4 Why `PtrNode Single` is 70,000

[C4 §4.3](../C4-Entities-And-Pools/C4-Entities-And-Pools.md) noted that the pointer-node pool is sized
for object-sector *memberships*, not objects. This chapter supplies the arithmetic behind that claim.

An entity is threaded into the list of **every sector its bounding box overlaps**. With 50-unit cells,
a large building spans several; a long road segment spans many. Against 13,000 buildings and 2,500
dummies, an average of roughly four memberships each already accounts for the bulk of 70,000 nodes.

This is why raising the `Buildings` limit without raising `PtrNode Single` fails in a way that looks
random: the building pool has room, but the world cannot link the new buildings into their sectors.
**The two limits are coupled through the grid, and the coupling factor is the average sector span of
your geometry** — which is why limit adjusters raise both, and why the correct ratio is
content-dependent rather than fixed.

## 5.5 What was not established

⏳ **Open — and this is the chapter's honest boundary:**

- **The sector array base addresses.** `0x00B7CD98` and `0x00B7D0B8` are the leading candidates by
  reference count in the `0xB7xxxx` band, but neither was confirmed as `ms_aSectors` /
  `ms_aRepeatSectors` by reading an indexing instruction against it. **Stated as candidates, not
  asserted.**
- **The `CSector` record layout** — how many pointer lists each cell holds and in what order.
- **The list-insertion path** — the function that threads an entity into its overlapping cells.

Everything in §5.1–5.3 is verified geometry recovered from the index arithmetic. The *containers* that
geometry indexes are the next pass. Publishing the grid without the arrays is the right split: the
arithmetic is certain and immediately useful, and pretending to know the bases would poison a chapter
that is otherwise solid.

## 5.6 Dependencies

```
CWorld
  ├── depends on: CEntity + the pool allocator      [C4]
  │               CPtrNode pools (70,000 / 3,200)   [C4]
  └── used by:    CStreaming (what to load near the camera)   [C2]
                  collision, rendering, AI queries
```

Note the cycle: `CStreaming` decides what to load based on where the camera is in the world, and
`CWorld` holds entities whose models `CStreaming` loaded. The two are mutually dependent at runtime and
are only separable on paper because the streaming side deals in IDs and the world side deals in
positions.

---

### Key takeaways

- The world is partitioned **twice**: 120 × 120 sectors of 50 units, and 16 × 16 repeat sectors of 375
  units, both spanning **−3000 … +3000**.
- `sectorIndex = floor(coord × 0.02 + 60)` — verified with the constants at `0x00858B38` and
  `0x00858B34`; the endpoints land exactly on 0 and 120.
- The repeat grid uses `and 0xF`, and **375.0 exists as a literal** — two independent derivations of the
  same cell size.
- Row stride `× 120` is confirmed by `imul reg, reg, 0x78` in five separate functions.
- **`PtrNode Single` = 70,000 is a consequence of the grid**: entities are linked into every sector
  they overlap, which is why building and pointer-node limits must be raised together.
- **The sector array bases are not established** — two candidates are recorded as candidates. The grid
  arithmetic is certain; the containers are the next pass.

**Next:** `C6 — Collision & the COL Model` — the first consumer of both the store partition and the
sector grid.

## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Loaded / consumed by:** [C0](../C0-Binary-Identity/C0-Binary-Identity.md), [C1](../C1-Streaming/C1-Streaming.md), [C2](../C2-CStreaming/C2-CStreaming.md), [C4](../C4-Entities-And-Pools/C4-Entities-And-Pools.md), [C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md), [X1](../X1-SDK-Cross-Reference/X1-SDK-Cross-Reference.md)
- **Known bugs / gotchas:** CWorld sector edge cases with fast-moving entities.
- **Modding:** the sector grid is where custom collision/entities register.
- **Performance:** spatial partition keeps per-frame queries bounded.
