# C48.3 — Extensions and the binary grid

The last four files divide into three that are *alternate copies* of formats already decoded — and one that
is the only **binary** file in the uncovered set. This page ties the three to their home chapters and opens
the binary one.

## `timecycp.dat` — the alternate timecycle (→ C15)

`timecycp.dat` is byte-for-byte the same **format** as `timecyc.dat` ([C15](../C15-Timecycle/C15-Timecycle.md)):
the identical column header (`Amb  Amb_Obj  Dir  Sky top  Sky bot  SunCore  SunCorona  SunSz …`), the same
24-column rows, weathers × hours. `derive_datasweep.py` confirms the shared header and the 437-line body. The
`p` suffix marks it as the **PS2 / alternate** timecycle — a second full set of atmospheric colour tables, a
lower-fidelity or platform-specific variant the engine can select instead of the main `timecyc.dat`. Nothing
about the format needs re-deriving; it *is* [C15](../C15-Timecycle/C15-Timecycle.md), with different numbers.
Its presence is the same "capability with an alternate data set" pattern C19 (five languages) and C23 (mask
atlases) showed — the engine supports two timecycles, and both ship.

## `water1.dat` — the alternate water (→ C16)

`water1.dat` is the water-surface geometry in the same grammar as `water.dat`
([C16](../C16-Popcycle-And-Water/C16-Popcycle-And-Water.md)): a `processed` header line, then **267** quad
rows, each four vertices of `x y z  <params>`:

```
processed
-2992.0 1184.0 0.0 1.0 1.0 0.39062 0.39062   -2832.0 1184.0 ...   (4 vertices per quad)
```

`derive_datasweep.py` confirms the header and 267 quads. Each quad is a patch of water surface with per-vertex
flow/wave parameters; `water1.dat` is the alternate/high-detail water set (the `1` suffix), paired with
`water.dat` the way `timecycp` pairs with `timecyc`. It ties directly to
[C16](../C16-Popcycle-And-Water/C16-Popcycle-And-Water.md)'s water documentation — same format, second set.

## `animgrp.dat` — animation groups (→ C17)

`animgrp.dat` maps a ped's **animation group** to the specific clips it uses. **42** groups, each a header
line and a list of animation names:

```
man, ped, walkcycle, 6
  walk_civi
  run_civi
  sprint_panic
  idle_stance
  ...
```

A group like `man` binds the generic locomotion slots (walk, run, sprint, idle) to concrete clips in the
[C17](../C17-IFP-Animation/C17-IFP-Animation.md) `ped.ifp` animation archive. Different ped types (man, woman,
old, fat, gang) reference different groups, which is how a fat ped walks differently from a gang member — same
skeleton and slots ([C44.2](../C44-Shaders/02-register-map-and-transform.md) skins them), different clips
bound here. It is the index between the ped tables ([C26](../C26-Ped-Tables/C26-Ped-Tables.md)) and the
animation data ([C17](../C17-IFP-Animation/C17-IFP-Animation.md)): 42 named movement styles.

## `polydensity.dat` — the one binary file

`polydensity.dat` is the outlier: the only **binary** file in the uncovered set. It is `288,020` bytes of
little-endian `uint32` values — **72,005** cells — with no text header. Reading the first values
(`0x48, 0x48, 0x48, 0x12E, 0xE6, 0xE6 …`) shows small integers, repeated in runs. This is a **map polygon-density
grid**: a per-cell count of how much geometry sits in each region of the world, used by the renderer/streamer
to budget detail — where the world is dense (downtown), the engine knows to be more conservative with draw
distance and LOD ([C40](../C40-Render-Pipeline/C40-Render-Pipeline.md)'s LOD lists). The exact cell→world
mapping (grid dimensions, cell size) is **⏳ not derived here** — 72,005 does not factor into an obvious
square, so the grid likely has a header or non-square dimensions this sweep did not resolve. What is proven
(`derive_datasweep.py`: `polydensity_binary_u32`) is the container: a flat `uint32[72005]` density array, the
lone binary member of `data/`.

## polydensity.dat is unreferenced in retail — a negative result

The grid geometry turns out to be unanswerable for a concrete reason: **nothing in the retail game loads
`polydensity.dat`.** `derive_openitems.py` (`polydensity_unreferenced`) confirms the string `polydensity`
appears **zero** times in `gta_sa.exe`, and the file is **not listed** in any boot manifest
(`gta.dat`/`default.dat`/`gta_quick.dat`, [C48.1](01-load-and-boot-config.md)). There is no loader, no
`sscanf`, no path reference. So it is a **dead/tool leftover** — a build-pipeline artifact shipped on the
disc but never read by the game, the same category as `animviewer.dat`
([C48.1](01-load-and-boot-config.md)) and the cut `NAVIG.ZON` ([C22](../C22-Map-Zones/C22-Map-Zones.md)).
The container is still what it is — a flat `uint32[72005]` array of density values — but its **cell→world
mapping is indeterminate because retail never establishes one**. This is a genuine closure of the open item:
the grid geometry cannot be derived because the code that would define it does not exist in the shipped
binary. (It was presumably consumed by Rockstar's map/streaming compiler, not the game.)

## Open items
- ⏳ Which conditions select `timecycp.dat` over `timecyc.dat`, and `water1.dat` over `water.dat`.

## Key takeaways

- `timecycp.dat`, `water1.dat` and `animgrp.dat` are **alternate/companion data** in formats already decoded —
  they extend [C15](../C15-Timecycle/C15-Timecycle.md), [C16](../C16-Popcycle-And-Water/C16-Popcycle-And-Water.md)
  and [C17](../C17-IFP-Animation/C17-IFP-Animation.md) respectively (a second timecycle, a second water set,
  42 animation groups).
- `polydensity.dat` is the **only binary** file — a flat `uint32[72005]` map polygon-density grid feeding
  LOD/streaming; its container is proven, its grid geometry is left ⏳ open.
- With these, the **`data/` folder is fully swept** — every file has a chapter or a home.

**Continue:** [back to the C48 hub →](C48-Data-Folder-Sweep.md) · then the SA:MP arc (next in the sweep).
