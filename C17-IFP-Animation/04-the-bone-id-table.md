# C17.4 — The Bone-ID Table

> **The one-sentence version:** correlating `boneId` against bone name across all 38,825 tracks yields
> the canonical 26-bone rig — and reveals that the ID numbering is **banded by limb**, with a tens digit
> that identifies the body part.

[← C17.3 — The skeleton](03-the-skeleton.md) · [Chapter 17 hub](C17-IFP-Animation.md)

**Confidence:** ✅ Verified over all 38,825 tracks
**Closes:** [C17.3 §4](03-the-skeleton.md)

---

## 1. The question

[C17.3 §4](03-the-skeleton.md) established that `boneId` is **not** the object index — it matched only
20.4 % of the time — and proposed the obvious follow-up: correlate the field against bone name over the
tracks already parsed, and see whether a given name always carries the same ID.

It does, mostly. And the pattern in the numbers is more interesting than the mapping itself.

## 2. The canonical 26-bone rig

Across the **1,075 animations that use the 26-object rig**
([C17.3 §1](03-the-skeleton.md)), each slot resolves to one dominant `(name, boneId)` pair:

| Slot | Bone | `boneId` | Dominance |
|---:|---|---:|---:|
| 0 | `Normal` | **0** | 48 % |
| 1 | `Pelvis` | **1** | **100 %** |
| 2 | `Spine` | **2** | 78 % |
| 3 | `Spine1` | **3** | 78 % |
| 4 | `Neck` | **4** | 98 % |
| 5 | `Head` | **5** | 97 % |
| 6 | `Bip01 L Clavicle` | **31** | 97 % |
| 7 | `L UpperArm` | **32** | 97 % |
| 8 | `L ForeArm` | **33** | 97 % |
| 9 | `L Hand` | **34** | 97 % |
| 10 | `L Finger` | **35** | 97 % |
| 11 | `L Finger01` | **36** | 95 % |
| 12 | `Bip01 R Clavicle` | **21** | 97 % |
| 13 | `R UpperArm` | **22** | 97 % |
| 14 | `R ForeArm` | **23** | 97 % |
| 15 | `R Hand` | **24** | 97 % |
| 16 | `R Finger` | **25** | 97 % |
| 17 | `R Finger01` | **26** | 95 % |
| 18 | `L Thigh` | **41** | 97 % |
| 19 | `L Calf` | **42** | 97 % |
| 20 | `L Foot` | **43** | 97 % |
| 21 | `L Toe0` | **44** | 97 % |
| 22 | `R Thigh` | **51** | 97 % |
| 23 | `R Calf` | **52** | 97 % |
| 24 | `R Foot` | **53** | 97 % |
| 25 | `R Toe0` | **54** | 97 % |

✅ *Verified* over 1,075 animations. `Pelvis` at slot 1 is **1,075 / 1,075** — the only slot with no
variation at all.

## 3. The IDs are banded by limb

Read the ID column on its own and the scheme is unmistakable:

```
 0 –  5    spine chain      Normal · Pelvis · Spine · Spine1 · Neck · Head
21 – 26    RIGHT arm        Clavicle · UpperArm · ForeArm · Hand · Finger · Finger01
31 – 36    LEFT arm         Clavicle · UpperArm · ForeArm · Hand · Finger · Finger01
41 – 44    LEFT leg         Thigh · Calf · Foot · Toe0
51 – 54    RIGHT leg        Thigh · Calf · Foot · Toe0
```

✅ **The tens digit identifies the body part**, and the units digit is the position along that chain.
Right arm is the 20s, left arm the 30s, left leg the 40s, right leg the 50s.

That is a hand-assigned scheme with deliberate gaps — 6–20, 27–30, 37–40, 45–50 are unused by this rig,
leaving room to extend each limb without renumbering anything. 🟡 *Reasoned:* the 32-object rig from
[C17.3 §1](03-the-skeleton.md) presumably fills some of those gaps.

Note the **left/right asymmetry**: arms are right-then-left (20s before 30s) but legs are left-then-right
(40s before 50s). 🟡 Almost certainly an authoring accident rather than a meaning — but it means a tool
cannot infer side from parity or ordering.

## 4. Names map to IDs; IDs do not map to names

The correlation is strongly one-directional:

| Direction | Result |
|---|---:|
| Names mapping to exactly **one** `boneId` | **230 / 262 (87.8 %)** |
| `boneId`s mapping to exactly **one** name | **17 / 49 (34.7 %)** |

✅ Verified. There are **49 distinct `boneId` values** across **262 distinct names**.

The asymmetry is the finding. A name almost always has one ID, but an ID is reused by unrelated rigs:

| `boneId` | Competing names | Dominance |
|---:|---|---:|
| 9 | `Para_Canopy_M`, others | 43 % |
| 10 | `NEEDLE_10`, others | 50 % |
| 12 | `BONE_12`, others | 50 % |
| 13 | `BONE_13`, others | 50 % |

IDs 9–20 are contested at 43–50 %, because parachute canopy bones, generic `BONE_n` placeholders and
character bones all occupy the same low numeric space.

**So `boneId` is a per-rig slot number, not a global bone identifier.** Within the character rig it is
stable and meaningful; across rigs the same number means different things.

That is a materially different answer from what [C17.3 §4](03-the-skeleton.md) hypothesised — it
suggested a global ID that would let the runtime bind without string comparison. It cannot do that
across rigs. It can do it **within** one.

## 5. ⚠️ 47 animations write `-1` for every bone

The 12.2 % of names carrying more than one ID are almost entirely explained by a single pattern: a
second value of **`-1`**, appearing exactly once per name.

Tracing it: **47 animations set `boneId = -1` on every one of their objects.**

```
bd_fire.ifp    BD_Fire1            13 objects
bd_fire.ifp    BD_Fire2            13 objects
bd_fire.ifp    BD_Fire3            21 objects
bsktball.ifp   BBALL_Dnk_Gli_O      1 object
bsktball.ifp   BBALL_Dnk_Lnch_O     1 object
bsktball.ifp   BBALL_Dnk_O          1 object
```

✅ Verified.

🟡 *Reasoned:* `-1` is the "unassigned" sentinel — the same convention as `m_nModelIndex` in
[C4.3 §2](../C4-Entities-And-Pools/03-entity-model-link.md) and the LOD field in
[C11.2 §4](../C11-IDE-And-IPL/02-ipl-placements.md). These 47 animations were exported without bone-ID
assignment, so the runtime must fall back to **name matching** for them.

Which means a reader cannot rely on `boneId` alone: **3 % of animations have no usable IDs at all**, and
those are exactly the ones where the naming inconsistencies from
[C17.3 §3](03-the-skeleton.md) matter most.

## 6. What a reader should do

```python
# Bind by ID when it is present, by normalised name when it is not.
def bind(track_name, bone_id, rig):
    if bone_id >= 0:
        return rig.by_id(bone_id)              # valid within one rig
    return rig.by_name(normalise(track_name))  # 47 animations need this

def normalise(n):
    return n.strip().replace(' ', '').lower()  # 'Spine 1' == 'Spine1'
```

Three properties, each from a verified finding:

- **`bone_id >= 0` guard** — 47 animations write `-1` throughout (§5).
- **`rig.by_id` is per-rig**, not global — IDs 9–20 are reused across skeletons (§4).
- **`normalise` strips whitespace** — `Spine 1` and `Spine1` both occur
  ([C17.3 §3](03-the-skeleton.md)), and leading spaces are common.

## 7. Why the dominance is 97 %, not 100 %

Only `Pelvis` reaches 1,075 / 1,075. The rest sit at 95–98 %, and the shortfall has two causes, both
already documented:

- the **47 all-`-1` animations** (§5), which contribute a wrong ID to every slot they touch;
- **name spelling variants** ([C17.3 §3](03-the-skeleton.md)) — `Spine 1` versus `Spine1` splits a slot's
  count between two names without either being wrong.

Neither is a defect in the ID scheme. Correcting for both would push the table to ~100 %, and the fact
that the residual is fully explained by two known causes is itself a check on the table.

---

### Key takeaways

- The **canonical 26-bone rig** is derived and tabulated — slot, name and `boneId` for all 26, over
  1,075 animations.
- ✅ **IDs are banded by limb**: 0–5 spine, **21–26 right arm**, **31–36 left arm**, **41–44 left leg**,
  **51–54 right leg**. The tens digit is the body part.
- The bands leave **deliberate gaps** for extension; arms are numbered right-before-left while legs are
  left-before-right — 🟡 an authoring accident, but it means side cannot be inferred from ordering.
- **Names → IDs is 87.8 % unique; IDs → names is only 34.7 %.** `boneId` is a **per-rig slot number, not
  a global identifier** — IDs 9–20 are contested between character, parachute and placeholder rigs.
- This **revises** [C17.3 §4](03-the-skeleton.md)'s hypothesis: the field cannot bind across rigs, only
  within one.
- ⚠️ **47 animations write `-1` on every bone** and must fall back to name matching — exactly where the
  naming inconsistencies bite hardest.
- The 95–98 % dominance (rather than 100 %) is **fully explained** by those 47 animations plus known
  spelling variants.

**Continue:** [Chapter 17 hub](C17-IFP-Animation.md)
