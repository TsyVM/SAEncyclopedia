# C17.3 — The Skeleton

> **The one-sentence version:** 262 bone names in 3ds Max Biped convention, a 26-bone rig used by 69 %
> of the game's animations, and a `boneId` field that is emphatically *not* the object index.

[← C17.2 — Frames](02-frames.md) · [Chapter 17 hub](C17-IFP-Animation.md)

**Confidence:** ✅ Verified (census) / 🟡 (rig interpretation) / ⏳ (`boneId` numbering)

---

## 1. The standard rig is 26 bones

| Objects per animation | Animations |
|---:|---:|
| **26** | **1,075 (69.0 %)** |
| 32 | 148 |
| 16 | 66 |
| 15 | 38 |
| 1 | 34 |
| 22 | 30 |

✅ *Verified.*

**More than two-thirds of every animation in San Andreas targets the same 26-bone skeleton.** That is
what makes animations interchangeable between characters — one rig, 1,075 animations, any of them
playable on any ped that uses it.

🟡 *Reasoned:* the 32-object group is a richer rig (🟡 plausibly with extra finger or facial bones), and
the 34 single-object animations are non-character motions — a door, a gate, a machine part. The 1-object
case is the clearest: it cannot be a skeleton.

## 2. The names identify the tool chain

Most frequent object names across all 38,825 tracks:

| Bone | Tracks |
|---|---:|
| `Bip01 L Clavicle` | 1,465 |
| `Bip01 R Clavicle` | 1,465 |
| `R UpperArm` | 1,465 |
| `Head` | 1,464 |
| `R Hand` | 1,463 |
| `R Finger` | 1,462 |
| `L UpperArm` | 1,460 |
| `L Hand` | 1,460 |

**262 distinct bone names** in total. ✅ Verified.

`Bip01` is **3ds Max's Biped** rig prefix — the default name its character-animation system assigns.
Combined with the literal `3dsmax5` string found in an animation name field
([C17.1 §4](01-anp3-container.md)), the pipeline identifies itself **twice by accident**: once through
a naming convention and once through uninitialised memory.

Neither was intended as documentation, which is what makes them reliable.

### The counts cluster tightly

The top eight bones appear in 1,460–1,465 tracks against 1,557 animations — a spread of just 5. 🟡
*Reasoned:* those bones are present in essentially every character animation, and the ~95 animations
missing them are the non-character motions from §1.

The near-identical `L Clavicle` and `R Clavicle` counts (1,465 each) confirm the rig is symmetric, which
is the sort of thing worth checking precisely because it would be surprising if it were not.

## 3. Naming is inconsistent

Three conventions coexist in the same files:

```
Bip01 L Clavicle        <- full Biped prefix
R UpperArm              <- prefix dropped
 Pelvis                 <- prefix dropped AND a leading space
Spine 1  /  Spine1      <- both spellings present
Root  /  normal         <- two names for the root object
```

⚠️ **Leading spaces are common and must be stripped**, not just NUL-truncated
([C17.1 §4](01-anp3-container.md)). And `Spine 1` versus `Spine1` means **a name-based bone lookup will
fail across packages** unless it normalises whitespace.

🟡 *Reasoned:* the inconsistency is hand-editing residue, the same as the four comma defects in
[C13](../C13-Vehicle-Data/C13-Vehicle-Data.md) and [C14](../C14-Peds-And-Weapons/C14-Peds-And-Weapons.md).
262 distinct names across a 26-bone standard rig is far more than the rig needs — much of that 262 is
spelling variation rather than genuinely distinct bones.

⏳ **Open:** how many of the 262 are true duplicates after normalisation. A cheap census, not run here,
and it would give the real bone count.

## 4. ⏳ `boneId` is not the object index

The third dword of each object header looked like a sequential index. It is not:

| Check | Result |
|---|---:|
| `boneId == object index` | **7,903 / 38,825 (20.4 %)** |

✅ Verified — and the 20.4 % is close to what coincidence would produce for small indices, so the field
is **independent of position**.

This is the opposite of the path-node case, where `nodeId` equalled the record index on **68,237 of
68,237** ([C12.1 §3](../C12-Path-Network/01-nodes-dat.md)). There, the match proved the field's meaning;
here, the *mismatch* proves it means something else.

🟡 *Reasoned:* it is a stable bone identifier — a hash or an enumerated ID that lets the runtime bind a
track to a skeleton bone **without string comparison**, which matters given §3's naming inconsistency.
A numeric ID sidesteps the `Spine 1` / `Spine1` problem entirely.

✅ **Closed in [C17.4](04-the-bone-id-table.md).** The correlation was run over all 38,825 tracks and
yields the canonical 26-bone rig with **IDs banded by limb** — 0–5 spine, 21–26 right arm, 31–36 left
arm, 41–44 left leg, 51–54 right leg.

⚠️ It also **revises the hypothesis above.** `boneId` is a **per-rig slot number, not a global
identifier**: names map to one ID 87.8 % of the time, but IDs map to one name only 34.7 % — low IDs are
reused by parachute, placeholder and character rigs alike. It cannot bind across rigs, only within one.
And **47 animations write `-1` throughout**, so name matching remains necessary as a fallback.

## 5. What this means for tooling

**Animations are portable.** 1,075 animations share one 26-bone rig, so retargeting between San Andreas
characters is a non-problem — which is why animation mods for this game are overwhelmingly
*replacements* rather than conversions.

**Bind by ID, not by name.** §3 shows names are inconsistent and §4 shows a numeric ID exists. A tool
matching on strings must normalise whitespace and handle both `Spine 1` and `Spine1`; one matching on
`boneId` does not — once the numbering is known.

**Do not trust the name field's tail.** Between the uninitialised buffer content
([C17.1 §4](01-anp3-container.md)) and the leading spaces, `b[off:off+24].split(b'\0')[0].strip()` is
the minimum safe read.

---

### Key takeaways

- **26 objects is the standard rig — 1,075 of 1,557 animations (69 %)**, which is why animations are
  interchangeable between characters.
- **262 distinct bone names**; the top eight appear in 1,460–1,465 tracks each, and the symmetric
  `L`/`R Clavicle` counts are identical at 1,465.
- **`Bip01` is 3ds Max Biped naming** — the pipeline identifies itself twice by accident, here and
  through the `3dsmax5` string in C17.1.
- ⚠️ Naming is **inconsistent**: leading spaces, dropped prefixes, and both `Spine 1` and `Spine1`. Any
  name-based lookup must normalise.
- ⏳ **`boneId` matches the object index only 20.4 % of the time** — it is a real identifier, not a
  position. The exact inverse of the path-node `nodeId` result, and the mismatch is what proves it.
- ✅ **Closed in [C17.4](04-the-bone-id-table.md)**: the rig is tabulated and IDs are **banded by limb** —
  but the field is **per-rig, not global**, which revises the hypothesis above.

**Continue:** [Chapter 17 hub](C17-IFP-Animation.md)
