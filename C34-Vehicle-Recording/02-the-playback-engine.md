# C34.2 — The 16-slot Playback Engine

> **The one-sentence version:** at most **16 vehicles** can be playing back a recording at once, and the
> number is not assumed — the per-slot car-pointer array at `0x97D840` is 16 dwords long because it ends
> exactly where the recording table begins (`0x97D880`), the same next-global proof that sized the table
> itself, so three consecutive globals pack with no gap.

**Subsystem category:** Vehicles — playback state
**Depends on:** [C34.1](01-the-recording-table.md), [C34 hub](C34-Vehicle-Recording.md)
**RE status:** Documented
**Confidence:** ✅ for the 16-slot count and the two parallel arrays · 🟡 for the per-slot minor fields
· 🔷 for methods not disassembled here

---

## 1. Parallel per-slot arrays

`CVehicleRecording::IsPlaybackGoingOnForCar` (`entry_va 0x004594C0`) checks whether a given car is being
driven by a recording. The compiler unrolled its loop, so the two per-slot arrays are visible slot by slot:

```
mov  dl, byte ptr [eax + 0x97D6F0]        ; slot 0 : active flag
cmp  dword ptr [eax*4 + 0x97D840], ecx     ; slot 0 : car pointer == this car?
mov  dl, byte ptr [eax + 0x97D6F1]        ; slot 1 : active flag
cmp  dword ptr [eax*4 + 0x97D844], ecx     ; slot 1 : car pointer
mov  dl, byte ptr [eax + 0x97D6F2]        ; slot 2 …
cmp  dword ptr [eax*4 + 0x97D848], ecx
… (unrolled)
```

Two parallel arrays index the same slot number: a **byte** active-flag array at `0x97D6F0`
(`base + slot`) and a **dword** car-pointer array at `0x97D840` (`base + slot×4`). A slot is a live playback
when its flag is set and its car pointer matches.

## 2. The next-global proof: 16 slots

How many slots? The car-pointer array starts at `0x97D840`, and the very next structure is the recording
table from [C34.1](01-the-recording-table.md) at `0x97D880`. The car-pointer array ends exactly there:

```
0x97D840 + 16 × 4 = 0x97D840 + 0x40 = 0x97D880   (= the recording table base)
```

So the car-pointer array is **16 dwords** — **16 simultaneous playback slots** — fixed by the same
next-global argument that sized the recording table, with no separate assumption. The two proofs even chain:
`0x97D840` (playback car-ptrs) → `0x97D880` (recording table) → `0x97F630` (recording count global) are three
consecutive globals that pack end-to-end. The flag array at `0x97D6F0` sits `0x150` (336) bytes before the
car-pointer array, leaving room for the other per-slot playback bookkeeping (current frame time, playback
speed, the pause state of §3) between them.

## 3. Pause / resume state

`PausePlaybackRecordedCar` (`0x00459740`) and `UnpausePlaybackRecordedCar` (`0x00459850`) toggle a slot's
paused state and are near-mirror images (272 bytes each), saving and restoring the vehicle's motion so a
scripted convoy can be frozen and released. `StartPlaybackRecordedCar` (`0x0045A980`, the most-called at 3
callers) is the entry point: it looks up the recording id in the [C34.1](01-the-recording-table.md) table,
finds a free playback slot in these arrays, binds the car pointer at `0x97D840[slot]` and sets the flag at
`0x97D6F0[slot]`. Their per-frame field reads/writes (current time, interpolation) are 🟡 — visible but not
fully traced here. Full list in [`vehicle_recording.json`](../RE-Data/data/vehicle_recording.json).

---

### Key takeaways

- Playback runs in **16 slots**, fixed by the car-pointer array at `0x97D840` ending exactly at the
  recording table base `0x97D880` (`16 × 4 = 0x40`).
- Two parallel per-slot arrays: **active-flag bytes at `0x97D6F0`**, **car pointers at `0x97D840`**; a live
  slot has flag set and car pointer matching.
- `0x97D840 → 0x97D880 → 0x97F630` are three consecutive globals packing end-to-end — the next-global proof
  applied twice, chained.
- Pause/resume are mirror-image methods; `StartPlaybackRecordedCar` binds a recording id to a free slot.

**Next:** [C34.3 — The recording file format and data loading](03-the-recording-file-format.md).
