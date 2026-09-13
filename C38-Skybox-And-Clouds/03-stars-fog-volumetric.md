# C38.3 — Stars, fog and volumetric cloud counts

This page collects the fixed tables and slot counts the sky renderer uses. It is where the chapter earns its
✅s in the project's usual currency — a table that tiles to the byte, and a count read from a compare
instruction rather than a header.

## The star field — three `float[9]` tables that tile

The night sky's stars are placed from three parallel tables of nine floats each: a Y position, a Z position,
and a size. They sit consecutively in `.rdata`:

| Table | VA | Values (read from the exe) |
|---|---|---|
| Star Y | `0x8D55EC` | `0.00, 0.05, 0.13, 0.40, 0.70, 0.60, 0.27, 0.55, 0.75` |
| Star Z | `0x8D5610` | `0.00, 0.45, 0.90, 1.00, 0.85, 0.52, 0.48, 0.35, 0.20` |
| Star size | `0x8D5634` | `1.00, 1.40, 0.90, 1.00, 0.60, 1.50, 1.30, 1.00, 0.80` |

The arithmetic closes exactly:

```
0x8D5610 − 0x8D55EC = 0x24 = 36 = 9 × 4      (Y table is 9 floats, no residue)
0x8D5634 − 0x8D5610 = 0x24 = 36 = 9 × 4      (Z table is 9 floats, no residue)
```

Three nine-float tables packed nose to tail with no gap and no overlap — the same tiling proof a record
width gets everywhere else in this encyclopedia, applied to a constant array. `derive_sky.py` asserts both
the stride (`star_tables_tile`) and the exact float values (`star_table_values`), and separately that
`CClouds::Render` @`0x713950` references all three table bases (`render_reads_star_tables`, at
`0x713E4D`/`0x713E6D`/`0x713ECC`).

**Eleven stars over nine positions.** The draw loop runs `for (i = 0; i < 11; i++)` and indexes the tables
with `i % 9`, so the first two entries are reused — 11 star draws from a 9-entry table. The 9 and the 11 are
both control-flow constants in `Render`, not header values.

## Moving fog — 350 slots, from the compare itself

`CClouds` maintains a pool of "moving fog" puffs (the low drifting ground fog). The allocator
`MovingFog_GetFirstFreeSlot` at `0x713710` caps the pool with a literal:

```
0x71371D: cmp eax, 0x15E        ; 0x15E = 350
```

`derive_sky.py` asserts this immediate (`moving_fog_count_350`). The cap is read from the instruction that
enforces it, which is the strongest form this kind of count takes — there is no "the header says 350" to
mistrust, only the code that would reject slot 350.

## Volumetric clouds — 180 slots (🟡)

The volumetric-cloud subsystem (`cloudhigh`) has its own pool, capped at **180** per the `gta-reversed`
reconstruction (`MAX_VOLUMETRIC_CLOUDS = 180`). This pass did **not** re-find `180` (`0xB4`) as an immediate
in the small window of `VolumetricClouds_GetFirstFreeSlot` at `0x7135C0`, so — following the house rule that
a borrowed number is not a proven one — it is recorded as **🟡 reasoned**, not ✅. Closing it means widening
the disassembly to the volumetric allocator's real bound check (the constant may be folded into a different
comparison or a pointer-range test); it is listed in the open items below rather than asserted.

## Low clouds and the rainbow

Two more fixed tables round out `Render`:

- **Low cloud offsets** — a `CVector[]` of directional offsets the `cloud1` sprite is drawn around
  (`{1,0,0}, {0.7,−0.7,1}, …`). Count read from the reconstruction's array: **12** entries (🟡 — a
  count of a data table not re-closed as an exe immediate this pass).
- **Rainbow** — drawn as **6** coloured lines (`NUM_RAINBOW_LINES = 6`) after rain (🟡, same footing).

Both are structurally real and small; they are marked 🟡 only because the count itself was taken from the
array definition rather than an in-exe residue proof. They are honest candidates for promotion by a later
pass that reads the array extents out of `.rdata` the way the star tables were read here.

## Status of every count in this chapter

| Quantity | Value | Tier | Proof |
|---|---:|:--:|---|
| Star position table entries | 9 | ✅ | `0x24 / 4`, values match |
| Stars drawn per frame | 11 | ✅ | `Render` loop bound + `% 9` |
| Star tables tile | 3 × 36 B | ✅ | consecutive, no residue |
| Moving-fog slots | 350 | ✅ | `cmp eax, 0x15E` |
| Moon size default | 3 | ✅ | global `0x8D4B60` |
| Horizon bands | 6 | ✅ | `aPosZ[6]` in `RenderSkyPolys` |
| Volumetric-cloud slots | 180 | 🟡 | `gta-reversed`; not re-closed in exe |
| Low-cloud offsets | 12 | 🟡 | array count, not exe residue |
| Rainbow lines | 6 | 🟡 | array count, not exe residue |

## Open items

- ⏳ Re-close **volumetric-cloud = 180** from the exe (find the bound test in the `0x7135C0` allocator or the
  `VolumetricCloudsRender` path at `0x716380`).
- ⏳ The `SKYP_*` horizon-height float constants — read their values and confirm which band is sea-horizon.
- ⏳ The moon **phase-stepping** logic (how `MoonSize` @`0x8D4B60` advances with the day counter) lives
  outside `CClouds` and is untraced.

## Key takeaways

- The star field is the chapter's cleanest ✅: three `float[9]` tables tiling at 36 bytes with values read
  from the image, 11 draws over 9 positions.
- Slot counts split by proof quality — moving fog's **350** is taken from the compare instruction (✅),
  volumetric clouds' **180** is borrowed from `gta-reversed` and left 🟡 until re-closed in the exe.
- The remaining small tables (12 low-cloud offsets, 6 rainbow lines) are real but held 🟡, pending the same
  read-the-extent treatment the star tables received.

**Continue:** [back to the C38 hub →](C38-Skybox-And-Clouds.md) · or [C15 — Timecycle](../C15-Timecycle/C15-Timecycle.md), which supplies the sky colours.
