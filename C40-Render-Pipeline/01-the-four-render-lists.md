# C40.1 — The four visible-entity lists

The heart of the render pipeline is not a function; it is four arrays. Before San Andreas draws anything, it
sorts every candidate `CEntity*` in view into four fixed-size pointer lists. This page recovers those lists,
and proves their sizes not by trusting a header but by the way the four arrays **pack end-to-end in memory**.

## The four lists

| List | Base | Count | Element | Bytes | What it holds |
|---|---|---:|---|---:|---|
| `ms_aInVisibleEntityPtrs` | `0xB745D8` | **150** | `CEntity*` (4 B) | `0x258` | entities to process but not draw |
| `ms_aVisibleSuperLodPtrs` | `0xB74830` | **50** | `CEntity*` | `0xC8` | the far-distance super-LOD stand-ins |
| `ms_aVisibleLodPtrs` | `0xB748F8` | **1000** | `CEntity*` | `0xFA0` | LOD models for the mid/far city |
| `ms_aVisibleEntityPtrs` | `0xB75898` | **1000** | `CEntity*` | `0xFA0` | full-detail entities near the camera |

Immediately after the fourth array come the four counts, one per list:

```
ms_nNoOfVisibleSuperLods  @ 0xB76838
ms_nNoOfInVisibleEntities @ 0xB7683C
ms_nNoOfVisibleLods       @ 0xB76840
ms_nNoOfVisibleEntities   @ 0xB76844
```

`derive_renderer.py` confirms all eight globals (four bases + four counts) are referenced from `.text`, so
these are real, located memory the render code reads and writes — not speculative addresses.

## The tiling proof

The sizes are proven the project's way. Each array is `CEntity*[count]`, so it occupies `count × 4` bytes.
Lay them out from the first base and every array's end lands **exactly** on the next array's base, and the
last one lands **exactly** on the count block:

```
0xB745D8 + 150 × 4  (0x258)  = 0xB74830   = ms_aVisibleSuperLodPtrs   ✓
0xB74830 +  50 × 4  (0x0C8)  = 0xB748F8   = ms_aVisibleLodPtrs        ✓
0xB748F8 + 1000 × 4 (0xFA0)  = 0xB75898   = ms_aVisibleEntityPtrs     ✓
0xB75898 + 1000 × 4 (0xFA0)  = 0xB76838   = ms_nNoOfVisibleSuperLods  ✓
```

No gap, no overlap, from `0xB745D8` straight into the count variables at `0xB76838`. This is the whole size
proof: the base addresses are fixed points the code references, and **only** the counts 150 / 50 / 1000 /
1000 make them tile. A wrong count anywhere would push every later base off its known address. It is the same
standard as a record width that leaves no residue (C25), an audio pack that tiles its file (C20), or a slot
table that fills its pool (C31) — applied here to four consecutive arrays in the executable's uninitialised
data. `derive_renderer.py` asserts the chain (`render_lists_tile`).

## Why four lists, and why these sizes

The split is the LOD system. Near the camera, up to **1000** full-detail entities draw from
`ms_aVisibleEntityPtrs`. For the mid and far city, up to **1000** cheaper LOD models draw from
`ms_aVisibleLodPtrs`, and beyond them **50** super-LODs (`ms_aVisibleSuperLodPtrs`) stand in for whole
districts — which is why San Andreas can show a skyline from across the map without drawing every building.
The **150**-slot invisible list holds entities that must be *updated* (collision, script, streaming
bookkeeping) but are not on screen. The distances that decide which bucket an entity lands in are governed by
two scale globals — `ms_lodDistScale` (default **1.2**) and `ms_lowLodDistScale` (default **1.0**) at
`0x8CD800`/`0x8CD804` — the knobs the game's draw-distance setting turns.

The counts are also hard caps: if more than 1000 full-detail entities are in view, the surplus is not drawn.
This is the origin of the classic "entity limit" pop-out in dense scenes — a fixed array with no growth,
exactly the kind of engine limit C29's pools and C31's slot tables also revealed, recovered here as the size
that makes the arrays tile rather than as folklore.

## Key takeaways

- `CRenderer` sorts visible entities into four fixed `CEntity*` arrays: invisible **150**, super-LOD **50**,
  LOD **1000**, visible **1000**.
- Their sizes are proven by tiling — the four arrays pack from `0xB745D8` into the count block at `0xB76838`
  with no residue, and only those counts make the code-referenced bases line up.
- The four-way split is the LOD system (full detail / LOD / super-LOD) plus an off-screen update list; the
  1000-caps are hard limits, tuned by the two `lodDistScale` globals.

**Continue:** [C40.2 — The pass order →](02-the-pass-order.md)
