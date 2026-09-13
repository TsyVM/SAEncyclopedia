# C5.6 — The Sector Arrays, Closed (and a Correction)

> **The one-sentence version:** both array bases are found and both extents prove themselves — and in
> the process the repeat grid turns out **not** to be a coarse partition of the map at all, which
> corrects C5.1 and C5.3.

[← C5.5 — Open: the sector arrays](05-open-the-sector-arrays.md) · [Chapter 5 hub](C5-CWorld.md)

**Confidence:** ✅ Verified
**Closes:** [C5.5](05-open-the-sector-arrays.md), [C5.1 §2](01-the-two-grids.md),
[C5.3 §5](03-repeat-sectors.md)
**Corrects:** [C5.1 §1](01-the-two-grids.md), [C5.3 §2](03-repeat-sectors.md)

---

## 1. The instructions that close it

`0x0054BA60`, following exactly the route C5.5 §4 proposed — read what the `× 120` site adds:

```
0054bb1b: imul ecx, ecx, 0x78              ; clamp(y,0,119) × 120
0054bb1e: add  ecx, edi                    ;   + clamp(x,0,119)
0054bb23: lea  ecx, [ecx*8 + 0xb7d0b8]     ; &ms_aSectors[flat]        stride 8

0054bb20: and  eax, 0xf                    ; RAW y & 15   <- note: unclamped
0054bb2a: and  edx, 0xf                    ; RAW x & 15
0054bb2d: shl  eax, 4                      ;   (y&15) × 16
0054bb37: add  eax, edx                    ;   + (x&15)
0054bb39: lea  ecx, [eax + eax*2]          ;   × 3
0054bb3c: lea  ecx, [ecx*4 + 0xb992b8]     ; &ms_aRepeatSectors[idx]   stride 12
```

✅ *Verified.*

| Array | Base | Stride | Count |
|---|---|---:|---:|
| `ms_aSectors` | **`0x00B7D0B8`** | **8** | 14,400 |
| `ms_aRepeatSectors` | **`0x00B992B8`** | **12** | 256 |

## 2. Both extents prove themselves

```
0x00B7D0B8 + 14400 × 8  = 0x00B992B8    ← exactly the repeat-sector base
0x00B992B8 +   256 × 12 = 0x00B99EB8    ← a referenced global (12 xrefs)
```

The two arrays are **adjacent**: the fine grid ends precisely where the coarse grid begins, and the
coarse grid ends precisely on the next known object. Reference counts confirm all three addresses are
real (`0xB7D0B8` 76, `0xB992B8` 78, `0xB99EB8` 12).

This is the fourth use of the technique from [C1.2 §2](../C1-Streaming/02-cdstream-layer.md) — capacity
verified by the array's computed end landing on its neighbour — and the cleanest instance yet, because
here the neighbour is the *other* array in the same chapter.

## 3. The record sizes settle the duty split

Stride 8 = **two** 4-byte list heads. Stride 12 = **three**.

That confirms — and promotes from 🟡 to ✅ — the inference in [C5.1 §2](01-the-two-grids.md):

| Grid | Lists | Consistent with |
|---|---|---|
| `ms_aSectors` (8 B) | 2 | static entities: `Buildings` 13,000 + `Dummys` 2,500 |
| `ms_aRepeatSectors` (12 B) | 3 | moving entities: `Vehicles` 110 + `Peds` 140 + `Objects` 350 |

And it explains the pointer-node split ([C5.4 §5](04-ptrnode-coupling.md)): two static lists drawing
from `PtrNode Single` (70,000), three dynamic lists drawing from `PtrNode Double` (3,200), whose
element sizes are 8 and 12 bytes respectively ([C4.5 §4](../C4-Entities-And-Pools/05-the-cpool-object.md)).

✅🔷 **Closed** in [X1 §3.5](../X1-SDK-Cross-Reference/X1-SDK-Cross-Reference.md): `CSector` is
`{ CPtrListSingleLink m_buildingList; CPtrListDoubleLink m_dummyList; }` and `CRepeatSector` is
`CPtrListDoubleLink m_lists[3]` = vehicles, peds, objects. Buildings use single-link nodes; everything
that moves uses double-link nodes, because a mover must unlink in O(1).

## 4. ⚠️ Correction — the repeat grid is not a coarse partition

**[C5.1 §1](01-the-two-grids.md) and [C5.3 §2](03-repeat-sectors.md) describe the repeat grid as 16 × 16
cells of 375 units covering −3000 … +3000. That is wrong.**

The masks at `0x0054BB20` and `0x0054BB2A` are applied to the **raw sector indices**, not to world
coordinates, and **not** to the clamped values used for the fine grid. So:

```
repeatIndex = ((sectorY & 15) × 16) + (sectorX & 15)
```

Since a sector is 50 units, the repeat grid **tiles every 16 × 50 = 800 units** and repeats across the
map. It is a **hash of position, not a region of it**: 6000 / 800 = 7.5 tiles per axis, so
**7.5² = 56.25 distinct world cells share each of the 256 buckets**.

That is what "repeat" means, and it is what the mask — rather than a clamp — was telling us in
[C5.3 §1](03-repeat-sectors.md). The page correctly noted that masking wraps rather than clamps and
correctly guessed that this was the origin of the name; it then failed to follow that observation to
its conclusion and instead assumed the grid spanned the map.

**What went wrong, specifically:** C5.3 §2 derived 375 as `6000 / 16`, found a literal `375.0` in
`.rdata`, and treated the agreement as verification. The derivation assumed the conclusion — that the
grid spans the map — and the literal was **coincidental**. Two "independent" derivations that share a
false premise are not independent.

The negative result in [C5.3 §5](03-repeat-sectors.md) — that `1/375` does **not** appear in `.rdata` —
was the correct clue and was correctly recorded. It should have been weighted more heavily than the
coincidence.

### What stands and what falls

| Claim | Status |
|---|---|
| Fine grid: 120 × 120 × 50 units, −3000 … +3000 | ✅ **stands** — three independent confirmations |
| `sectorIndex = floor(c × 0.02 + 60)` | ✅ **stands** |
| Repeat grid is 16 × 16 | ✅ **stands** |
| Repeat cells are 375 units, spanning the map | ❌ **withdrawn** |
| Repeat grid is a **wrapping hash**, tiling every 800 units | ✅ **replaces it** |
| The grids "do not nest" ([C5.3 §3](03-repeat-sectors.md)) | ❌ **withdrawn** — they nest exactly, 16 sectors per tile |
| 375.0 at `0x0086555C` is the repeat cell size | ❌ **withdrawn** — unrelated constant |

## 5. Why the coarse grid still helps

The original rationale ([C5.3 §4](03-repeat-sectors.md)) — fewer re-links for moving entities — survives
in modified form, and is arguably better:

A moving entity changes repeat bucket only when it crosses an 800-unit boundary, so re-link churn drops
by 16× per axis versus the fine grid. And because the grid wraps, an entity leaving the map never falls
out of the array — the mask cannot produce an out-of-range index, which is why this path needs no
clamp while the fine-grid path does.

The trade is that a bucket may contain entities from up to 56 unrelated places in the world, so a query
must filter by actual distance. With only 250 movers total that is cheap; with 13,000 buildings it would
not be, which is exactly why static geometry uses the clamped, non-wrapping fine grid instead.

## 6. Remaining open

⏳ The population path is still unread, so the per-list class assignment in §3 stays 🟡. The other four
items C5.5 §5 listed are now closed or superseded:

| Item | Status |
|---|---|
| Which classes populate which grid | 🟡 strongly constrained by 2-vs-3 list widths |
| Whether every path clamps | ✅ answered — the fine path clamps, the repeat path **masks and needs no clamp** |
| Repeat pre-mask arithmetic | ✅ closed — it masks the sector index; there is no float math |
| Single vs double node split | ✅ closed — 8 B / 12 B element sizes match 2-list / 3-list records |
| Measured average sector span | ⏳ still open — needs IPL parsing |

---

### Key takeaways

- **`ms_aSectors` = `0x00B7D0B8`, stride 8, 14,400 entries. `ms_aRepeatSectors` = `0x00B992B8`,
  stride 12, 256 entries.**
- The arrays are **adjacent**, and each extent lands exactly on the next known object — the strongest
  form of this proof so far.
- Record widths (**2 lists vs 3**) promote the static/dynamic duty split to ✅ and explain the
  `PtrNode Single`/`Double` sizes.
- ⚠️ **Correction:** the repeat grid is a **wrapping hash tiling every 800 units**, not a 375-unit
  partition of the map. 56.25 world cells share each bucket.
- The failure mode is worth remembering: **a derivation that assumes its conclusion plus a coincidental
  constant is not two independent confirmations.** The recorded negative result was the better evidence.
- Masking is why the repeat path needs **no clamp** — an out-of-map entity wraps instead of overflowing.

**Continue:** [Chapter 5 hub](C5-CWorld.md) · next chapter: `C6 — Collision & the COL Model`
