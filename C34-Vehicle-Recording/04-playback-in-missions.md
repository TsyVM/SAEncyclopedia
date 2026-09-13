# C34.4 — Playback in Missions: Trains, the Driving-School Ghost, and SCM Integration

> **The one-sentence version:** `CVehicleRecording` is the physics bypass that makes San Andreas's
> trains run on rails and its driving-school ghost move with perfect fidelity — the vehicle's position
> is overwritten from the sample stream every frame, bypassing the rigid-body integrator entirely,
> which is why the train never derails and the ghost follows the exact route even at variable frame
> rates — and the system is fully script-controlled through four SCM opcodes that give mods complete
> command of route, speed, looping, and sync.

**Subsystem category:** Vehicles — mission integration
**Depends on:** [C34.1](01-the-recording-table.md), [C34.2](02-the-playback-engine.md),
[C34.3](03-the-recording-file-format.md), [C18](../C18-SCM-Script/C18-SCM-Script.md),
[C30.2](../C30-Gameplay-Managers/02-creplay.md)
**RE status:** Documented
**Confidence:** ✅ for the two system distinctions and the four SCM opcodes · 🟡 for the per-frame
physics-bypass mechanism (inferred from the absence of physics callers in the playback path)

---

## 1. The physics bypass: why the train "never derails"

The distinction between a *recorded* vehicle and a *physics-simulated* vehicle is not cosmetic. Every
normal vehicle in San Andreas has its position integrated by `CPhysical::ProcessControl` each frame:
forces are computed, velocity is integrated, the rigid body advances. A vehicle under
`CVehicleRecording` playback **does not use this path** for its position. Instead,
`PlaybackThisRecordingFile` (`0x0045A980`) runs each frame for every active playback slot and
directly writes the interpolated position, heading, and speed from the sample stream into the
vehicle's transform matrix and velocity fields:

```
; inside PlaybackThisRecordingFile, per-frame update (illustrative)
fld   [sample_a + 0x00]   ; x_a
fld   [sample_b + 0x00]   ; x_b
fsub  st(1), st(0)
fmul  [interp_t]          ; lerp factor (0.0 – 1.0)
fadd                      ; lerped X
fstp  [vehicle + MATRIX_X_OFFSET]
; ... same for Y, Z, heading
; speed written to CPhysical.m_vecMoveSpeed directly
```

The interpolation factor `t` is computed from the elapsed time and the sample's `uint16` time-delta
field (C34.3 §3). Linear interpolation between adjacent samples produces smooth motion even at the
24-bytes-per-sample granularity. The vehicle's collision geometry still participates — the train
*can* hit the player, and the player's character registers the contact force — but the train itself
is on rails: no collision impulse can redirect it, no slope can slow it, and it will clip through
geometry that a physics-driven vehicle would stop on. This is a deliberate design choice: scripted
trains must be reliable, not physically correct.

## 2. The train routes

SA's two visible train lines (the freight train loop and the passenger tram) are both driven by
`CVehicleRecording`. The route recordings are authored assets — binary `.rrr` files in `models/gta3.img`
— and the train vehicles are created by a long-running mission script that loops the playback:

```
; schematic of the train-spawning script (from SCM structure)
LOAD_RECORDING_FILE  recording_id_freight
CREATE_CAR FREIGHT 0 0 0 -> $train_handle    ; create at any position
PLAY_RECORDED_CAR $train_handle recording_id_freight
wait_loop:
  IS_RECORDED_CAR_AT_END $train_handle -> $done
  IF $done
    PLAY_RECORDED_CAR $train_handle recording_id_freight   ; restart loop
  END
  WAIT 0
  GOTO wait_loop
```

The `PLAY_RECORDED_CAR`→`IS_RECORDED_CAR_AT_END`→`PLAY_RECORDED_CAR` loop is the looping idiom: when
the recording stream is exhausted, the script restarts playback from the beginning. The vehicle's
position at "end of recording" is the same as at "start of recording" because the recording was
authored as a closed loop, so there is no position discontinuity at the restart.

The train's **apparent spawning position** (`0 0 0` in the `CREATE_CAR` call) does not matter — within
the first frame of `PLAY_RECORDED_CAR`, `PlaybackThisRecordingFile` overwrites the vehicle's transform
with the first sample's position, teleporting it to the correct location before it is ever rendered.

## 3. The driving-school ghost vehicle

The driving-school missions in San Fierro use `CVehicleRecording` for the *ghost vehicle* — the
semi-transparent demonstration car the player is supposed to follow or beat. The ghost's setup:

**Step 1: Create the vehicle.**  
`CREATE_CAR` spawns a normal vehicle of the target model (e.g., a Greenwood sedan). The vehicle
enters the `CVehicle` pool ([C4](../C4-Entities-And-Pools/C4-Entities-And-Pools.md)) and starts
with full physics enabled.

**Step 2: Start playback.**  
`PLAY_RECORDED_CAR $ghost_handle recording_id` immediately overrides the vehicle's position and
heading with the first sample.

**Step 3: Set alpha / render flags.**  
The ghost's semi-transparency is achieved by setting the vehicle entity's render alpha via the SCM
`SET_VEHICLE_ALPHA` opcode (or an equivalent flag on `CVehicle::m_nModelAlpha`). `CVehicleRecording`
does not know about alpha — that is purely a rendering concern. The script handles it separately.

**Step 4: Sync the player's timer.**  
`IS_RECORDED_CAR_AT_END $ghost_handle` fires when the ghost's recording is exhausted — one of the
success conditions for the school challenge is that the player finishes the course within the ghost's
time. If the ghost reaches the end and the player has not, the time-limit condition can fail the
challenge. The script reads the ghost's completion as the authoritative timer.

`GET_POSITION_IN_RECORDED_SCRIPT $ghost_handle → $progress` returns a 0.0–1.0 float representing
how far through the recording the ghost has progressed. The driving-school HUD uses this to show
progress along the course map.

## 4. The four SCM opcodes

All four opcodes operate through the 16-slot playback engine (C34.2):

| Opcode | SA Opcode ID | Arguments | Action |
|--------|-------------|-----------|--------|
| `PLAY_RECORDED_CAR` | `0x04BA` | vehicle handle, recording id | Start playback; loads recording if not in table; claims a free playback slot; overwrites vehicle position immediately |
| `STOP_RECORDED_CAR` | `0x04BB` | vehicle handle | End playback; releases the playback slot; vehicle returns to physics integration |
| `IS_RECORDED_CAR_AT_END` | `0x04BC` | vehicle handle → bool | True when the sample stream is exhausted and the playback slot has wrapped past the last sample |
| `GET_POSITION_IN_RECORDED_SCRIPT` | `0x04BD` | vehicle handle → float | Returns `elapsed_ms / total_recording_ms` as a 0.0–1.0 progress value |

`PLAY_RECORDED_CAR` is the most consequential opcode: it searches the 475-slot table for the requested
recording id (O(475) scan, same as [C34.3 §2](03-the-recording-file-format.md#2-addrecordingfile-the-o475-free-slot-scan)),
then searches the 16-slot playback engine (C34.2) for a free slot (`flag byte == 0`). If no playback
slot is free, the call silently fails — the vehicle remains in its current position, physics-driven.
A mission script that needs to play back 17 recorded paths simultaneously will lose the 17th silently.

`STOP_RECORDED_CAR` releases the playback slot but does **not** evict the recording from the 475-slot
table — the `+0x04` data pointer remains, and the `+0x0C` in-use bit is cleared. The recording
stays loaded until `RemoveRecordingFile` or `RemoveAllRecordingsThatArentUsed` is called (the latter
runs at mission-boundary cleanup).

## 5. The pause/resume mechanism during loading screens

C34.2 identified two mirror-image methods:

- `PausePlaybackRecordedCar` (`0x00459740`) — sets the pause bit in the per-slot flags; `PlaybackThisRecordingFile` skips updating a paused slot's vehicle transform, so the vehicle freezes in world space.
- `UnpausePlaybackRecordedCar` (`0x00459850`) — clears the pause bit; the vehicle resumes from the frame it was frozen on, not from the start of the recording.

These are called during **loading screen transitions** (when the game suspends most gameplay logic)
and at the start of long-running **cutscenes** that reposition the camera away from the recorded
vehicle. The underlying reason is consistency: if `PlaybackThisRecordingFile` continued running
during a loading screen where `CTimer` is frozen, the time accumulator would stay constant and the
interpolation factor would be stale — but the vehicle transform would still be set. Pausing avoids
a potential one-frame position jump when the game resumes.

A mod that spawns its own recorded vehicles must account for these pause calls: if the game pauses
all recorded cars during a cutscene and then your mod assumes playback is live, the vehicle will
be stationary until `UnpausePlaybackRecordedCar` is called.

## 6. CReplay vs CVehicleRecording — a definitive table

| Property | `CReplay` (C30.2) | `CVehicleRecording` (C34) |
|----------|-------------------|--------------------------|
| **Data source** | Live capture ring buffer (800 KB) | Authored binary `.rrr` in IMG archive |
| **Time direction** | Both forward and backward (rewind) | Forward only |
| **Scope** | Entire world state snapshot | Single vehicle's path |
| **Control** | Player-triggered (replay button) | Script-triggered (SCM opcodes) |
| **Physics** | Re-simulates from captures | Overrides physics directly |
| **Persistence** | Transient (overwritten by next capture) | Permanent (authored asset) |
| **Slot limit** | 1 (one replay buffer) | 16 simultaneous |
| **Max length** | ~30 seconds of gameplay | Unbounded (file size only) |
| **Loops** | No | Yes (script restarts `PLAY_RECORDED_CAR`) |

The two systems share no code and no data structures. The only conceptual overlap is that both
involve "replaying" vehicle state — everything else differs.

## 7. What modders control

| Goal | Technique |
|------|-----------|
| New train route | Author a `.rrr` binary, add to `gta3.img`, assign a new recording ID, use the `PLAY_RECORDED_CAR` → `IS_RECORDED_CAR_AT_END` → restart loop |
| Ghost vehicle for a custom race | `CREATE_CAR` + `PLAY_RECORDED_CAR` + `SET_VEHICLE_ALPHA 128` |
| Multi-vehicle convoy | Up to 16 simultaneous `PLAY_RECORDED_CAR` calls on distinct vehicle handles |
| Custom looping bus route | Closed-loop `.rrr` file + restart loop in a persistent script thread |
| Extend beyond 16 simultaneous | Patch the 16-slot arrays (car-pointer `0x97D840`, flag-byte `0x97D6F0`) and all `cmp …, 0x10` bounds; the chained boundary at `0x97D880` is the address immediately after the extended array, so the recording table must also shift |

The last item is the hardest: because the three arrays pack end-to-end in BSS (C34.3 §4), shifting
the playback array forces the recording table and the live-count global to shift too. A robust
extension requires patching every reference to `0x97D880` and `0x97F630` in the binary — a
significant but well-defined task.

---

### Key takeaways

- `PlaybackThisRecordingFile` **directly writes position/heading/speed** into the vehicle's transform
  each frame, completely bypassing `CPhysical` integration — the vehicle cannot be physically
  deflected and collision impulses do not redirect it.
- Both trains and the driving-school ghost use this system: the train loops via `IS_RECORDED_CAR_AT_END`
  → restart; the ghost uses `GET_POSITION_IN_RECORDED_SCRIPT` for HUD progress.
- Four SCM opcodes (play, stop, at-end query, progress query) form the complete control API; silent
  failure on slot exhaustion means scripts should not assume more than **16 simultaneous** recordings.
- `PausePlaybackRecordedCar` / `UnpausePlaybackRecordedCar` are called during loading screens and
  cutscenes; mod scripts must account for them.
- `CReplay` and `CVehicleRecording` share only the word "recording" — their data, mechanisms, and
  use cases are entirely distinct.

**Previous:** [C34.3 — The recording file format and data loading](03-the-recording-file-format.md)
**Up:** [C34 — CVehicleRecording hub](C34-Vehicle-Recording.md)
