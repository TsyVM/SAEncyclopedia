# C5.4 — The PtrNode Coupling

> **The one-sentence version:** `PtrNode Single` is 70,000 because entities are threaded into every
> sector they overlap — which means the building limit and the pointer-node limit are coupled through
> the grid, and raising one without the other fails in a way that looks random.

[← C5.3 — Repeat sectors](03-repeat-sectors.md) · [Chapter 5 hub](C5-CWorld.md) ·
[Next: C5.5 — Open: the sector arrays →](05-open-the-sector-arrays.md)

**Confidence:** 🟡 Reasoned, from ✅ verified inputs

---

## 1. The two numbers

| From | Value | Source |
|---|---:|---|
| `PtrNode Single` capacity | 70,000 | [C4.1](../C4-Entities-And-Pools/01-the-pool-table.md) ✅ |
| `PtrNode Double` capacity | 3,200 | [C4.1](../C4-Entities-And-Pools/01-the-pool-table.md) ✅ |
| `Buildings` capacity | 13,000 | [C4.1](../C4-Entities-And-Pools/01-the-pool-table.md) ✅ |
| `Dummys` capacity | 2,500 | [C4.1](../C4-Entities-And-Pools/01-the-pool-table.md) ✅ |
| Fine cell size | 50 units | [C5.2](02-sector-index-arithmetic.md) ✅ |

70,000 pointer nodes against 13,000 buildings is **5.4 nodes per building** if buildings were the only
consumer. That ratio is the observation this page explains.

## 2. The membership model

An entity occupies a *region*, not a point. To answer "what is near here" by looking in one cell, the
world must have already placed every entity in **every cell its bounding box overlaps**. Each such
placement is one list node.

So the pool is not sized for objects. It is sized for **memberships**:

```
nodes ≈ Σ over entities of (cells spanned by that entity)
```

🟡 *Reasoned.* The membership model is the standard spatial-hash design and is the only reading
consistent with a 70,000-node pool serving ~15,500 static entities. But this pass did not read the
insertion code, so the model is inference from the numbers rather than proof from instructions.

## 3. The arithmetic of spanning

At 50 units per cell, an axis-aligned box of size `w × h` spans about

```
(w/50 + 1) × (h/50 + 1)
```

cells — the `+1` because a box rarely aligns to the grid.

| Entity size | Cells spanned |
|---|---:|
| 10 × 10 (a prop) | 1 |
| 50 × 50 (a small building) | ~4 |
| 100 × 100 (a large building) | ~9 |
| 200 × 20 (a road segment) | ~10 |

Against 13,000 buildings plus 2,500 dummies, an average of four memberships each gives ~62,000 nodes —
which is why 70,000 is the number and not 20,000. The pool is sized with modest headroom over the
shipped map's actual membership count.

> ⚠️ **Corrected — see [C11.3 §2](../C11-IDE-And-IPL/03-what-placements-prove.md).** The arithmetic above
> multiplies *pool capacities* as if they were the world's object count. The placement census shows the
> world holds **45,884 placements against 15,500 resident building+dummy slots** — the pools are a
> streaming window, roughly 3× smaller than the world. The pool is sized against **peak concurrent
> memberships**, giving ~4.5 nodes per resident entity. The conclusion — memberships, not objects —
> stands; this number was reached by a route that does not hold.

🟡 The average of four is an illustrative fit, not a measurement. Measuring it properly means parsing
the IPL placements and their model bounds — entirely doable with the archive reader from
[C1.1](../C1-Streaming/01-img-ver2-archive-model.md), and not done here.

## 4. Why this is the coupling that breaks limit adjusters

Raising `Buildings` from 13,000 to 30,000 without raising `PtrNode Single` produces a game that:

- accepts the new buildings into the pool (there is room);
- fails to link some of them into their sectors (there is not);
- renders and collides inconsistently, in a location-dependent way, with no error.

The failure is location-dependent because it depends on which entities happened to exhaust the node
pool first — which is why it reads as random corruption rather than a limit being hit.

**The coupling factor is content-dependent.** It is not "N nodes per building" as a constant; it is the
average sector span of *your* geometry. A mod that adds many small props needs fewer extra nodes per
object than one that adds large structures. This is why limit adjusters raise both and why no single
correct ratio exists.

## 5. Single versus double

`PtrNode Single` 70,000 and `PtrNode Double` 3,200 — a 22:1 split.

🟡 *Reasoned:* singly-linked nodes suffice for lists that are only ever walked and rebuilt wholesale
(static geometry per sector); doubly-linked nodes are needed where an individual entry must be removed
in O(1) without walking — which is what a **moving** entity requires when it leaves a cell.

3,200 double nodes against 110 vehicles + 140 peds = 250 movers gives ~12.8 memberships each, which is
plausible for entities tracked in both grids.

⏳ **Open.** This is the most testable inference in the chapter: find the allocation site for each pool
and see which insertion path draws from which. That single observation would confirm or refute both
this section and the duty split in [C5.1 §2](01-the-two-grids.md).

## 6. What this means for the SDK

An SDK that exposes spatial queries must not hide this cost. `SA::World::EntitiesNear(point, radius)`
looks like a cheap call and is — but `SA::World::Add(entity)` is O(cells spanned), and a plugin that
moves many entities per frame is doing far more work than the call site suggests.

The honest surface reports it:

```cpp
// Cheap: one cell lookup.
Result<Span<EntityHandle>> EntitiesInSector(SectorIndex);

// Cost is proportional to the entity's bounding box, not O(1).
Result<void> Relink(EntityHandle) noexcept;
```

And per [C4.4](../C4-Entities-And-Pools/04-pools-and-the-sdk.md), any SDK-side book-keeping sized
against node counts must read the live pool capacity, never the 70,000 documented here.

---

### Key takeaways

- The pointer-node pool is sized for **memberships, not objects** — every entity is linked into every
  cell it overlaps.
- 70,000 nodes ÷ ~15,500 **resident** static slots ≈ **4.5 memberships each** — but see
  [C11.3 §2](../C11-IDE-And-IPL/03-what-placements-prove.md): the world has **45,884 placements**, so the
  pools hold a streaming window, not the world.
- **Building and node limits are coupled through the grid**, and the factor is *content-dependent* —
  there is no universal ratio.
- Raising `Buildings` alone yields **location-dependent, silent** failure, which is why it looks random.
- 🟡 The 22:1 single/double split plausibly separates static (rebuild-wholesale) from dynamic (O(1)
  removal) lists — the single most testable open inference in the chapter.
- The SDK must not present relinking as O(1); its cost scales with bounding-box size.

**Continue:** [C5.5 — Open: the sector arrays](05-open-the-sector-arrays.md)
