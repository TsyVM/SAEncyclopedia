# C39.2 — The CCam object

If `CCamera` is the manager ([C39.1](01-the-camera-singleton.md)), `CCam` is the camera itself — the object
that actually holds a position, a look direction, a field of view, and the mode that decides how all three
move. There are three of them in the manager's `m_aCams` array, and each one is exactly **`0x238` bytes**.

## The size is the array stride — `0x238`, proven 27 ways

`CCam`'s size is not asserted from a header here; it is read from the multiply that steps through the array.
Every access of the form `m_aCams[i]` compiles to `i × sizeof(CCam) + &m_aCams[0]`, and the compiler emitted
that stride as an explicit immediate:

```
imul reg, reg, 0x238        ; sizeof(CCam) = 0x238 = 568 bytes
```

`derive_camera.py` finds this exact instruction **27 times** in the camera code region
(`ccam_stride_0x238`). A stride that shows up in twenty-seven independent indexing sites cannot be a
misread displacement or a one-off constant — it is the object's true size, and it agrees with
`gta-reversed`'s `VALIDATE_SIZE(CCam, 0x238)`. This is the chapter's cleanest ✅, in the same currency as
C29's garage stride and C31's slot strides: *the number the code multiplies by is the size of the thing.*

## What a single CCam holds

A `CCam` is large (568 bytes) because it carries both the camera's current state and the working variables of
whichever mode it is running. From the field layout, the object holds, in broad groups:

| Group | Representative fields | Role |
|---|---|---|
| Mode | `m_nMode` (`eCamMode`, `uint16`) | which of the 66 behaviours this camera runs ([C39.3](03-sixty-six-modes.md)) |
| Optics | `m_fFOV`, `m_fFOVSpeed` | field of view and how fast it eases |
| Placement | `m_vecSource`, `m_vecTargetCoors`, `m_vecUp`, `m_vecFront` | where the camera is and what it looks at |
| Motion state | `m_vecSourceSpeedOverOneFrame`, look-behind / fudge vectors | per-frame smoothing and interpolation scratch |
| Flags | `m_bBelowMinDist`, directly-behind / in-front bits | collision and framing conditions |

The mode field is the switch that gives one object its many behaviours: a `CCam` in `MODE_FOLLOWPED` runs
the on-foot chase logic against `m_vecSource`/`m_vecTargetCoors`, the same object in `MODE_SNIPER` runs the
zoomed aiming logic against `m_fFOV`, and so on. The object does not change size or type between modes — only
`m_nMode` changes, and the per-frame update reads it to pick the code path. (The precise byte offset of each
field beyond `m_nMode` is not individually pinned in this pass and is left as an ⏳ detail; the object's
*size* and its role as the array element are what this page proves.)

## Why one object, many modes, matters for RE

This is the structural reason the camera "feels" like dozens of systems but is really one. When you switch
from driving (`MODE_CAM_ON_A_STRING`) to on-foot (`MODE_FOLLOWPED`) to aiming a sniper
(`MODE_SNIPER`), you are not creating or destroying camera objects — the manager keeps the same
`m_aCams[m_nActiveCam]` and rewrites its `m_nMode`, blending from the interpolate-from slot
([C39.1](01-the-camera-singleton.md)) while the transition completes. Anyone reversing a specific camera
behaviour should therefore look for the **mode dispatch** inside the per-frame `CCam` update, not for a
distinct class per behaviour.

## Key takeaways

- `CCam` is **`0x238` bytes**, proven by the `imul reg,reg,0x238` array stride seen 27 times — the chapter's
  hardest ✅, matching `gta-reversed`'s `VALIDATE_SIZE`.
- One `CCam` carries the camera's optics, placement, motion-smoothing scratch, and a `uint16` `m_nMode`; the
  mode field is what switches its behaviour.
- The game's many "cameras" are one object type running different modes — reverse the mode dispatch, not a
  class hierarchy.

**Continue:** [C39.3 — Sixty-six camera modes →](03-sixty-six-modes.md)
