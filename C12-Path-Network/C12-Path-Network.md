# Chapter 12 — The Path Network

> **Goal of this chapter:** decode the largest authored data set in San Andreas — 68,237 navigation
> nodes across 64 binary files — and show that the 165,152 text rows from C11 are the *same network* in
> a different representation.

**Subsystem category:** World / AI data
**Depends on:** [C5 — CWorld & Spatial Partitioning](../C5-CWorld/C5-CWorld.md),
[C11 — IDE & IPL](../C11-IDE-And-IPL/C11-IDE-And-IPL.md)
**Ties:** [C4](../C4-Entities-And-Pools/C4-Entities-And-Pools.md), [C5](../C5-CWorld/C5-CWorld.md), [C6](../C6-Collision/C6-Collision.md), [C7](../C7-RenderWare-Stream/C7-RenderWare-Stream.md), [C9](../C9-Materials-And-Textures/C9-Materials-And-Textures.md), [C10](../C10-2dEffect/C10-2dEffect.md), [C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md)
**RE status:** Verified
**Confidence:** ✅ Verified over the full population
**Closes:** [C11.2 §1](../C11-IDE-And-IPL/02-ipl-placements.md) (the `path` section)

---

## Deep-dive pages

- [C12.1 — NODES\*.DAT: the binary node file](01-nodes-dat.md): the 28-byte record, decoded and proved
  four ways.
- [C12.2 — The text path source](02-text-path-source.md): 12,704 groups of exactly twelve.
- [C12.3 — Two representations of one network](03-two-representations.md): the 16× / 8× relationship,
  and a third confirmation of the sector grid.
- [C12.4 — Pathfinding and AI integration](04-pathfinding-and-ai-integration.md): vehicle vs. ped
  path-graph split; `CCarAI` routing through `CPathFind::FindShortestRoute`; police pursuit via path
  nodes; roadblock placement; path-node modding requirement for new roads.

---

## 12.1 Where the network actually lives

[C11.2](../C11-IDE-And-IPL/02-ipl-placements.md) found `path` at 165,152 rows — 93.1 % of all text IPL
content — and deferred it. The deferral was right, because the text is only half the story:

| Form | Location | Records |
|---|---|---:|
| **Binary** | `data/Paths/NODES*.DAT` — **64 files** | **68,237 nodes** |
| Text | `paths*.ipl` — 5 files, `path` sections | 12,704 groups × 12 = 152,448 rows |

Both describe the same navigation network. [C12.3](03-two-representations.md) proves it: the text
coordinates divided by 16 match the binary coordinates divided by 8 to within **0.07–0.21 units** at
every extreme — exactly the quantisation error of storing the same positions as `int16` at ⅛-unit
precision.

## 12.2 The binary file, verified four ways

Each `NODES*.DAT` is a 28-byte header followed by 28-byte node records
([C12.1](01-nodes-dat.md)). Four independent structural checks, all over the **full population of
68,237 nodes**:

| Check | Result |
|---|---|
| Constant marker `0x7FFE` at record `+0x06` | **68,237 / 68,237** |
| `areaId` at `+0x0A` equals the filename's index | **68,237 / 68,237** |
| `nodeId` at `+0x0C` equals the record's sequential index | **68,237 / 68,237** |
| Header `numVehNodes + numPedNodes == numNodes` | **exact in every file** |

✅ The `areaId` check is the decisive one. `NODES12.DAT` contains 2,215 records and every single one
carries `12` in that field. A field that reproduces the filename across 64 files and 68,237 records is
not a coincidental alignment — it is that field.

**30,587 vehicle nodes + 37,650 pedestrian nodes = 68,237.** The header's own totals add up to its own
node count, in every file.

## 12.3 The world grid, confirmed a third time

Positions are `int16` at `+0x00`, `+0x02`, `+0x04`, scaled by ⅛:

| Axis | Range |
|---|---|
| X | **−2992.0 … 2946.2** |
| Y | **−2932.5 … 2853.6** |
| Z | −46.0 … 2023.2 |
| **Nodes outside −3000 … +3000** | **0 of 68,237** |

✅ This is the **third** independent confirmation of the sector grid derived in
[C5.2](../C5-CWorld/02-sector-index-arithmetic.md) from three floating-point instructions:

| Source | Evidence | Result |
|---|---|---|
| [C5.2](../C5-CWorld/02-sector-index-arithmetic.md) | `× 0.02 + 60.0` in compiled code | −3000 … +3000 |
| [C11.3](../C11-IDE-And-IPL/03-what-placements-prove.md) | 36,569 object placements | 0 outside |
| **C12** | **68,237 navigation nodes** | **0 outside** |

Three sources — code, art placement, AI data — none derived from the others, all agreeing.

Note also the **64 files over a 6000-unit world**: an 8 × 8 grid of **750 × 750-unit areas**. A third
spatial partition, distinct from both grids in [C5.1](../C5-CWorld/01-the-two-grids.md), and coarser
than either.

## 12.4 Scale

| Quantity | Value |
|---|---:|
| Navigation nodes | **68,237** |
| Node links | **143,622** |
| Navi (car-path) nodes | 31,466 |
| Object placements ([C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md)) | 45,884 |
| Collision models ([C6](../C6-Collision/C6-Collision.md)) | 10,155 |

**There are more navigation nodes than object placements**, and more than three times as many links as
collision models. The single largest authored artefact in San Andreas is not its geometry — it is the
graph the AI drives on.

That reframes [C10.2 §2](../C10-2dEffect/02-effect-type-census.md)'s finding that 85.7 % of `2dEffect`
records are cover points: the world is annotated for the AI far more densely than it is decorated for
the player.

## 12.5 What remains

⏳ **Open:** the block after the node array. Each file carries `numLinks` link records, `numNavi`
car-path nodes, and further arrays in the remaining bytes — but a linear fit of
`remaining = a·navi + b·links + c·nodes` across all 64 files found **no exact solution** for small
integer coefficients, so the tail contains more than three fixed-size arrays.

That negative result is recorded rather than papered over: it rules out the simplest layout and tells
the next pass to look for a variable-length or index structure in the tail
([C12.1 §5](01-nodes-dat.md)).

---

### Key takeaways

- The navigation network lives in **64 `NODES*.DAT` files: 68,237 nodes, 143,622 links** — the text
  `path` rows are the same data in another form.
- The 28-byte record is proved **four ways over the full population**, most decisively by `areaId`
  reproducing the filename index on **68,237 / 68,237** records.
- Header arithmetic checks out: **30,587 vehicle + 37,650 ped = 68,237 nodes**, in every file.
- ✅ **Zero of 68,237 nodes fall outside −3000 … +3000** — the **third** independent confirmation of
  [C5.2](../C5-CWorld/02-sector-index-arithmetic.md), after code and object placements.
- The 64 files form an **8 × 8 grid of 750-unit areas** — a third spatial partition, coarser than either
  grid in C5.
- **More navigation nodes than object placements.** The largest authored artefact in the game is the AI
  graph, not the geometry.
- ⏳ The post-node block resists a three-array linear fit — a recorded negative result.

**Next:** [C12.1 — NODES\*.DAT: the binary node file](01-nodes-dat.md)

## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Loaded / consumed by:** [C4](../C4-Entities-And-Pools/C4-Entities-And-Pools.md), [C5](../C5-CWorld/C5-CWorld.md), [C6](../C6-Collision/C6-Collision.md), [C7](../C7-RenderWare-Stream/C7-RenderWare-Stream.md), [C9](../C9-Materials-And-Textures/C9-Materials-And-Textures.md), [C10](../C10-2dEffect/C10-2dEffect.md)
- **Known bugs / gotchas:** dead gridref reader (C22) shows cut path features; broken nodes strand AI.
- **Modding:** NODES*.DAT path graph is edited for custom traffic/AI routes.
- **Performance:** path queries are graph walks; cached per area.
