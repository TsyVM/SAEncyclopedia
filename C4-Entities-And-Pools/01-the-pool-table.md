# C4.1 — The Pool Table

> **The one-sentence version:** seventeen fixed-capacity pools, built in one function, each carrying
> the engine's own name string — so the famous San Andreas limits can be read off the binary with
> their labels attached rather than measured from the outside.

[← Chapter 4 hub](C4-Entities-And-Pools.md) · [Next: C4.2 — The construction idiom →](02-construction-idiom.md)

**Confidence:** ✅ Verified
**Function:** `CPools::Initialise` at `0x00550F10`

---

## 1. The table

| # | Pool global | Capacity | Engine's name string |
|---:|---|---:|---|
| 1 | `0x00B74484` | 70,000 | `PtrNode Single` |
| 2 | `0x00B74488` | 3,200 | `PtrNode Double` |
| 3 | `0x00B7448C` | 500 | `EntryInfoNode` |
| 4 | `0x00B74490` | 140 | `Peds` |
| 5 | `0x00B74494` | 110 | `Vehicles` |
| 6 | `0x00B74498` | 13,000 | `Buildings` |
| 7 | `0x00B7449C` | 350 | `Objects` |
| 8 | `0x00B744A0` | 2,500 | `Dummys` |
| 9 | `0x00B744A4` | 10,150 | `ColModel` |
| 10 | `0x00B744A8` | 500 | `Task` |
| 11 | `0x00B744AC` | 200 | `Event` |
| 12 | `0x00B744B0` | 64 | `PointRoute` |
| 13 | `0x00B744B4` | 32 | `PatrolRoute` |
| 14 | `0x00B744B8` | 64 | `NodeRoute` |
| 15 | `0x00B744BC` | 16 | `TaskAllocator` |
| 16 | `0x00B744C0` | 140 | `PedIntelligence` |
| 17 | `0x00B744C4` | 64 | `PedAttractors` |

✅ *Verified* — each row read from the `push <count>` / `push <name>` / `mov [global], eax` triple. The
globals are contiguous at 4-byte spacing from `0x00B74484` through `0x00B744C4`: seventeen slots, no
gaps.

Note the spelling `Dummys`. It is the engine's, and it is reproduced rather than corrected — a name
table is evidence, and silently normalising it would make the entry harder to match against the binary.

## 2. Why the names matter

Most published GTA SA limit tables are community measurements: someone spawned objects until the game
broke. This table is different — the counts and the labels come out of the same instruction sequence,
so there is no question of a label being attached to the wrong number.

That also settles ambiguities the community tables carry. `PedIntelligence` at exactly 140 matching
`Peds` at exactly 140 is visible here as a deliberate one-per-ped pairing, not a coincidence to be
inferred.

## 3. Reading the numbers

### The pointer-node pools dominate

`PtrNode Single` at **70,000** is by far the largest — five times `Buildings`. It is not sized for
objects; it is sized for **object-to-sector memberships**, because an entity is threaded into the list
of every world sector its bounds overlap ([C5.4](../C5-CWorld/04-ptrnode-coupling.md)).

### The famous small ones

`Peds` 140 and `Vehicles` 110 are the two limits San Andreas is best known for, and the contrast with
`Buildings` 13,000 is the design in one line: static geometry is cheap, and anything that thinks is
not.

### The shared AI budgets

`Task` 500 and `Event` 200 are **shared across all peds**, unlike `PedIntelligence` which is
per-ped-slot. At 140 peds that is roughly 3.5 concurrent tasks per ped before the pool is exhausted.
These are the limits that bite in heavily-scripted scenes, and they are far less discussed than the ped
and vehicle counts.

### `ColModel` at 10,150

Sitting between `Buildings` and `Dummys` in magnitude, and notably not a round number. 🟡 *Reasoned:*
it is sized against the collision archives' actual content rather than chosen — the kind of number that
comes from measuring a shipped data set.

## 4. What a pool global holds

Each `0x00B744xx` slot holds a **pointer to a 20-byte `CPool` object**, not the storage itself
([C4.2](02-construction-idiom.md)). Reading the capacity at runtime therefore means dereferencing the
global and reading a field of the pool object — which is what the SDK must do
([C4.4](04-pools-and-the-sdk.md)), because the numbers above are the *shipped defaults* and limit
adjusters change them.

⏳ **Open:** the `CPool` object's own field layout — where the capacity, the storage pointer and the
slot-state array live within those 20 bytes — was not recovered in this pass. That is the gap between
"we know the defaults" and "the SDK can read the live values", and it is the obvious next step for this
chapter.

---

### Key takeaways

- **Seventeen pools**, contiguous globals `0x00B74484 … 0x00B744C4`, built at `0x00550F10`.
- Capacities come **with the engine's own name strings**, so no label is guessed — including the
  engine's spelling, `Dummys`.
- **`PtrNode Single` 70,000** is sized for sector memberships, not objects.
- `Peds` 140 / `Vehicles` 110 against `Buildings` 13,000 is the static-vs-thinking split.
- **`Task` 500 and `Event` 200 are shared** across all peds — the under-discussed limits.
- `PedIntelligence` 140 is visibly **one per ped slot**, not a coincidence.
- Each global holds a **pointer to a 20-byte pool object**; the object's internal layout is **open**,
  which currently blocks runtime capacity reads.

**Continue:** [C4.2 — The construction idiom](02-construction-idiom.md)
