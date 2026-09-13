# C18.2 — The Mission and External Tables

> **The one-sentence version:** 135 missions that tile the tail of the file with no gap and no overlap,
> and 79 streamed scripts addressed in a virtual space that begins at exactly `len(main.scm)` — each table
> confirmed by a field it had no obligation to agree with.

[← C18.1 — The segment chain](01-the-segment-chain.md) · [Chapter 18 hub](C18-SCM-Script.md) ·
[Next: C18.3 — The code stream →](03-the-code-stream.md)

**Confidence:** ✅ Verified over the full tables

---

## 1. The mission table

```c
struct MissionSegment {          // segment 3, ID = 1
    uint32_t mainSize;           // where the main script ends
    uint32_t largestMission;     // bytes
    uint32_t numMissions;        // 135
    uint32_t unknown;            // 964  — ⏳
    uint32_t offset[135];        // absolute file offsets
};
```

| Field | Value |
|---|---:|
| `mainSize` | **194,125** |
| `largestMission` | **68,439** |
| `numMissions` | **135** |
| `unknown` | ⏳ **964** |

Size check: `1 + 16 + 135 × 4 = 557`, and the segment body is **557** bytes. ✅ Exact — which is what
fixes the header at four dwords rather than three or five.

## 2. Two fields prove the table

Neither of these is redundant bookkeeping; both compare a *stored* value against a value **derived from
the offsets themselves**:

| Check | Stored | Derived | |
|---|---:|---:|---|
| `mainSize` == first mission offset | 194,125 | **194,125** | ✅ |
| `largestMission` == largest gap between consecutive offsets | 68,439 | **68,439** | ✅ |

And the offsets partition the remainder of the file completely:

| Check | Result |
|---|---:|
| Offsets strictly increasing | **✅ 134 / 134 pairs** |
| Σ (mission sizes) | **2,885,619** |
| `len(main.scm) − mainSize` | **2,885,619** |

✅ **The 135 missions tile `[194125, 3079744)` exactly** — no gap, no overlap, no padding.

**That is a much stronger statement than "the offsets look plausible."** A single wrong offset would break
the sum; a wrong header width would shift every offset and break the tiling; a wrong `numMissions` would
break the segment size. Three independent failure modes, none of which fire.

🟡 *Reasoned:* the exact tiling means missions are **not individually addressable resources** — they are
consecutive slices of one buffer, and the offset table is a slice index. The game loads one at a time into
a mission buffer sized by `largestMission`, which is precisely why that field is stored: it is an
allocation size, computed at build time so the runtime never has to scan.

**68,439 bytes is the mission buffer.** That number is the hard ceiling on how large any single San
Andreas mission can be without the loader being patched — a fact that follows directly from the field's
role and is worth knowing before designing a mission mod.

## 3. The external script table

```c
struct ExternalSegment {         // segment 4, ID = 2
    uint32_t largestExternal;    // 35,122
    uint32_t numExternals;       // 79
    struct {
        char     name[20];
        uint32_t base;           // virtual address
        uint32_t size;
    } script[79];                // 28 bytes each
};
```

Size check: `1 + 8 + 79 × 28 = 2,221`, body is **2,221**. ✅ Exact.

| Check | Result |
|---|---:|
| `largestExternal` == largest declared `size` | **35,122 == 35,122** ✅ |
| First `base` == `len(main.scm)` | **3,079,744 == 3,079,744** ✅ |
| `base[i] + size[i] == base[i+1]` | **77 / 77 consecutive pairs** ✅ |

### The virtual address space

The first external script's base address is **exactly the length of `main.scm`**, and the bases form an
unbroken running sum from there:

```
main.scm            0 … 3,079,744
PLAYER_PARACHUTE    3,079,744 + 5,631
PARACHUTE           3,085,375 + 7,354
BCESAR2             3,092,729 + 6,383
   …
HOTDOG              3,577,506 +   725  →  3,578,231
```

✅ 77 of 77 consecutive pairs chain. Total: `3,079,744 + 498,487 = 3,578,231`.

**San Andreas gives every script a unique address in one flat space, even though the scripts live in two
different files.** `main.scm` occupies the bottom; the 79 streamed scripts are laid end to end above it in
table order.

🟡 *Reasoned:* that is what makes a single 32-bit value sufficient to identify a jump target anywhere in
the game's logic, regardless of which file the target happens to be stored in. The alternative — a
(file, offset) pair — would have doubled the width of every branch.

The addresses are **fictional in the sense that nothing is ever laid out that way on disk**: `script.img`
stores each script sector-aligned in its own slot ([C1.1](../C1-Streaming/01-img-ver2-archive-model.md)),
and the base field is a coordinate, not a location.

### ⚠️ The `AAA` outlier

One record breaks the chain: `("AAA", base = 0, size = 8)`, last in the table. Base 0 sits *inside*
`main.scm`, which no real external script can. Combined with the finding that `aaa.scm`'s eight bytes are
byte-identical to segment 6 ([C18.1 §5](01-the-segment-chain.md)), it is a **build artefact occupying a
table slot** rather than a script.

It is also why the chain check above is stated as 77 of 77 pairs across 78 records rather than 78 of 78
across 79 — the outlier is excluded *and named*, not quietly dropped.

## 4. The tables agree with `script.img`

| Direction | Result |
|---|---:|
| Segment 4 names → `script.img` entries | **79 / 79** ✅ |
| `script.img` entries → segment 4 names | **79 / 79** ✅ |
| Declared `size` fits inside its sector-aligned slot | **79 / 79** ✅ |

✅ **A bijection in both directions**, with every declared size consistent with the archive slot that
holds it.

This is the same class of evidence as `areaId` reproducing its filename in
[C12.1 §3](../C12-Path-Network/01-nodes-dat.md) and `packageName` reproducing its filename in
[C17.1 §3](../C17-IFP-Animation/01-anp3-container.md): **a field that independently reproduces information
held in another file cannot be at the wrong offset.** Here the agreement is across two files in two
different formats, which is why it also confirms the 28-byte record width and the 20-byte name field
simultaneously.

## 5. What the tables say about the game

**135 missions and 79 ambient scripts.** The split is informative: the 135 are story content loaded one at
a time into a shared buffer; the 79 are the things that run *while* you walk around — `BARBER`, `TATTOO`,
`SLOT_MACHINE`, `FOOD_VENDOR`, `GYMBENCH`, `VENDING_MACHINE`, `STRIP_AMBIENCE`.

🟡 *Reasoned:* the external mechanism exists because those scripts are **many, small and independently
resident**, which is the opposite of a mission's profile. Streaming them individually costs a table entry
each and saves the mission buffer entirely.

The largest external is 35,122 bytes against a largest mission of 68,439 — so an ambient script can be
half the size of the biggest mission in the game, which is a larger budget than the word "ambient"
suggests.

---

### Key takeaways

- **135 missions tile `[194125, 3079744)` exactly** — offsets strictly increasing, sizes summing to the
  span with no gap or overlap.
- ✅ `mainSize` and `largestMission` both match values **derived from the offset table**, not merely
  restated from it.
- **`largestMission` = 68,439 is the mission buffer size** — the hard ceiling on a single mission.
- **79 external scripts share one flat virtual address space with `main.scm`**, beginning at exactly
  `len(main.scm)` and chaining across **77 / 77** consecutive pairs.
- ⚠️ The `AAA` record (`base 0`, `size 8`) is the **build artefact from
  [C18.1 §5](01-the-segment-chain.md)** occupying a table slot.
- ✅ **79 / 79 names match `script.img` in both directions**, confirming the 28-byte record and 20-byte
  name field at the same time.
- ⏳ Segment 3's fourth dword (**964**) is unnamed.

**Continue:** [C18.3 — The code stream](03-the-code-stream.md)
