# C5.3 — Repeat Sectors

> **The one-sentence version:** the coarse grid indexes with a bitmask instead of a multiply — 16 × 16
> cells of 375 units — and the cell size exists both as a derived quantity and as a literal in
> `.rdata`, which is what verifies it.

[← C5.2 — Sector index arithmetic](02-sector-index-arithmetic.md) · [Chapter 5 hub](C5-CWorld.md) ·
[Next: C5.4 — The PtrNode coupling →](04-ptrnode-coupling.md)

**Confidence:** ⚠️ **Substantially superseded** — see [C5.6 §4](06-the-sector-arrays-closed.md)

> ⚠️ **Correction.** §2's central claim — that repeat cells are 375 units and span the map — is
> **withdrawn**. The mask in §1 is applied to the **sector index**, not to a world coordinate, so the
> grid tiles every 16 × 50 = **800 units** and wraps. §3's "the grids do not nest" is also withdrawn:
> they nest exactly, 16 sectors per tile. The `375.0` literal is **coincidental**.
>
> §1 (the mask), §4 (why a coarse grid helps) and §5 (the negative result on `1/375`) stand — and §5
> was the clue that should have been trusted over the coincidence. The page is kept intact so the
> reasoning error is legible.

---

## 1. The mask

```
00409116: and  ebp, 0xf
```

✅ *Verified.* **16 cells per axis, 256 total.**

A mask rather than a multiply-and-floor is a different indexing strategy from the fine grid
([C5.2](02-sector-index-arithmetic.md)), and the difference is informative: masking wraps rather than
clamps. An index outside `0 … 15` does not go out of bounds, it aliases back into range.

🟡 *Reasoned:* this is very likely why the grid is called "repeat" — the addressing repeats, so a query
never falls off the edge. That reading fits the instruction exactly but was not confirmed against the
population code.

## 2. The cell size, twice

**Derived:** the grid spans the same −3000 … +3000 as the fine grid
([C5.1](01-the-two-grids.md)), so

```
6000 / 16 = 375
```

**Literal:** the value exists in `.rdata`:

```
0x0086555C = 375.0
```

✅ Verified — and it is the *only* occurrence of `375.0` as a float constant in the scanned range, which
makes the coincidence-of-two-derivations argument strong rather than accidental.

This is the same verification pattern as the fine grid's endpoints landing on 0 and 120: a quantity
that can be computed two independent ways, and does.

## 3. Fine and coarse compared

| | Sectors | Repeat sectors |
|---|---|---|
| Cells per axis | 120 | 16 |
| Cell size | 50 | 375 |
| Total cells | 14,400 | 256 |
| Index method | `floor(c × 0.02 + 60)` | `index & 0xF` |
| Out-of-range behaviour | clamp (where present) | wrap |
| Ratio of cell areas | 1× | **56.25×** |

The 7.5× ratio per axis is not a round number in cells (120 / 16 = 7.5), which means the two grids do
**not** nest — a repeat-sector boundary does not align with a sector boundary except every other one.
🟡 *Reasoned:* they are independent partitions of the same space rather than a hierarchy, so code cannot
cheaply map one to the other and must compute each from the coordinate.

## 4. Why a coarse grid exists at all

The cost of the fine grid is re-linking. An entity moving at 30 units/second crosses a 50-unit cell
boundary roughly every 1.7 seconds per axis; across 110 vehicles and 140 peds
([C4.1](../C4-Entities-And-Pools/01-the-pool-table.md)) that is a continuous churn of list removals and
insertions, each consuming and releasing a pointer node.

At 375 units the same entity crosses a boundary every ~12 seconds — a 7.5× reduction in link churn per
axis.

🟡 *Reasoned:* this is the standard justification and fits the numbers, but as
[C5.1 §2](01-the-two-grids.md) notes, the population code was not read. What is certain is the geometry;
the duty split is inference.

⏳ **Open:** confirming which entity classes populate which grid. The cheapest route is the pointer-node
allocation sites — whichever grid's insertion path draws from `PtrNode Double` rather than
`PtrNode Single` is likely the dynamic one, given the 3,200 vs 70,000 split
([C5.4](04-ptrnode-coupling.md)).

## 5. Reading a repeat-sector index

The mask alone is not the whole conversion — a coordinate still has to become an integer before masking.
This pass observed the mask but not the multiply-and-offset that precedes it, so:

⏳ **Open:** the exact pre-mask arithmetic. By analogy with the fine grid it would be
`floor(coord × (1/375) + 8)`, and `1/375 = 0.0026666…` — but **that constant was searched for in
`.rdata` and not found**, so the analogy does not hold and the real computation is something else.

That negative result is worth recording rather than discarding: it rules out the obvious hypothesis and
tells the next pass to look for a different formulation — possibly deriving the coarse index from the
fine one, or using an integer shift on an already-computed value.

---

### Key takeaways

- **16 × 16 = 256 cells**, indexed by `and reg, 0xF` — masking, so out-of-range **wraps** rather than
  clamping.
- Cell size **375 units**, confirmed twice: `6000 / 16` and a literal `375.0` at `0x0086555C`.
- The grids do **not nest** — 120 / 16 = 7.5, so boundaries align only every other coarse cell.
- 🟡 The coarse grid plausibly exists to cut link churn for moving entities by ~7.5× per axis; the duty
  split is inference, not proof.
- ⏳ The pre-mask arithmetic is **not** the obvious analogue of the fine grid — `1/375` does not appear
  in `.rdata`, a negative result that redirects the next pass.

**Continue:** [C5.4 — The PtrNode coupling](04-ptrnode-coupling.md)
