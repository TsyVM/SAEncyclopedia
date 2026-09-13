# C26.1 — `ped.dat`: the Acquaintance Matrix

> **The one-sentence version:** `ped.dat` is a 17-type table of who Hates and who Respects whom — the data
> behind gang wars and cops-versus-criminals — using two of its four documented verbs, and every type name
> in it is a PEDTYPE the executable also knows.

[Chapter 26 hub](C26-Ped-Tables.md) · next: [C26.2 — population groups](02-pedgrp-population-groups.md)

**Confidence:** ✅ Verified (the block grammar, the verb set, the exe cross-check, the clique, the
asymmetries)

---

## 1. The block format

`ped.dat` is small — 1,276 bytes — and block-structured. An unindented line names a ped **type**; the
indented lines under it give that type's feelings, one verb followed by one or more target types:

```
COP
    Hate CRIMINAL DEALER
    Respect MEDIC FIREMAN COP
```

Seventeen types are defined, in file order: `CIVMALE`, `CIVFEMALE`, `COP`, `GANG1`…`GANG10`, `MEDIC`,
`FIREMAN`, `CRIMINAL`, `PROSTITUTE`. Each carries a `Hate` list, a `Respect` list, or both. That is the
whole grammar — a verb and a set of targets, read against the block's owning type.

✅ *Verified:* 17 PEDTYPE blocks, each a set of `<verb> <target>…` relations.

## 2. Two of four verbs

The header documents four acquaintance options — **Hate, Dislike, Like, Respect** — a graded scale from
hostile to friendly. The data uses only the two extremes:

```
verbs documented :  Hate  Dislike  Like  Respect
verbs used       :  Hate                 Respect
```

`Dislike` and `Like` are never used by any of the 17 types. The relationship model the engine supports is
finer than the relationships the shipped data expresses — the same capability-without-data pattern this
project keeps meeting: [C19](../C19-GXT-Text/C19-GXT-Text.md)'s five named languages with two shipped,
[C21](../C21-Particles/C21-Particles.md)'s three compiled-but-unused particle types,
[C23](../C23-Fonts-HUD/C23-Fonts-HUD.md)'s two absent mask atlases. Here two of the four relationship verbs
are wired into the format and never authored.

✅ *Verified:* only `Hate` and `Respect` are used; `Dislike` and `Like` are documented but absent.

## 3. Every name is a PEDTYPE in the executable

The type names are not free text — they are the engine's `PEDTYPE` enumeration, and the executable proves
it. Collecting every name that appears in `ped.dat`, whether as a block owner or as a target of some
relation, gives 18 distinct names (the 17 defined types plus `DEALER`, which is targeted but not defined —
§4). Every one of the 18 appears as a NUL-terminated string in `gta_sa.exe`:

```
ped.dat type/target names present in the exe  →  18 / 18
```

This is the same cross-subsystem check [C24](../C24-Surfaces/C24-Surfaces.md) used for surface names: a data
file and the compiled engine agreeing, name for name, on an enumeration. The acquaintance table speaks the
engine's own PEDTYPE vocabulary, which is why no external list is needed to interpret it.

✅ *Verified:* all 18 ped.dat names are PEDTYPE strings in the executable.

## 4. The gang clique, and two asymmetries

The ten gangs are the heart of the table, and they form a perfect **mutual-hatred clique**. Each `GANG`n
hates exactly the other nine gangs and respects only itself:

```
GANG1
    Hate GANG2 GANG3 GANG4 GANG5 GANG6 GANG7 GANG8 GANG9 GANG10
    Respect GANG1
```

Checked across all ten, the pattern is exact — every gang's `Hate` set is the other nine, every gang's
`Respect` list is itself alone. That symmetric structure is the data behind gang territory rivalry: any two
gangs are hostile, none allied.

Two **named asymmetries** sit outside the clique and are worth recording rather than smoothing over:

- **`DEALER` is targeted but never defined.** `COP` hates `CRIMINAL` and `DEALER`, but there is no `DEALER`
  block, so dealers have no acquaintances of their own. `DEALER` is a real PEDTYPE (it is in the exe, §3);
  the acquaintance file simply never gives it a row. Cops dislike dealers; dealers, per this file, feel
  nothing back.
- **`PROSTITUTE` is defined but never referenced.** It has a block (`Hate COP`), yet no other type mentions
  prostitutes. A one-directional entry — prostitutes fear the police, and nobody's feelings point back at
  them.

Neither is a parse error; both are shipped, deliberate-looking gaps in an otherwise symmetric table, and
the chapter names them as [C13](../C13-Vehicle-Data/C13-Vehicle-Data.md) and
[C25](../C25-Object-Physics/C25-Object-Physics.md) name their exceptions.

⚠️ *Named exceptions:* `DEALER` (referenced, no block); `PROSTITUTE` (block, unreferenced).

---

### Key takeaways

- ✅ `ped.dat` is a **17-type** block table of `Hate`/`Respect` relations — the data behind gang and
  police hostility.
- ✅ Only **two of four** documented verbs are used (`Hate`, `Respect`); `Dislike` and `Like` are a
  capability without data.
- ✅ All **18** type/target names are **PEDTYPE strings in the executable** — the file speaks the engine's
  enumeration.
- ✅ `GANG1…GANG10` form a **mutual-hatred clique** (each hates the other nine, respects itself).
- ⚠️ `DEALER` is referenced without a block; `PROSTITUTE` has a block nobody references — two named
  asymmetries.

**Continue:** [C26.2 — `pedgrp.dat`: the population groups](02-pedgrp-population-groups.md)
