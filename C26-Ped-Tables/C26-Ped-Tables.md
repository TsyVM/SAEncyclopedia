# Chapter 26 — The Ped Behaviour Tables: Acquaintances and Population Groups

> **Goal of this chapter:** decode the two ped data files [C14](../C14-Peds-And-Weapons/C14-Peds-And-Weapons.md)
> left untouched — `ped.dat`, the acquaintance matrix that says which ped types hate or respect which, and
> `pedgrp.dat`, the population groups that say which ped models spawn together — and tie both back to the
> executable and to C14's ped table. Two clean cross-checks anchor the chapter: **18 / 18** ped-type names
> are hard-coded in `gta_sa.exe`, and **185 / 185** population-group model names resolve against C14's 276
> peds.

**Subsystem category:** Peds / AI / population
**Depends on:** [C14 — Peds, Weapons & Stats](../C14-Peds-And-Weapons/C14-Peds-And-Weapons.md) (the 276 ped
models and `pedstats.dat`) · [C16 — Popcycle & Water](../C16-Popcycle-And-Water/C16-Popcycle-And-Water.md)
(the population groups the spawn cycle draws from)
**Ties:** [C13](../C13-Vehicle-Data/C13-Vehicle-Data.md), [C14](../C14-Peds-And-Weapons/C14-Peds-And-Weapons.md), [C16](../C16-Popcycle-And-Water/C16-Popcycle-And-Water.md), [C19](../C19-GXT-Text/C19-GXT-Text.md), [C21](../C21-Particles/C21-Particles.md), [C23](../C23-Fonts-HUD/C23-Fonts-HUD.md), [C24](../C24-Surfaces/C24-Surfaces.md), [C25](../C25-Object-Physics/C25-Object-Physics.md)
**RE status:** Verified
**Confidence:** ✅ Verified (both grammars, the two cross-checks, the executable's per-group cap, the named
asymmetries) / ⏳ (the runtime weighting of a group's members)

---

## Deep-dive pages

- [C26.3 — How acquaintances drive AI](03-how-acquaintances-drive-ai.md): the runtime chain from PEDTYPE → acquaintance lookup → CTask selection; gang wars, cop-criminal pursuit, the two unused verbs as a modding opportunity.
- [C26.4 — Modding the ped tables](04-modding-ped-tables.md): adding relationships, gang alliances, the 21-slot cap, repeats for spawn frequency, and popcycle integration.
- [C26.1 — `ped.dat`: the acquaintance matrix](01-ped-dat-acquaintances.md): 17 ped types, a Hate/Respect
  relation using two of the four documented verbs, the ten-gang mutual-hatred clique, and two named
  asymmetries — every type name confirmed against the executable's PEDTYPE strings.
- [C26.2 — `pedgrp.dat`: the population groups](02-pedgrp-population-groups.md): 57 spawn groups of ped
  models, all 185 resolving against C14's ped table, tagged with 18 popcycle group labels — and an
  engine capacity of **21** per group that contradicts the file's own "should contain 32".

---

## 26.1 The result first

| Claim | Evidence |
|---|---|
| `ped.dat` defines **17 PEDTYPE** relationship blocks | counted; `CIVMALE … PROSTITUTE` ✅ |
| It uses **two of four** documented verbs | `Hate` and `Respect`; `Dislike`/`Like` unused ✅ |
| Every type/target name is a **PEDTYPE in the exe** | **18 / 18** present as strings ✅ |
| `GANG1…GANG10` form a **mutual-hatred clique** | each Hates the other 9, Respects itself ✅ |
| Two **named asymmetries** | `DEALER` referenced, no block; `PROSTITUTE` block, unreferenced ✅ |
| `pedgrp.dat` has **57** spawn groups | counted ✅ |
| Every group model resolves in **C14's ped table** | **185 / 185** in `peds.ide` (276 models) ✅ |
| The engine caps a group at **21**, not 32 | `cmp ebp, 0x15` at `0x5BD10F`; data max is 21 ✅ |
| Groups carry **18** popcycle labels | `POPCYCLE_GROUP_*` in the row comments ✅ |

## 26.2 Two tables, two directions of the same subsystem

`ped.dat` and `pedgrp.dat` describe the ped population from opposite ends. `pedgrp.dat` decides **who
appears** — which ped *models* the streaming population spawns together in a given part of the map, keyed to
the popcycle groups of [C16](../C16-Popcycle-And-Water/C16-Popcycle-And-Water.md). `ped.dat` decides **how
they treat each other** once they exist — which ped *types* are hostile or friendly, the data behind gang
wars and the police hating criminals. One is about spawning models; the other is about the behaviour of
type classes. They never reference each other, and they operate on different name spaces (185 model names
vs 17 type names), which is exactly why each needs its own cross-check to a third party.

## 26.3 Why the two cross-checks matter

Neither file can be validated against the other, so the chapter validates each against an independent
authority — the same discipline [C24](../C24-Surfaces/C24-Surfaces.md) used for surfaces:

- `ped.dat`'s type names are checked against the **executable**. All 17 defined types and the one extra
  referenced name (`DEALER`) appear as hard-coded PEDTYPE strings in `gta_sa.exe` — **18 / 18**. The
  acquaintance file is speaking the engine's own enumeration, not an invented vocabulary.
- `pedgrp.dat`'s model names are checked against **C14's ped table**. Every one of the 185 distinct models
  a group can spawn is one of the 276 peds defined in `peds.ide` — **185 / 185**, no dangling reference.
  The population groups draw only from models the game actually has.

Two files, two independent authorities, two clean resolutions. Neither needs a community list to be pinned
down.

## 26.4 What this chapter does not claim

⏳ The **runtime weighting** inside a group is not derived. `pedgrp.dat` lists a group's members and often
repeats a model to make it more likely (a business group naming `HMYRI` several times), so the multiplicity
is clearly a frequency hint — but exactly how the spawn code samples the list is population-system behaviour
this chapter does not trace. The **membership** of every group is ✅; the sampling weights are ⏳.

The chapter also **closes a C14 loose end** in passing:
[C14.3](../C14-Peds-And-Weapons/03-stats-and-crosschecks.md) left the eleven `pedstats.dat` column meanings
undecoded; they are in fact documented by that file's own header (flee distance, heading-change rate, fear,
temper, lawfulness, sexiness, attack strength, defend weakness, shooting rate, decision-maker) and are
recorded there now.

---

### Key takeaways

- ✅ `ped.dat` is a **17-type acquaintance matrix** using **Hate** and **Respect** (two of four documented
  verbs); every type/target name is a **PEDTYPE string in the executable** — **18 / 18**.
- ✅ `GANG1…GANG10` form a **mutual-hatred clique**; `DEALER` is referenced without a block and
  `PROSTITUTE` has a block nobody references — two named asymmetries.
- ✅ `pedgrp.dat` is **57 population groups** of ped models, **185 / 185** resolving against C14's 276-model
  ped table, tagged with **18** `POPCYCLE_GROUP_*` labels.
- ⚠️ The engine caps a group at **21** members (`cmp ebp, 0x15`), storing `uint16` model indices at
  `0xC0F358` — the header's "should contain 32" is contradicted by the code, and the data's own maximum is
  21. The loader is shared with `CARGRP.DAT`.
- ⏳ The runtime **weighting** of a group's members (via repeats) is left to the population system.

**Continue:** [C26.1 — `ped.dat`: the acquaintance matrix](01-ped-dat-acquaintances.md)

## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Loaded / consumed by:** [C13](../C13-Vehicle-Data/C13-Vehicle-Data.md), [C14](../C14-Peds-And-Weapons/C14-Peds-And-Weapons.md), [C16](../C16-Popcycle-And-Water/C16-Popcycle-And-Water.md), [C19](../C19-GXT-Text/C19-GXT-Text.md), [C21](../C21-Particles/C21-Particles.md), [C23](../C23-Fonts-HUD/C23-Fonts-HUD.md)
- **Known bugs / gotchas:** engine ped-group cap is 21 not the file's 32 (silent truncation).
- **Modding:** ped.dat acquaintances + pedgrp populations tune gang/civilian behaviour.
- **Performance:** acquaintance lookup per ped decision.
