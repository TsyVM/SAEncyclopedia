# C39.3 — Sixty-six camera modes

One `CCam` object ([C39.2](02-the-ccam-object.md)) produces the entire game's worth of camera behaviour by
reading a single field, `m_nMode`, and branching. That field is an `eCamMode` — a **`uint16` enum with 66
entries**, numbered `0` through `65` with no gaps. This page lays the enum out and groups it by what the
mode is for, so the switch is legible.

## The count

`m_nMode` is `uint16` and the enum runs `MODE_NONE = 0` … `MODE_AIMWEAPON_ATTACHED = 65` — **66 values,
contiguous**. That contiguity matters for reversing the dispatch: the per-frame update can (and does) use
`m_nMode` as a dense switch index, so the mode number *is* the branch selector. The enum itself is a
source-level artifact recovered from `gta-reversed`; individual mode numbers surface in the exe as
immediates in comparisons (e.g. the first-person check against `0x10` = `MODE_1STPERSON`), which is why the
count is recorded **🟡** — corroborated by immediates, fixed by the enum — rather than claimed as a table
read from the binary.

## The modes, grouped by purpose

**Baseline / gameplay (on foot & default)**

| # | Mode | Use |
|---:|---|---|
| 0 | `MODE_NONE` | no camera / uninitialised |
| 1 | `MODE_TOPDOWN` | top-down |
| 2 | `MODE_GTACLASSIC` | classic top-down (nostalgia mode) |
| 4 | `MODE_FOLLOWPED` | the default on-foot chase camera |
| 20 | `MODE_FOLLOW_PED_WITH_BIND` | follow-ped variant |
| 37 | `MODE_TOP_DOWN_PED` | top-down on foot |

**Vehicles**

| # | Mode | Use |
|---:|---|---|
| 3 | `MODE_BEHINDCAR` | behind the car |
| 18 | `MODE_CAM_ON_A_STRING` | the default driving camera ("on a string" behind the vehicle) |
| 22 | `MODE_BEHINDBOAT` | behind a boat |
| 14 | `MODE_WHEELCAM` | wheel/chase showcase |

**Aiming & weapons (first-person / zoomed)**

| # | Mode | Use |
|---:|---|---|
| 5 | `MODE_AIMING` | generic aim |
| 7 | `MODE_SNIPER` | sniper scope |
| 8 | `MODE_ROCKETLAUNCHER` | rocket launcher aim |
| 16 | `MODE_1STPERSON` | first-person |
| 34 | `MODE_M16_1STPERSON` | M16 iron-sights |
| 45 | `MODE_HELICANNON_1STPERSON` | heli cannon |
| 53 | `MODE_AIMWEAPON` | aim weapon |
| 55 | `MODE_AIMWEAPON_FROMCAR` | drive-by aim |
| 65 | `MODE_AIMWEAPON_ATTACHED` | attached-weapon aim (the last mode) |

**Cinematic, scripted & special set-pieces**

| # | Mode | Use |
|---:|---|---|
| 15 | `MODE_FIXED` | fixed script camera |
| 17 | `MODE_FLYBY` | scripted fly-by |
| 24–26 | `MODE_CAM_ON_TRAIN_ROOF`, `MODE_CAM_RUNNING_SIDE_TRAIN`, `MODE_BLOOD_ON_THE_TRACKS` | the train mission set-pieces |
| 32–33 | `MODE_ARRESTCAM_ONE/TWO` | the bust cutscene |
| 38 | `MODE_LIGHTHOUSE` | a specific scripted shot |
| 47 | `MODE_ATTACHCAM` | attach-to-entity scripted cam |

**Two-player**

| # | Mode | Use |
|---:|---|---|
| 48 | `MODE_TWOPLAYER` | co-op camera |
| 49 | `MODE_TWOPLAYER_IN_CAR_AND_SHOOTING` | co-op drive-by |
| 50/54 | `MODE_TWOPLAYER_SEPARATE_CARS(_TOPDOWN)` | co-op split |

**"DW" — the deer-hunter / side-mission cameras (56–64)**

`MODE_DW_HELI_CHASE`, `MODE_DW_CAM_MAN`, `MODE_DW_BIRDY`, `MODE_DW_PLANE_SPOTTER`, `MODE_DW_DOG_FIGHT`,
`MODE_DW_FISH`, `MODE_DW_PLANECAM1/2/3` — a contiguous block of nine modes for the Las Venturas
"Vertical Bird" / spotter-style set-pieces and minigames. Their tight numbering (56–64) is a hint they were
added as a group late in development.

*(The list above is representative, not exhaustive; all 66 values `0..65` are enumerated in
[`camera_structure.json`](../RE-Data/data/camera_structure.json).)*

## What the modes tie to

Several mode families are the camera end of systems documented elsewhere:

- **`MODE_CAM_ON_A_STRING` / spline cams** follow curve data — the camera analogue of
  [C12](../C12-Path-Network/C12-Path-Network.md)'s path network (the `CamPathSplines` helper).
- **The aiming modes** (`MODE_SNIPER`, `MODE_M16_1STPERSON`, `MODE_AIMWEAPON*`) pair with the weapon records
  of [C14](../C14-Peds-And-Weapons/C14-Peds-And-Weapons.md) — the four scoped weapons C14.4 isolated are
  exactly the ones that drive the camera into a zoomed mode.
- **The cinematic / fixed modes** are what the mission scripts ([C18](../C18-SCM-Script/C18-SCM-Script.md))
  invoke through the camera SCM commands (`TakeControl`, `SET_CAMERA…`), and what the
  recorded/replay cameras of [C34](../C34-Vehicle-Recording/C34-Vehicle-Recording.md) /
  [C30](../C30-Gameplay-Managers/C30-Gameplay-Managers.md) drive during set-pieces.

## Open items

- ⏳ The exact **byte offset of `m_nMode`** within `CCam` (this pass proved the object size, not each field
  offset).
- ⏳ The **mode-dispatch table** in the per-frame `CCam` update — whether it is a dense `jmp [table + mode*4]`
  switch (the contiguity suggests it) or a chain of compares.
- ⏳ Which modes are **retail-dead** (declared but never entered), the camera analogue of C21's three unused
  particle types and C19's five-but-two languages.

## Key takeaways

- `m_nMode` is a `uint16` `eCamMode` with **66 contiguous values** (`0..65`), so the mode number doubles as a
  dense dispatch index.
- The modes group cleanly into gameplay, vehicle, aiming, cinematic/scripted, two-player, and the nine
  "DW" set-piece cameras (56–64).
- Mode families are the camera side of other chapters — spline cams tie [C12](../C12-Path-Network/C12-Path-Network.md),
  aiming ties [C14](../C14-Peds-And-Weapons/C14-Peds-And-Weapons.md), cinematic ties
  [C18](../C18-SCM-Script/C18-SCM-Script.md)/[C34](../C34-Vehicle-Recording/C34-Vehicle-Recording.md).

**Continue:** [back to the C39 hub →](C39-Camera.md) · or [C38 — Skybox & Clouds](../C38-Skybox-And-Clouds/C38-Skybox-And-Clouds.md), which already read this camera.
