# C5.1 — The Two Grids

> **The one-sentence version:** San Andreas partitions the same 6000 × 6000-unit square twice — a fine
> 120 × 120 grid and a coarse 16 × 16 one — and the two derivations agreeing on the world extent is
> what makes the geometry certain.

[← Chapter 5 hub](C5-CWorld.md) · [Next: C5.2 — Sector index arithmetic →](02-sector-index-arithmetic.md)

**Confidence:** ✅ Verified (fine grid) / ⚠️ superseded (repeat grid)

> ⚠️ **Correction — see [C5.6 §4](06-the-sector-arrays-closed.md).** The repeat-grid row in §1 below is
> **wrong**. The repeat grid is not a 375-unit partition spanning the map; it is a **wrapping hash that
> tiles every 800 units**, with 56.25 world cells sharing each bucket. Everything about the *fine* grid
> on this page stands. The row is left in place, struck, so the correction is legible rather than
> invisible.

---

## 1. The two partitions

| Grid | Cells | Cell size | Extent | Total cells |
|---|---|---|---|---:|
| **Sectors** | 120 × 120 | 50 × 50 units | −3000 … +3000 | 14,400 |
| ~~**Repeat sectors**~~ | ~~16 × 16~~ | ~~375 × 375 units~~ | ~~−3000 … +3000~~ | ~~256~~ |
| **Repeat sectors** (corrected) | 16 × 16 | 50 × 50, **tiling every 800 units** | wraps | 256 |

The fine grid is ✅ verified three independent ways
([C5.2](02-sector-index-arithmetic.md)). The repeat grid's *cell count* is verified; its geometry was
misread here and is corrected in [C5.6](06-the-sector-arrays-closed.md).

## 2. Why two

A single grid cannot be right for both jobs.

**Static geometry wants fine cells.** A collision or render query asks "what is near this point", and
with 13,000 buildings ([C4.1](../C4-Entities-And-Pools/01-the-pool-table.md)) a coarse grid would return
far too many candidates per query. 50-unit cells keep the candidate set small.

**Moving entities want coarse cells.** A ped or vehicle changes cell constantly. With 50-unit cells,
anything moving at speed re-links itself several times a second — and each re-link is a list removal and
insertion. 375-unit cells cut that by roughly 7.5× per axis.

🟡 *Reasoned:* this division — fine grid for static, coarse for dynamic — is the standard reading and is
consistent with the cell counts (14,400 vs 256) and with `PtrNode Single` being sized for static
memberships ([C5.4](04-ptrnode-coupling.md)). This pass verified the *geometry* of both grids but did
not read the population code, so which entity classes go into which grid is inference, not proof.

⏳ **Open:** the population path — the function that inserts an entity into its cells — would settle it
outright.

## 3. The extent is chosen, not derived

Worth stating plainly: **−3000 … +3000 is baked into the index arithmetic as a constant**, not computed
from map data. The `+60.0` in the sector formula and the `× 0.02` together define the origin and the
resolution ([C5.2](02-sector-index-arithmetic.md)).

Consequences:

- The playable area cannot exceed 6000 × 6000 units without changing code, not data.
- Anything placed outside that square produces an out-of-range sector index. Whether that is clamped or
  simply indexes past the array was not established here — and it is the mechanism behind the
  long-standing "objects far from the map behave strangely" reports.

⏳ **Open:** whether the index is clamped. The one clamp observed in this pass is in a different
function (`0x00546670`, `cmp esi, 0x77; jge → esi = 0x77` — a clamp to 119, the last sector index),
which shows clamping happens *somewhere*, but not that it happens on every path.

That `0x77 = 119` clamp is itself a nice independent confirmation of the 120-cell count: the code
clamps to the maximum valid index, and that maximum is 119.

## 4. 36 km² in 14,400 cells

At 50 units per cell and the conventional ~1 unit ≈ 1 metre reading, the world is 6 km × 6 km = 36 km²,
each fine cell 2,500 m².

The numbers are worth holding because they set expectations for everything downstream: a query against
the fine grid touches one cell of 14,400, a sweep over the coarse grid touches 256 cells, and an entity
spanning a 200-unit bounding box occupies 16 fine cells and 1 coarse one — which is exactly the ratio
that drives the pointer-node pool size in [C5.4](04-ptrnode-coupling.md).

---

### Key takeaways

- **Two grids over the same square**: 120 × 120 × 50 units, and 16 × 16 × 375 units, both spanning
  −3000 … +3000.
- The two are derived by **different arithmetic** (multiply-and-offset vs mask) and agree on the extent
   — that agreement is the verification.
- 🟡 Fine grid for **static geometry**, coarse for **moving entities** — consistent with the cell counts
  but not proved here; the population path is open.
- The extent is a **constant in the code**, so 6000 × 6000 is a code limit, not a data limit.
- A `cmp esi, 0x77` clamp to **119** independently confirms the 120-cell count.

**Continue:** [C5.2 — Sector index arithmetic](02-sector-index-arithmetic.md)
