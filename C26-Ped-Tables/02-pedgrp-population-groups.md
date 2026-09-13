# C26.2 — `pedgrp.dat`: the Population Groups

> **The one-sentence version:** `pedgrp.dat` is 57 spawn groups of ped models, every one of the 185 models
> resolving against C14's ped table, tagged with 18 popcycle group labels — and the engine reads at most 21
> models per group, not the 32 the file's own header claims.

[← C26.1 — The acquaintance matrix](01-ped-dat-acquaintances.md) · [Chapter 26 hub](C26-Ped-Tables.md)

**Confidence:** ✅ Verified (the group grammar, the 185/185 cross-check, the executable's 21-cap and array,
the popcycle labels) / ⏳ (the per-member spawn weighting)

---

## 1. Comma-separated spawn groups

Each non-comment line of `pedgrp.dat` is one population group: a comma-separated list of ped **model**
names, followed by a comment naming the popcycle group it serves.

```
HMOGAR, WMYCON, WMYCONB, WMYMECH, WMYSGRD      # POPCYCLE_GROUP_WORKERS
WMYCON, WMYCONB, WMYSGRD, SWMOCD, HMOGAR       # POPCYCLE_GROUP_WORKERS (SF)
```

There are **57** such groups. Their sizes vary from a single model to 21, and the popcycle comments show why
there are more groups than group *types*: a type like `WORKERS` appears several times, once per city
(`(SF)`, `(VEGAS)`), so the streaming system can spawn locally appropriate crowds. Eighteen distinct
`POPCYCLE_GROUP_*` labels appear across the rows — `WORKERS`, `BUSINESS`, `CRIMINALS`, `BEACHFOLK`,
`GOLFERS`, `PROSTITUTES`, and so on — tying this file to the popcycle system of
[C16](../C16-Popcycle-And-Water/C16-Popcycle-And-Water.md).

✅ *Verified:* 57 groups, tagged with 18 distinct popcycle group labels.

## 2. 185 / 185 — every model is a real ped

A spawn group is only meaningful if the models it names exist. Collecting every distinct model across all 57
groups gives 185 names, and each one is checked against `peds.ide`, the ped table
[C14.1](../C14-Peds-And-Weapons/01-the-ped-table.md) decoded to **276** models:

```
distinct pedgrp models     :  185
defined in peds.ide (C14)  :  276
pedgrp models that resolve :  185 / 185     (no dangling reference)
```

Every model a population group can spawn is one the game actually defines. This is the cross-check that
validates `pedgrp.dat` — it cannot be checked against `ped.dat` (different name space entirely), so it is
checked against C14's ped table, and it comes back complete. The 91 peds in `peds.ide` that no group names
are the special and mission characters that the ambient population is not meant to spawn.

✅ *Verified:* all 185 population-group models are defined in C14's 276-model ped table.

## 3. The engine reads 21, the header says 32

The file's header instructs authors that "each ped group should contain 32 ped type names". The engine
disagrees. The loader at VA `0x5BD040` walks a group's comma-list and stops at 21:

```asm
005BD105  mov  word ptr [ecx*2 + 0xC0F358], dx   ; store this model's uint16 index
005BD10D  inc  ebx
005BD10E  inc  ebp
005BD10F  cmp  ebp, 0x15                          ; 0x15 = 21
005BD116  jl   0x5BD085                           ; ...read at most 21 per group
```

`cmp ebp, 0x15` bounds the per-group loop at **21**, and each accepted model is written as a `uint16` index
into the group array based at `0xC0F358`. The data agrees with the code, not the comment: the largest group
in the file has exactly **21** members, never 32. So the header's "32" is stale documentation — the real
capacity is 21, proven twice over (the loop bound and the data's own maximum).

This is the mirror image of [C25](../C25-Object-Physics/C25-Object-Physics.md)'s object mass cap: there the
data overflowed a limit the header stated; here the header overstates a limit the code enforces. Either way
the executable, not the comment, is the authority.

✅ *Verified:* the engine caps a group at **21** models (`cmp ebp, 0x15`); the header's 32 is wrong.

The same loader loads `CARGRP.DAT` immediately afterward with identical code — vehicle groups are the
vehicle-side twin of this file, and belong to [C13](../C13-Vehicle-Data/C13-Vehicle-Data.md)'s subject
rather than this one. Their sharing one routine is why both files use the same comma grammar and the same
21-cap.

## 4. What is not derived

⏳ The **per-member weighting** is not decoded. Groups routinely repeat a model — a business crowd names
`HMYRI` and `HFYRI` several times each — which is plainly a way to make some models more common than others
in the spawn. The multiplicity is visible in the data and its intent is clear, but exactly how the
population sampler turns a 21-slot list with repeats into spawn probabilities is population-system behaviour
this chapter does not trace. The **membership** of each group is ✅; the **weighting** is ⏳.

---

### Key takeaways

- ✅ `pedgrp.dat` is **57** comma-separated population groups of ped models, tagged with **18** distinct
  `POPCYCLE_GROUP_*` labels (city variants explain why groups outnumber group types).
- ✅ **185 / 185** distinct group models resolve against C14's 276-model `peds.ide` table — no dangling
  reference.
- ⚠️ The engine caps a group at **21** members (`cmp ebp, 0x15` at `0x5BD10F`), storing `uint16` model
  indices at `0xC0F358`; the header's "should contain 32" is contradicted by both the code and the data.
- 🧭 The loader is **shared with `CARGRP.DAT`** — the vehicle-group twin (C13's subject).
- ⏳ The **spawn weighting** implied by repeated models is left to the population system.

**Continue:** [Chapter 26 hub](C26-Ped-Tables.md)
