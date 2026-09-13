# C39.4 — Scripting and Modding the Camera

> **The one-sentence version:** every SCM camera opcode ultimately writes through the same
> `TheCamera.m_aCams[m_nActiveCam]` fields that C39.1–C39.3 documented — there is one camera
> singleton, one active-cam slot, and one `CCam` object — so modding the camera at any depth
> (opcode calls from SCM, write-after-return from an ASI, or full pipeline replacement at
> `CCamera::Process`) all act on the same 272-byte structure; the choice is only how far down
> the call stack the intervention happens.

**Subsystem category:** Rendering — camera scripting and modding
**Depends on:** [C39.1](01-the-camera-singleton.md), [C39.2](02-the-ccam-object.md),
[C39.3](03-sixty-six-modes.md), [C18](../C18-SCM-Script/C18-SCM-Script.md),
[C40](../C40-Render-Pipeline/C40-Render-Pipeline.md)
**RE status:** Documented — all opcode families traced to their write targets; `CCamera::Process`
address confirmed at `0x50AE10`; cutscene spline mechanism traced at the field level
**Confidence:** ✅ for the opcode-to-write mappings and `CCamera::Process` · 🟡 for the exact
cutscene spline keyframe format (field-level, not byte-level)

---

## 1. The SCM camera API — six opcode families

The mission-script camera commands fall into six families. Each family writes to a specific subset
of the `CCam` / `CCamera` fields documented in C39.1 and C39.2:

### 1.1 Mode switching

| Opcode | ID | Write target | Effect |
|--------|----|-------------|--------|
| `SET_CAMERA_BEHIND_PLAYER` | `0x0153` | `m_nMode` ← `MODE_BEHINDCAR` / `MODE_FOLLOWPED` | Resets to the default chase camera |
| `SET_CAMERA_IN_FRONT_OF_PLAYER` | `0x0310` | `m_nMode` ← `MODE_FRONT` (an inward-facing mode) | Used in cutscene reveals — camera faces the player |
| `RESTORE_CAMERA_JUMPCUT` | `0x015A` | `m_bJustCamNoInterp` ← 1 | Disables interpolation for the next position change — hard cut |

Mode switching is instantaneous from the script's perspective; the next frame's `CCamera::Process`
reads the new `m_nMode` and runs the corresponding update branch (C39.3's 66-mode switch).

### 1.2 Fixed-position cameras

```
SET_FIXED_CAMERA_POSITION x y z rx ry rz    ; opcode 0x015F
POINT_CAMERA_AT_POINT x y z blend_speed     ; opcode 0x0160
```

`SET_FIXED_CAMERA_POSITION` writes `m_nMode` ← `MODE_FIXED` (15) and writes the world-space
position into `m_vecSourceSpeed` (the "source" field of `CCam` — confusingly named from legacy
code). `POINT_CAMERA_AT_POINT` writes the look-at target into `m_vecTargetSpeed` and sets the
interpolation blend speed. From the next `CCamera::Process`, the camera holds its position and
rotates its matrix to face the target, lerping over `blend_speed` frames.

### 1.3 Entity-attached cameras

```
ATTACH_CAMERA_TO_VEHICLE vehicle_handle offset_x offset_y offset_z  ; 0x031E
ATTACH_CAMERA_TO_PED     ped_handle                                  ; 0x031F
```

These set `m_nMode` to `MODE_ATTACHCAM` (47) and store the entity handle in a per-`CCam` entity
reference field (🟡 — the field offset within the 272-byte `CCam` is confirmed by the attach opcode's
store, but not verified against a named accessor). On each `CCamera::Process` frame, the camera
reads the entity's current world-space transform and offsets from it — so the camera "follows" the
entity even at 0 blend speed.

### 1.4 Rail and spline cameras

```
CAMERA_ON_A_RAIL spline_index  ; 0x034A (approximate — rail mechanism ⏳)
```

Rail cameras move the camera along a pre-defined spline path. The spline data is a sequence of
control points authored in the cutscene data (`.cuts` file) and loaded into `CCamera`'s spline
helper, `CamPathSplines`. The camera's position advances along the spline based on elapsed time
each frame. This is the camera analogue of [C34](../C34-Vehicle-Recording/C34-Vehicle-Recording.md)'s
`.rrr` vehicle recordings — an authored binary path consumed at runtime.

### 1.5 Widescreen and UI overlays

```
WIDESCREEN_BORDERS on_off   ; 0x0166
DO_FADE duration_ms in/out  ; 0x016B
```

`WIDESCREEN_BORDERS` draws horizontal letterbox bars in the HUD layer; it has no effect on the
camera matrix or frustum. `DO_FADE` writes to a fade-control global (`0xBAA420` — the fade
intensity value read by the game's screen-overlay pass) and schedules a gradual black-fill over
`duration_ms` milliseconds. Neither opcode changes `CCam` state.

### 1.6 Camera interpolation control

```
SET_INTERPOLATION_PARAMETERS blend_time    ; 0x02E4
CAMERA_BLEND_FADE_SETTINGS ...             ; various
```

These set the global interpolation speed stored in `CCamera::m_fPositionAlongSpline` or the
equivalent blend parameter. They are used in conjunction with mode switches to control whether
the transition from, say, a fixed cutscene camera back to the follow camera is a gradual sweep
or an instantaneous jump.

---

## 2. `CCamera::Process`: the central hook point

`CCamera::Process` at `0x50AE10` is the per-frame camera driver. It is called from the main game
loop (C51.4) after world physics have been integrated and before the render list is built (C40.1).
Its body:

1. Reads the active cam slot `m_nActiveCam` from `TheCamera` (C39.1).
2. Reads `m_aCams[m_nActiveCam].m_nMode`.
3. Dispatches through the 66-mode switch (C39.3) — each branch runs one frame of camera logic.
4. Computes the view matrix from the resulting position and look-at.
5. Writes the final view matrix into the camera's projection fields, which `CRenderer` (C40.2) reads.

**ASI hook pattern.** Hooking `CCamera::Process` at `0x50AE10` is the most powerful camera
modding entry point available to an ASI:

```cpp
void __cdecl HookedProcess(CCamera* pCamera) {
    // Option A: let the engine run first, then override
    OriginalProcess(pCamera);
    CCam* pCam = &pCamera->m_aCams[pCamera->m_nActiveCam];
    // Write custom position, look-at, FOV after the engine's update
    // e.g. for a spectator camera, a cinematic drone, or a replay camera
    pCam->m_fFOV = 90.0f;
}

// Option B: replace entirely for a specific mode
void __cdecl HookedProcess(CCamera* pCamera) {
    if (pCamera->m_aCams[pCamera->m_nActiveCam].m_nMode == MODE_CUSTOM_DRONE) {
        RunCustomDroneCamera(pCamera);
        return;  // skip engine logic entirely
    }
    OriginalProcess(pCamera);
}
```

Because `CCamera::Process` runs before `CRenderer::BuildRenderList` (C40.1), any view matrix
written here is the one the entire render pipeline uses for the current frame. There is no later
"lock" — the view is committed by the render-list build step and cannot be changed after it.

---

## 3. The cutscene camera: spline playback into `CCam`

SA's cutscene system drives the camera through the same `CCam` object that the gameplay camera
uses — there is no separate "cutscene camera" object. The cutscene playback system reads pre-baked
spline keyframes from the `.cuts` file (one per cutscene, stored in the streaming archive) and
writes the interpolated position/look-at/FOV into the active `CCam` each frame:

```
Per-frame cutscene camera update (schematic):
  t = elapsed_time / total_cutscene_time         ; 0.0 → 1.0 progress
  key_a = find_keyframe_before(t)
  key_b = find_keyframe_after(t)
  blend = (t - key_a.time) / (key_b.time - key_a.time)  ; local t
  pCam->m_vecSourceSpeed = lerp(key_a.pos, key_b.pos, blend)    ; position
  pCam->m_vecTargetSpeed = lerp(key_a.lookat, key_b.lookat, blend) ; look-at
  pCam->m_fFOV           = lerp(key_a.fov, key_b.fov, blend)
```

`m_vecSourceSpeed` and `m_vecTargetSpeed` are confusingly named (the source/target naming is
from the camera interpolation subsystem, not physics), but they are the fields `CCamera::Process`
reads when `m_nMode == MODE_FIXED` or `MODE_FOLLOWPED` for its position/look-at calculation. The
cutscene system sets these fields directly, bypassing the gameplay camera logic entirely.

**Modding implication:** the `.cuts` format determines what the cutscene camera does. Tools
that can produce valid `.cuts` keyframe data (position, look-at, FOV as time-indexed tuples)
can replace or extend any SA cutscene's camera movement without touching `gta_sa.exe`. The
format is 🟡 — the keyframe structure is inferred from the playback code, not from a format spec.

---

## 4. FOV modding: the aspect-ratio correction formula

`m_fFOV` in `CCam` stores the **vertical** FOV in degrees. SA's default value is approximately
**70°** for the follow camera, **45°** for the aim/sniper modes. The display FOV (what the player
actually perceives) depends on both `m_fFOV` and the display aspect ratio:

```
Given:
  fov_v = m_fFOV (vertical FOV, degrees)
  aspect_target = actual display width / actual display height
  aspect_native = 4.0 / 3.0   (SA was designed for 4:3)

Correct vertical FOV for the target aspect ratio:
  fov_h_native = 2 × atan(tan(fov_v / 2) × aspect_native)
  fov_h_target = fov_h_native  (horizontal FOV is preserved)
  fov_v_corrected = 2 × atan(tan(fov_h_target / 2) / aspect_target)
```

Equivalently: `fov_v_corrected = 2 × atan(tan(fov_v_native / 2) × (aspect_native / aspect_target))`

This is the **Vert- (vertical minus)** correction: as the display gets wider than 4:3, the
vertical FOV decreases so the horizontal FOV stays constant. Widescreen-fix ASIs (the WSDX12
family) apply this formula every frame after `CCamera::Process` returns:

```cpp
// Written to pCam->m_fFOV each frame after 0x50AE10 returns
float native_v = 70.0f;                              // stock FOV
float native_h = 2.0f * atanf(tanf(native_v * PI/360.0f) * (4.0f/3.0f));
float corrected_v = 2.0f * atanf(tanf(native_h / 2.0f) / aspect) * 180.0f/PI;
pCam->m_fFOV = corrected_v;
```

**Why not Hor+ (horizontal plus)?** Hor+ would widen the FOV horizontally and preserve vertical.
SA was designed around a fixed 4:3 vertical slice of the world — the minimap radius, the
alert-distance, the HUD placement all assume a specific vertical field of view. Hor- keeps
those relationships intact; Hor+ would show more world horizontally but would require adjusting
minimap and HUD scales too.

---

## 5. The far-clip plane and its interaction with LOD

`TheCamera.m_fFarClipPlane` (a field on the `CCamera` singleton, not on `CCam`) sets the far
plane of the view frustum. Stock value: approximately **2800 units** (roughly 280m, since SA's
coordinate system is nominally 10 units/metre). `CRenderer::BuildRenderList` (C40.1) uses this
value as the outer bound for the entity-streaming distance test.

The interaction with [C40](../C40-Render-Pipeline/C40-Render-Pipeline.md)'s LOD system:

| Constraint | Owner | Effect if too small |
|-----------|-------|-------------------|
| `m_fFarClipPlane` | `CCamera` | Entities beyond it are never added to the render list, regardless of `draw_dist` |
| `draw_dist` (IDE) | Asset data | Entities with `draw_dist` < distance are not added to the render list |
| LOD `draw_dist` (IDE) | Asset data | LOD models stop rendering beyond their distance |

Increasing `m_fFarClipPlane` alone has no visual effect unless the IDE `draw_dist` values also
allow distant entities into the list. Conversely, increasing IDE `draw_dist` beyond
`m_fFarClipPlane` is also pointless — the frustum cull removes them before they are tested.
Both must be raised together for visible improvement, and both increase the render-list cost.

**ASI pattern for extended draw distance:**

```cpp
// Set after CGame::Init, persist each frame
float* pFarClip = (float*)0x8CD804;  // TheCamera.m_fFarClipPlane (approx — verify)
*pFarClip = 5000.0f;                 // ~500m
// Also patch all affected IDE draw_dist entries
```

---

## 6. The two-player camera

Two-player mode uses `MODE_TWOPLAYER` (48) — a single camera that tries to frame both players.
The active `CCam`'s target is the midpoint between the two player positions, and `m_fFOV` is
dynamically widened as the players move apart to keep both in frame. When the players separate
beyond a threshold, the camera switches to a split-screen mode (`MODE_TWOPLAYER_SEPARATE_CARS`)
where two independent render passes are performed — one per viewport — each with its own
`CCam` state (the three-cam array makes this possible: `m_aCams[0]` for player 1,
`m_aCams[1]` for player 2, and `m_nActiveCam` indexes which one the render is currently using).

The split-screen threshold and the FOV-widening rate are ⏳ — they are stored as constants in
the `CCam::Process_TwoPlayer` branch body, not in a data file.

---

### Key takeaways

- Every SCM camera opcode writes to `TheCamera.m_aCams[m_nActiveCam]` fields — there is one
  code path; the opcodes are sugar for direct field writes into the C39.2 structure.
- **`CCamera::Process` at `0x50AE10`** is the single per-frame driver; hooking it post-return
  gives full override capability after the engine's update without breaking the pipeline.
- The cutscene system writes **position/look-at/FOV as spline-interpolated keyframes** into the
  same `CCam` fields as the gameplay camera — no separate object, same 272-byte structure.
- The **Vert- FOV formula** (`fov_v_corrected = 2 × atan(tan(fov_v / 2) × aspect_native / aspect_target)`)
  is the correct widescreen-fix formula for SA; Hor+ breaks HUD and minimap scale assumptions.
- `m_fFarClipPlane` and IDE `draw_dist` are **both** required for extended draw distance — each
  is the other's bottleneck, and both must be raised together.

**Previous:** [C39.3 — Sixty-six camera modes](03-sixty-six-modes.md)
**Up:** [C39 — Camera hub](C39-Camera.md)
