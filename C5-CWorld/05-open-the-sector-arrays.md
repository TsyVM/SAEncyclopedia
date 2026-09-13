# C5.5 — Open: the Sector Arrays

> **The one-sentence version:** the grid arithmetic is certain and the containers it indexes are not —
> two candidate base addresses, neither confirmed, recorded as candidates because guessing here would
> poison an otherwise solid chapter.

[← C5.4 — The PtrNode coupling](04-ptrnode-coupling.md) · [Chapter 5 hub](C5-CWorld.md)

**Confidence:** ⏳ Open
**Status:** the chapter's honest boundary

---

## 1. What is missing

Chapter 5 establishes how to compute a sector index ([C5.2](02-sector-index-arithmetic.md)) and a
repeat-sector index ([C5.3](03-repeat-sectors.md)). It does **not** establish what those indices index.

Specifically, three things are unknown:

1. **The array base addresses** — `ms_aSectors` and `ms_aRepeatSectors`.
2. **The per-cell record layout** — how many pointer lists each cell holds, and in what order.
3. **The insertion path** — the function that threads an entity into its overlapping cells.

## 2. The candidates, and why they stay candidates

Reference counts in the `0x00B7xxxx` band, gathered while disassembling the functions that use the
`× 120` stride:

| Address | Refs |
|---|---:|
| `0x00B71670` | 35 |
| `0x00B79538` | 34 |
| `0x00B7CD98` | 19 |
| `0x00B70198` | 19 |
| `0x00B71A60` | 13 |
| `0x00B7D0B8` | 8 |

`0x00B7CD98` and `0x00B7D0B8` are the leading candidates for the two sector arrays, on two weak grounds:
they sit in the right region, and they are the two highest-referenced addresses in the band that are not
obviously something else.

**That is not evidence.** No instruction was read that indexes either of them with a sector index. A
plausible address with a plausible reference count is exactly the kind of claim that looks like a
finding and is not one.

Per the tiering in [C0.3 §1](../C0-Binary-Identity/03-external-analysis-corroboration-and-conflicts.md),
these are recorded so the next pass has a starting point, and marked so nobody builds on them.

## 3. Why the chapter ships anyway

The split is deliberate. The grid arithmetic is:

- **verified three independent ways** (the `+60.0` endpoints, the `× 120` stride in five functions, the
  `0…119` clamp);
- **immediately useful** — any tool that needs to know which sector a coordinate falls in can compute it
  today;
- **independent of the containers** — the formula does not change when the array base is learned.

Publishing verified arithmetic without the arrays is more useful than withholding both, and far more
useful than publishing arithmetic plus a guessed base that later turns out wrong. A wrong base address
would not fail loudly; it would read plausible garbage, in the manner of the unsigned-`m_nModelIndex`
trap in [C4.3 §2](../C4-Entities-And-Pools/03-entity-model-link.md).

## 4. How to close it

In rough order of cost:

**Find an indexing instruction.** The five functions already located that use `imul reg, reg, 0x78`
([C5.2 §4](02-sector-index-arithmetic.md)) must, shortly after the multiply, add a base address. Reading
what each adds gives the base directly. This is the cheapest route and is the obvious next action —
the functions are `0x004090A0`, `0x00409210`, `0x0041A820`, `0x00546670`, `0x0054BA60`.

**Derive the record size from the stride.** Once a base is known, the multiplier applied after the
`× 120` gives the per-cell record size, and the array extent follows as
`base + 14400 × recordSize` — which can then be checked against the next known global, the technique
that proved both `CdStream` tables ([C1.2 §2](../C1-Streaming/02-cdstream-layer.md)) and the streaming
array ([C2.1 §3](../C2-CStreaming/01-the-id-space.md)).

**Read the insertion path.** That settles the record layout, the single/double node question from
[C5.4 §5](04-ptrnode-coupling.md), and the fine/coarse duty split from
[C5.1 §2](01-the-two-grids.md) — three open items in one function.

## 5. What else this chapter left open

Collected for the next pass:

| Item | Page |
|---|---|
| Which entity classes populate which grid | [C5.1 §2](01-the-two-grids.md) |
| Whether every index path clamps | [C5.2 §5](02-sector-index-arithmetic.md) |
| The repeat-sector pre-mask arithmetic (`1/375` is **not** in `.rdata`) | [C5.3 §5](03-repeat-sectors.md) |
| Single vs double node duty split | [C5.4 §5](04-ptrnode-coupling.md) |
| Measured average sector span of shipped geometry | [C5.4 §3](04-ptrnode-coupling.md) |

Every one of them is answered or strongly constrained by reading the insertion path. That makes it the
highest-value single target in the chapter, and the natural first task of the next pass.

---

### Key takeaways

- The chapter's boundary is explicit: **grid arithmetic verified, containers unknown.**
- `0x00B7CD98` and `0x00B7D0B8` are **candidates on reference-count grounds only** — recorded, not
  asserted.
- Shipping verified arithmetic without the arrays is correct; a guessed base would read plausible
  garbage rather than failing loudly.
- The cheapest close is reading what the **five known `× 120` sites** add after the multiply.
- **The insertion path answers five of this chapter's open items at once** — the highest-value next
  target.

**Continue:** [Chapter 5 hub](C5-CWorld.md) · next chapter: `C6 — Collision & the COL Model`
