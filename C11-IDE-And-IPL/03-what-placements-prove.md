# C11.3 — What the Placements Prove

> **The one-sentence version:** 36,569 real object positions confirm the sector grid derived from three
> floating-point instructions in C5 — and correct the pool arithmetic C5.4 built on an estimate.

[← C11.2 — IPL placements](02-ipl-placements.md) · [Chapter 11 hub](C11-IDE-And-IPL.md)

**Confidence:** ✅ Verified
**Confirms:** [C5.2](../C5-CWorld/02-sector-index-arithmetic.md), [C3.1](../C3-Model-Stores/01-the-partition.md)
**Corrects:** [C5.4 §3](../C5-CWorld/04-ptrnode-coupling.md)

---

## 1. The world fits its grid to within 7 units

[C5.2](../C5-CWorld/02-sector-index-arithmetic.md) derived the sector grid from three instructions:

```
fmul dword ptr [0x858b38]     ; × 0.02
fadd dword ptr [0x858b34]     ; + 60.0
```

giving 120 cells of 50 units spanning **−3000 … +3000**. That was a claim about *code*.

Testing it against the 36,569 binary placements — a claim about *content*, authored by level designers
through an entirely separate tool chain:

| Axis | Minimum | Maximum | Grid bound | Margin |
|---|---:|---:|---:|---:|
| X | **−2993.8** | **2955.5** | ±3000 | 6.2 / 44.5 |
| Y | **−2937.5** | **2956.4** | ±3000 | 62.5 / 43.6 |
| Z | −76.1 | 1382.2 | *(ungridded)* | — |

| Check | Result |
|---|---|
| **Placements outside −3000 … +3000** | **0 of 36,569 (0.00 %)** |

✅ Not one. And the closest approach is **6.2 units** — 0.1 % of the grid's width.

This is the cleanest cross-domain confirmation in the encyclopedia. The constants were read from
compiled floating-point instructions; the extents come from hand-placed art. They agree exactly, and
the content presses right up against the boundary — which tells you the designers knew the limit and
used all of it.

**The grid is not a description of where the content happens to be. It is a constraint the content was
built to fill.**

## 2. ⚠️ Correction: C5.4's pool arithmetic

[C5.4 §3](../C5-CWorld/04-ptrnode-coupling.md) reasoned about the 70,000-entry `PtrNode Single` pool
like this:

> *"Against 13,000 buildings plus 2,500 dummies, an average of four memberships each gives ~62,000
> nodes — which is why 70,000 is the number."*

That arithmetic used **pool capacities** as if they were the world's object count. The placement census
shows why that was the wrong quantity:

| Quantity | Value |
|---|---:|
| Total world placements | **45,884** |
| `Buildings` pool capacity | 13,000 |
| `Dummys` pool capacity | 2,500 |
| `PtrNode Single` capacity | 70,000 |

**There are 45,884 placements but only 15,500 building + dummy slots.** The world contains roughly
three times more objects than can be resident at once, so the pools hold a *streaming window*, not the
world.

The corrected reading: the node pool is sized against **peak concurrent memberships** — how many
sector-list entries the resident window needs at its worst — not against total placements and not
against pool capacity either. At 70,000 nodes over at most 15,500 resident static entities, the true
headroom is **4.5 nodes per resident entity**, which is a plausible average sector span for 50-unit
cells.

So C5.4's *conclusion* — that the pool is sized for memberships and that building and node limits are
coupled — stands. Its *number* was reached by multiplying two capacities that do not multiply. The
figure happened to land near 62,000 because 15,500 × 4 is coincidentally close to the right answer by a
different route.

🟡 *Reasoned:* "peak concurrent" is not directly measurable from static data — it depends on where the
camera goes. What is ✅ is that placements (45,884) exceed resident capacity (15,500) by 3×, which is
enough to invalidate the original arithmetic.

## 3. Definitions versus placements

| Quantity | Value |
|---|---:|
| Object definitions (`objs` + `tobj` + `anim`) | 14,266 |
| Distinct IDs placed — text | 6,453 |
| Distinct IDs placed — binary | 5,241 |
| Total placements | 45,884 |

🟡 *Reasoned:* fewer than half of all defined objects appear in any placement file. The remainder are
placed by **script** (mission props), spawned by **code** (pickups, vehicles), or are simply unused
definitions left in the data — the last being ordinary for a shipped game, and consistent with the
duplicate-ID sloppiness in [C11.1 §4](01-ide-definitions.md).

The ratio also explains the streaming budget's viability: 13.18 MiB
([C2.3](../C2-CStreaming/03-memory-budget-and-stream-ini.md)) cannot hold 14,266 models, but it does not
need to — it holds the few hundred distinct models visible from one point in a world built by repeating
about 5,000 of them.

## 4. Density

45,884 placements over 36 km² is **1,275 objects per km²**, or roughly one object every 28 metres
squared.

Per sector — 14,400 cells of 50 × 50 units ([C5.1](../C5-CWorld/01-the-two-grids.md)) — that averages
**3.2 placements per sector**. Sparse enough that most sector lists are very short, which is exactly the
condition under which a 50-unit grid pays off: a query touches one cell and finds three candidates.

🟡 The average hides enormous variance — city blocks against empty desert — but the order of magnitude
is what justifies the cell size.

## 5. What this chapter could not test

⏳ The 9,315 text placements were **not** included in the extent check; only the 36,569 binary ones
were. Text IPLs include interiors, which are conventionally placed far outside the map on a hidden
shelf — so they may well fall outside the grid legitimately, and testing them without accounting for
that would produce a misleading result.

That is a real gap, and it is the kind that matters: **the "0 outside the grid" claim covers 80 % of the
world, not all of it.** Extending it properly means separating exterior from interior placements first.

---

### Key takeaways

- **Zero of 36,569 binary placements fall outside the −3000 … +3000 grid**, with the closest approach
  **6.2 units** — code-derived constants and hand-authored content agreeing exactly.
- The grid is **a constraint the content was built to fill**, not a description of where it happens to
  be.
- ⚠️ **C5.4's node arithmetic is corrected**: it multiplied pool capacities as if they were world
  counts. There are **45,884 placements against 15,500 resident slots** — pools hold a streaming window,
  not the world. The conclusion stands; the number was reached wrongly.
- Fewer than half of defined objects are placed by IPL — the rest come from script, code, or are unused.
- **3.2 placements per sector** on average, which is what makes a 50-unit grid worth having.
- ⏳ The extent check covers **binary IPLs only (80 % of the world)**; text placements include interiors
  and need separating first.

**Continue:** [Chapter 11 hub](C11-IDE-And-IPL.md) · next chapter: `C12 — The Path Network`
