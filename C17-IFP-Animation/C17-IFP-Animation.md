# Chapter 17 — IFP: the Animation Format

> **Goal of this chapter:** decode San Andreas's animation container end to end — 132 packages, 1,557
> animations, **868,337 keyframes** — with every file walking byte-exact, and a size field that
> validates the decode on all 1,557 animations.

**Subsystem category:** Animation / asset format
**Depends on:** [C1.1 — The IMG VER2 archive model](../C1-Streaming/01-img-ver2-archive-model.md)
**Ties:** [C1](../C1-Streaming/C1-Streaming.md), [C4](../C4-Entities-And-Pools/C4-Entities-And-Pools.md), [C10](../C10-2dEffect/C10-2dEffect.md), [C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md), [C12](../C12-Path-Network/C12-Path-Network.md), [C13](../C13-Vehicle-Data/C13-Vehicle-Data.md), [C14](../C14-Peds-And-Weapons/C14-Peds-And-Weapons.md), [C15](../C15-Timecycle/C15-Timecycle.md)
**RE status:** Verified
**Confidence:** ✅ Verified over the full population

---

## Deep-dive pages

- [C17.1 — The ANP3 container](01-anp3-container.md): three nested 36-byte headers, and a name field
  that reproduces its own filename.
- [C17.2 — Frames](02-frames.md): two frame types, 10 and 16 bytes, and the size field that proves them.
- [C17.3 — The skeleton](03-the-skeleton.md): 262 bone names, a 26-bone standard rig, and root motion.
- [C17.4 — The bone-ID table](04-the-bone-id-table.md): the canonical rig derived, and IDs banded by limb.

---

## 17.1 The result first

Every claim in this chapter rests on one check:

| Measurement | Value |
|---|---:|
| IFP packages in `anim.img` | **132** |
| Files walking without error | **132 / 132** |
| **Walks landing exactly on `8 + fileSize`** | **132 / 132** |
| Animations | **1,557** |
| Object (bone) tracks | **38,825** |
| Keyframes | **868,337** |

✅ **Zero exceptions.** A format with three levels of nesting, two frame sizes, and 868,337 records
walks to the declared byte on every one of 132 files.

## 17.2 The structure

```
ANP3 file
├── header 36 B          'ANP3' · fileSize · packageName[24] · numAnimations
└── animation × N
    ├── header 36 B      name[24] · numObjects · frameBytes · 1
    └── object × M
        ├── header 36 B  name[24] · frameType · numFrames · boneId
        └── frame × F    10 B (type 3) or 16 B (type 4)
```

**Three nested headers, all exactly 36 bytes, all `name[24]` + three dwords.** That regularity is the
format's whole design, and it is what makes the walk trivial once the frame sizes are known.

## 17.3 Two self-validating fields

The decode is confirmed by two fields that had no obligation to agree with it:

**The package name reproduces the filename.** `airport.ifp` carries `AIRPORT`, `bar.ifp` carries `BAR`.
✅ **132 / 132** match case-insensitively — the same class of evidence as `areaId` reproducing its
filename in [C12.1 §3](../C12-Path-Network/01-nodes-dat.md).

**The animation header declares its own frame-byte total.** The third dword of each animation header
equals the sum of `frameSize × numFrames` across that animation's objects:

| Hypothesis | Matches |
|---|---:|
| `d1 == total frame bytes` | **1,557 / 1,557** |
| `d1 == frame bytes + 36 per object` | 0 |

✅ Exact on every animation. **That field is the proof the frame sizes are right** — get either wrong
and the sum diverges immediately ([C17.2 §3](02-frames.md)).

## 17.4 Two frame types

| Type | Size | Contents | Tracks |
|---:|---:|---|---:|
| **3** | **10 B** | rotation `int16[4]` + time `uint16` | 35,791 |
| **4** | **16 B** | rotation `int16[4]` + time `uint16` + translation `int16[3]` | 3,034 |

✅ Verified. Only two types exist across all 38,825 tracks.

**Type 4 is root motion.** It appears as the *first* object in **1,336 of 1,557 animations (85.8 %)**,
and 1,146 animations contain exactly one type-4 track. Everything else is rotation-only — the standard
skeletal-animation economy where only the root translates and the rest of the skeleton just rotates.

## 17.5 The skeleton

**262 distinct bone names**, and the naming gives away the tool chain:

```
Bip01 L Clavicle    Bip01 R Clavicle    R UpperArm    Head    R Hand    R Finger
```

`Bip01` is 3ds Max's Biped rig prefix. Combined with the literal string **`3dsmax5`** found inside an
animation name field ([C17.1 §4](01-anp3-container.md)), the export pipeline identifies itself twice.

**26 objects per animation is the dominant shape** — 1,075 of 1,557 animations, the standard character
rig. Details in [C17.3](03-the-skeleton.md).

## 17.6 What remains

⏳ **Open:** the rotation quantisation. Frames store four `int16` values for rotation, which is a
quaternion at some fixed scale, but the divisor was not derived — that needs either the decoder in the
executable or a reconstruction check against a known pose.

✅ **`boneId` numbering is now closed** in [C17.4](04-the-bone-id-table.md): the canonical 26-bone rig is
tabulated, and the IDs are **banded by limb** (0–5 spine, 21–26 right arm, 31–36 left arm, 41–44 left
leg, 51–54 right leg). It is a **per-rig slot number, not a global identifier**.

Also open: the time unit. Observed deltas between consecutive frames were 4, 6, 24 and 32, so the step is
variable rather than fixed.

---

### Key takeaways

- **132 / 132 IFP files walk exactly to `8 + fileSize`** — 1,557 animations, 38,825 tracks, **868,337
  keyframes**, zero exceptions.
- The format is **three nested 36-byte headers**, each `name[24]` plus three dwords.
- **The package name reproduces the filename on 132/132**, and **the animation header's frame-byte total
  matches the computed sum on 1,557/1,557** — two independent self-validations.
- **Two frame types only**: 10 bytes (rotation + time) and 16 bytes (+ translation).
- **Type 4 is root motion** — first object in 85.8 % of animations, exactly one per animation in 73.6 %.
- **262 bone names** in 3ds Max `Bip01` Biped convention; **26 objects is the standard rig** (69 % of
  animations).
- ✅ **`boneId` closed** in [C17.4](04-the-bone-id-table.md) — canonical rig tabulated, IDs **banded by
  limb**, and the field is per-rig rather than global.
- ⏳ Rotation quantisation and the time unit remain underived.

**Next:** [C17.1 — The ANP3 container](01-anp3-container.md) · next chapter:
[C18 — SCM: the mission script format](../C18-SCM-Script/C18-SCM-Script.md)

## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Loaded / consumed by:** [C1](../C1-Streaming/C1-Streaming.md), [C4](../C4-Entities-And-Pools/C4-Entities-And-Pools.md), [C10](../C10-2dEffect/C10-2dEffect.md), [C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md), [C12](../C12-Path-Network/C12-Path-Network.md), [C13](../C13-Vehicle-Data/C13-Vehicle-Data.md)
- **Known bugs / gotchas:** anim group mismatch (C48 animgrp) yields wrong locomotion clips.
- **Modding:** ped.ifp + animgrp.dat drive custom animations; skinning is C44.
- **Performance:** IFP decoded at load; blend per animated ped.
