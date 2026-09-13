# Chapter 39 — The Camera: `CCamera`, the Three-Cam Array, and Sixty-Six Modes

> **Goal of this chapter:** structure the system that frames everything you see. San Andreas routes all
> viewing through a single global camera manager, **`TheCamera`** (a `CCamera` at `0xB6F028`), which owns a
> small fixed array of **`CCam`** camera objects — one active, one to interpolate from during a cut, one for
> debug. This chapter sizes both objects to the byte from `gta_sa.exe`, recovers the array's offset and the
> active-camera index that selects into it, and lays out the **66-entry** camera-mode enum that decides how
> each `CCam` behaves (follow-the-ped, aim-down-sniper, on-a-string, cinematic, two-player, cut-scene…).

**Subsystem category:** Rendering / view — the camera manager (`CCamera`) and camera object (`CCam`)
**Depends on:** [C28](../C28-Class-Catalogue/C28-Class-Catalogue.md) (the disassembly + `imul`-stride
method) · corroborated by the external `gta-reversed` reconstruction (names + the `VALIDATE_SIZE` assertions)
**Ties:** [C38](../C38-Skybox-And-Clouds/C38-Skybox-And-Clouds.md) (the sky renderer already read
`TheCamera`'s matrix and heading), [C12](../C12-Path-Network/C12-Path-Network.md) (spline / on-a-string
cams follow path data), [C34](../C34-Vehicle-Recording/C34-Vehicle-Recording.md) &
[C30](../C30-Gameplay-Managers/C30-Gameplay-Managers.md) (recorded / replay cameras)
**RE status:** Documented
**Confidence:** ✅ for `TheCamera` @`0xB6F028`, the **`CCam` stride `0x238`**, `m_aCams` @`+0x174`,
`m_nActiveCam` @`+0x59`, and the `GetActiveCam` indexing (all re-checked by `derive_camera.py`, 6/6) · 🟡 for
the full `CCamera` size **`0xD78`** (bracketed from the exe, taken exact from `gta-reversed`'s
`VALIDATE_SIZE`) and the 3-cam count
**Data artifact:** [`RE-Data/data/camera_structure.json`](../RE-Data/data/camera_structure.json) — generated
by [`tools/derive_camera.py`](../tools/derive_camera.py)

---

## Deep-dive pages

- [C39.1 — TheCamera singleton and the three-cam array](01-the-camera-singleton.md): the `CCamera` at
  `0xB6F028`, the `m_aCams[3]` array of `CCam` objects at `+0x174`, the `m_nActiveCam` selector at `+0x59`,
  and `GetActiveCam` = `m_aCams[m_nActiveCam]` read straight from the indexing code.
- [C39.2 — The CCam object](02-the-ccam-object.md): one camera is **`0x238` bytes** (proven by the `imul`
  that indexes the array 27 ways); its mode field and what a single `CCam` holds.
- [C39.3 — Sixty-six camera modes](03-sixty-six-modes.md): the `eCamMode` enum (`uint16`, `0..65`), from
  `MODE_FOLLOWPED` through the deer-hunter and two-player modes — the switch that gives one `CCam` its
  dozens of personalities.
- [C39.4 — Scripting and modding the camera](04-scripting-and-modding-the-camera.md): the **six SCM opcode families** (mode-switch, fixed-position, entity-attach, rail/spline, widescreen/fade, interpolation-control) with opcode IDs and write targets; **`CCamera::Process` at `0x50AE10`** as the hook point — post-return override vs. mode-intercept patterns; the **cutscene spline system** (keyframe-interpolated position/look-at/FOV written into the same `CCam` fields as gameplay); the **Vert- FOV formula** (why Hor+ breaks HUD/minimap scale assumptions); `m_fFarClipPlane` and IDE `draw_dist` as a co-bottleneck pair; the two-player FOV-widening and split-screen mode mechanism.

---

## 39.0 The result first

| Claim | Value | Tier | Evidence |
|---|---|:--:|---|
| Camera manager singleton | `TheCamera` @`0xB6F028` | ✅ | field cluster; the sky chapter ([C38](../C38-Skybox-And-Clouds/C38-Skybox-And-Clouds.md)) already read `+0x10`/`+0x14` |
| One camera object (`CCam`) | **`0x238`** (568 B) | ✅ | `imul reg,reg,0x238` at **27** sites in the camera code |
| Camera array `m_aCams` | at `+0x174`, **3** × `0x238` | ✅ / 🟡 | `add reg,0xB6F19C` after the multiply (5 sites); count 3 from `gta-reversed` |
| Active-camera index `m_nActiveCam` | byte at `+0x59` | ✅ | `movzx reg, byte [0xB6F081]` (13 reads) |
| `GetActiveCam` | `m_aCams[m_nActiveCam]` | ✅ | the exact `movzx / imul 0x238 / add 0xB6F19C` sequence |
| Camera manager size (`CCamera`) | **`0xD78`** (3448 B) | 🟡 | bracketed `0x81C ≤ 0xD78 < 0xE18` from the exe; exact from `gta-reversed` |
| Camera modes (`eCamMode`) | **66** (`0..65`) | 🟡 | `gta-reversed` enum; mode constants appear as immediates |
| Automated checks | 6 / 6 | — | `tools/derive_camera.py` refuses to write otherwise |

## 39.1 Why the camera is one manager over a tiny array

Most engines of this era keep exactly one live camera and blend to it. San Andreas is no exception: the
`CCamera` manager is a global (`TheCamera`), and inside it sits `m_aCams`, an array of just **three** `CCam`
objects. At any instant one is the *active* camera (`m_nActiveCam` selects it); a second holds the camera
you are cutting *from* so the engine can interpolate a smooth transition; the third is the debug/free cam.
Every "camera mode" in the game — the chase cam, aiming, cinematic, the two-player split — is not a separate
object but the **same `CCam` running a different mode**, chosen from the 66-entry `eCamMode` enum
([C39.3](03-sixty-six-modes.md)).

That is why this chapter's spine is two struct sizes and one index: size the manager (`0xD78`), size the
camera (`0x238`), and show how `m_nActiveCam` picks one of three. Everything else — FOV, shake, the look
vectors — lives *inside* the `0x238`-byte `CCam`, and the behaviours live in the mode switch.

## 39.2 The cross-tie to C38 was already there

When [C38](../C38-Skybox-And-Clouds/C38-Skybox-And-Clouds.md) decoded the sky, `RenderSkyPolys` opened by
reading `[0xB6F03C]` (the camera matrix) and `[0xB6F038]` (the camera heading) to orient the sky to the
view. Those two addresses are `TheCamera + 0x14` and `TheCamera + 0x10` — the sky renderer was reading this
chapter's object before this chapter existed. `derive_camera.py` asserts that both fall inside
`[0xB6F028, 0xB6F028 + 0xD78)` (`c38_sky_reads_thecamera`), which is a small but real cross-chapter
agreement: two independently-written chapters reference the same global at consistent offsets.

---

## Key takeaways

- All viewing goes through one global manager, `TheCamera` (`CCamera` @`0xB6F028`), which owns a fixed
  array of **three** `CCam` objects at `+0x174`; `m_nActiveCam` (`+0x59`) picks the live one.
- A single `CCam` is **`0x238` bytes** — proven the project's usual way, by the `imul reg,reg,0x238` that
  indexes the array, seen 27 times.
- Each `CCam` runs one of **66** modes (`eCamMode`, `uint16`), which is how three objects produce the game's
  entire repertoire of camera behaviours.

**Continue:** [C39.1 — TheCamera singleton and the three-cam array →](01-the-camera-singleton.md)


## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Key functions:** `CCamera::Init` (0x5bc520), `CCamera::Process` (0x52b730), `TakeControl` (0x50c7c0), `GetGameCamPosition` (0x50ae50)
- **Callers:** **25** `.text` call-sites reach this chapter's functions.
- **Callees:** **22** distinct functions called from within them.
- **Known bugs / gotchas:** —
- **Modding:** —
- **Performance:** —
