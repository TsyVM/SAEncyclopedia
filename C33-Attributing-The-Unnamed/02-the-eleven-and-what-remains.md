# C33.2 — The Eleven, and What Remains

> **The one-sentence version:** eleven of C27's 124 unnamed functions carry a STRONG class attribution —
> data reference plus address-cluster agreement — four of them spot-checked from disassembly to prove the
> reference is real; four more are weaker data-only leads; and the other 109 simply don't touch any of the
> twelve signatures, which is itself informative.

**Subsystem category:** Binary substrate — results
**Depends on:** [C33.1 — The classifier](01-the-classifier.md)
**RE status:** Documented
**Confidence:** 🟡 for the class attributions · ✅ for the four spot-checks
**Data artifact:** [`RE-Data/data/attribution.json`](../RE-Data/data/attribution.json)

---

## 1. The eleven STRONG attributions

Each references its class's signature data **and** sits inside that class's named-function cluster:

| `entry_va` | Extent | Attributed class | Signature hit |
|---|---:|---|---|
| `0x0044D520` | 608 | `CPathFind` | node/link tables `0x96F854`/`0x96FA94` (**17 refs**) |
| `0x0043EF00` | 32 | `CEntryExitManager` | pool object `0x96A7D8` |
| `0x0043EF20` | 112 | `CEntryExitManager` | pool object `0x96A7D8` (×2) |
| `0x0043EF90` | 64 | `CEntryExitManager` | pool `0x96A7D8` + stride `0x3C` |
| `0x0043E400` | 16 | `CEntryExitManager` | pool region `0x96A7CC` |
| `0x0043F720` | 96 | `CEntryExitManager` | stride `0x3C` |
| `0x0043F7D0` | 112 | `CEntryExitManager` | stride `0x3C` |
| `0x00448990` | 96 | `CGarages` | garage pool `0x96C048` (×2) |
| `0x00407800` | 32 | `CStreaming` | streaming-info `0x8E4CCC` |
| `0x00410F80` | 96 | `CColStore` | stride `0x2C` |
| `0x00411030` | 112 | `CColStore` | stride `0x2C` |

Six of the eleven are `CEntryExitManager` — unsurprising, since that class's pool object and 60-byte record
([C30.1](../C30-Gameplay-Managers/01-centryexitmanager.md)) are touched by many small helpers, and the class
sits in a tight `.text` cluster (`0x43E…0x440`) so proximity readily agrees.

## 2. Four spot-checks — the reference is real

To prove a hit is genuine manipulation and not a constant that merely *looks* like a signature address, four
were disassembled in full:

**`0x00407800` → CStreaming.** `lea eax,[eax+eax*4]` then `mov ecx,[eax*4 + 0x8E4CCC]` — index × 20 into the
streaming-info array at base + 0xC, returning whether that pointer field is set. This is a streaming-info
accessor, unambiguously ([C28.3](../C28-Class-Catalogue/03-cstreaming-and-rederiving-c2.md)).

**`0x00410F80` → CColStore.** `imul eax,eax,0x2C` then two allocator calls whose results are stored at
`[esi]` and `[esi+4]` — it builds a 44-byte CColStore slot's sub-arrays, i.e. an `AddColSlot`-family method
([C31.1](../C31-Streaming-Slot-Tables/01-ccolstore.md)).

**`0x0043EF00` → CEntryExitManager.** `mov ecx,[0x96A7D8]; mov edx,[ecx+4]; cmp byte [edx+eax],0; jns …` —
reads the entry/exit pool object, its per-slot flag-byte array at +4, and tests a slot's validity: a pool
slot-validity check ([C30.1](../C30-Gameplay-Managers/01-centryexitmanager.md)).

**`0x00448990` → CGarages.** `mov edx,0x96C048` then a loop reading `[edx+0x4C]` (the garage type byte) — it
scans the garage pool by type ([C29.2](../C29-Gameplay-Object-Pools/02-cgarages-the-garage-pool.md)).

In all four the referenced static is *used as* the structure the source chapter proved it to be — the hit is
real.

## 3. Four data-only leads

Four more functions reference a class's data but sit **outside** that class's cluster, so they are weaker —
most likely helpers or callers that live in another translation unit yet operate on the class's structures:

| `entry_va` | Extent | Callers | References |
|---|---:|---:|---|
| `0x00422590` | 144 | 4 | CPathFind node table `0x96F854` (×4) |
| `0x0044CC20` | 304 | 1 | CTheScripts global var space `0xA49960` (×3) |
| `0x0042F870` | 352 | 16 | CPathFind singleton `0x96F050` |
| `0x0046AA80` | 112 | 1 | CStreaming region `0x964E08` |

These are recorded as leads, **not** attributions: a function that reads the path-node table but lives far
from `CPathFind` in `.text` may be a pathfinding *consumer* (traffic, peds) rather than a `CPathFind` method,
so the class vote is real but the ownership is ambiguous.

## 4. Why 109 don't resolve — and that's a result

The classifier finds a signature reference in only 15 of 124. The other 109 are silent, and the reason is
structural, not a failure: the signature map covers **twelve** classes — the ones [C28–C32](../C28-Class-Catalogue/C28-Class-Catalogue.md)
chose for their clean fixed structures. C27's 124 unnamed are dominated by the **long tail** of one-method
`CTask*`/`CEvent*` constructors and small helpers ([C27.2 §2](../C27-Function-Catalogue/02-the-catalogue-and-class-map.md#2-reading-the-shape-of-the-list))
whose classes were never structured, so there is no fingerprint to match. The classifier therefore has a
known ceiling: it can only attribute a function to a class it has already characterised. Extending it is
mechanical — structure another class (a future C28-style chapter), add its statics to the signature map,
re-run — and the 109 will shrink each time. This is the honest shape of the result: **11 closed on two
signals, 4 leads, and a method that grows with the encyclopedia rather than a one-time sweep.**

## 5. The living classifier — state at last run vs. current map

The `attribution.json` above reflects the classifier's state when `derive_attribution.py` was last run. The
signature map has since grown: [C30.4](../C30-Gameplay-Managers/04-cgangwars.md) added two new fingerprints —
the six `CGangWars` active-war globals (`0xC8A4A4`–`0xC8A4B8`) and the 320-byte zone ownership byte array
at `0xC8B2C0`, plus the 10-entry × 16-byte `CGangs` static table at `0xC091F0` with its `×0x10` stride.

These are now in [C33.1](01-the-classifier.md)'s signature table. The counts above (11 STRONG, 4 data-only)
therefore reflect the pre-C30.4 run. Whether any of the 109 unnamed functions reference `0xC8B2C0` or
`0xC091F0` is not yet known without a re-run. The zone-ownership array is particularly distinctive — it is
a 320-byte contiguous block at a unique address, and any unnamed function that byte-indexes it is doing
gang-territory work. The tool re-run is the only step needed to find out.

Until then, treat the 11 and 4 figures as a **floor** of what the data-flow classifier finds — not an
upper bound.

---

### Key takeaways

- **11** unnamed functions gain a STRONG class attribution (data + proximity); six are `CEntryExitManager`,
  the rest split across CPathFind, CGarages, CStreaming and CColStore.
- **4** spot-checks confirm the referenced static is genuinely used as its source chapter's structure.
- **4** data-only leads reference a class's data from off-cluster — recorded as leads, ownership ambiguous.
- The **109** non-matches are the unstructured long tail; the classifier's ceiling is "classes already
  characterised," so it grows as more chapters land — a repeatable method, not a finished sweep.
- The current 11+4 counts reflect the **pre-C30.4 run**; CGangWars and CGangs entries are now in the map
  but a tool re-run is needed to attribute any functions that reference them.

**Next:** [C33.3 — The remaining 109 and what would close them](03-the-remaining-109.md).
