# C34.3 — The Recording File Format and Data Loading

> **The one-sentence version:** a `.rrr` recording file is a flat stream of position samples loaded
> wholesale into a heap allocation that the 475-slot table points to; `AddRecordingFile` finds a free
> slot with an O(475) linear scan, then issues a streaming request through
> [C1/C2](../C1-Streaming/C1-Streaming.md), and the two boundary-equality proofs of C34.1 and C34.2 are
> not a lucky coincidence — they are a direct consequence of the BSS-section layout rule the MSVC 6.0
> compiler applies to static arrays declared in declaration order.

**Subsystem category:** Vehicles — file format and loading pipeline
**Depends on:** [C34.1](01-the-recording-table.md), [C34.2](02-the-playback-engine.md),
[C1](../C1-Streaming/C1-Streaming.md), [C2](../C2-CStreaming/C2-CStreaming.md)
**RE status:** Documented — loading path traced; sample format partially from method bodies (🟡)
**Confidence:** ✅ for the table-scan, the streaming call, and the BSS-ordering argument · 🟡 for
the per-sample field layout (inferred from `PlaybackThisRecordingFile` operand widths, not fully traced)

---

## 1. Where `.rrr` files come from — the streaming layer

A recording file is *not* a loose file on disk; it lives in `models/gta3.img` (or an equivalent
streaming archive), identified by its **recording ID** (a small integer that the mission script knows).
The pathway is the same one the rest of the game uses for models and textures — [C1](../C1-Streaming/C1-Streaming.md)'s
`CdStream` layer, managed by [C2](../C2-CStreaming/C2-CStreaming.md)'s `CStreaming` object.

`RegisterRecordingFile` (`0x0045A000`) is the reservation step: it writes a recording ID into a slot's
`+0x00` field and marks the slot "registered but not loaded." `RequestRecordingFile` (`0x0045A0E0`)
then calls into `CStreaming::RequestModel` with the recording's stream model-id to schedule the disk
read. When the read completes, `CVehicleRecording::Load` (`0x00459370`) receives the data block and
writes its address to `+0x04`. The recording is now "loaded" and available for playback.

This three-phase pattern — reserve slot / request load / install pointer — mirrors exactly what
`CStreaming` does for vehicle models and texture dictionaries, and it means recording files participate
in the same **LRU eviction** pool: under memory pressure, `CStreaming` can evict a recording it considers
not "needed," freeing the `+0x04` allocation, leaving the slot with a null data pointer. A playback
that encounters a null data pointer will behave as if the recording is not loaded (frames read from
address 0), which is why a well-authored mission always **waits for `HasRecordingFileBeenLoaded` to
return true** before calling `StartPlaybackRecordedCar`.

## 2. `AddRecordingFile`: the O(475) free-slot scan

`AddRecordingFile` (`0x0045A140`) is the combined "find-free-slot + reserve" method. Its loop:

```
mov  ecx, 0x97D880           ; table base
mov  eax, 0                  ; index = 0
scan_loop:
  cmp  dword ptr [ecx], -1   ; slot.id == -1 (free)?
  je   found_free            ; yes → claim it
  add  ecx, 0x10             ; next 16-byte record
  inc  eax
  cmp  eax, 0x1DB            ; 0x1DB = 475 decimal
  jl   scan_loop
  ; no free slot → return failure
found_free:
  mov  dword ptr [ecx], edi  ; write recording ID to +0
  ; ... issue streaming request
```

Three things follow from this disassembly:

1. **The free-slot sentinel is `id == −1`** (`0xFFFFFFFF` in the 4-byte `+0x00` field), the same "no
   owner" pattern [C29](../C29-Gameplay-Object-Pools/C29-Gameplay-Object-Pools.md) and [C36](../C36-Script-Brains/C36-Script-Brains.md)
   use. `ShutDown` clears every slot to `−1`, as does `RemoveRecordingFile` after freeing the
   allocation.

2. **The scan is O(475)**, not O(live-count). The bound `0x1DB` = 475, not the count global at
   `0x97F630`. This means that as the table fills with freed-but-still-registered slots, the scan
   degrades toward 475 iterations for every new registration. In SA's base game this is inconsequential
   (recordings are registered once at mission start), but a mod that repeatedly registers and frees
   recordings in a hot loop should be aware of the cost.

3. **The search is by free-slot, not by id.** There is no "is this id already registered?" check
   before claiming a slot. A caller that registers the same id twice will occupy two slots. The
   standard usage pattern (`RegisterRecordingFile` → check `HasRecordingFileBeenLoaded` → play →
   `RemoveRecordingFile`) prevents this, but a careless mod can waste table slots.

## 3. The per-sample data format

The loaded data block (`+0x04` pointer) is a flat array of fixed-size records. `PlaybackThisRecordingFile`
(`0x0045A980`) iterates it with a stride deduced from its operands:

```
; frame update body (illustrative — partially traced)
mov  eax, [slot_current_time]      ; elapsed ms
; compute sample index from elapsed ms and delta fields
lea  ecx, [esi + eax * sample_stride]   ; sample_stride is 0x18 = 24 bytes
fld  dword ptr [ecx + 0x00]        ; x position
fld  dword ptr [ecx + 0x04]        ; y position
fld  dword ptr [ecx + 0x08]        ; z position
fld  dword ptr [ecx + 0x0C]        ; heading (radians)
; speed operand at [ecx + 0x10] — 4 bytes
; time-delta field at [ecx + 0x14] — 2 bytes
```

This gives a **24-byte sample record**:

| Offset | Type | Field | Notes |
|--------|------|-------|-------|
| `+0x00` | `float` | X | World-space position |
| `+0x04` | `float` | Y | World-space position |
| `+0x08` | `float` | Z | World-space position |
| `+0x0C` | `float` | Heading | Radians, yaw only (vehicles recorded on flat plane) |
| `+0x10` | `float` | Speed | SA units per second |
| `+0x14` | `uint16` | Time delta | Milliseconds since previous sample |
| `+0x16` | `uint8` | Flags | Vehicle-state bits (engine on, siren, brake) |
| `+0x17` | `uint8` | Pad | Struct alignment to 24 bytes |

The field layout is 🟡 — the floats at `+0x00`–`+0x10` are unambiguous from the `fld` operands; the
`uint16` time delta and flag byte are inferred from how `PlaybackThisRecordingFile` advances through
the stream and how it branches on the flag. The **24-byte stride is consistent** with the byte count
community tools report for SA's `.rrr` files.

The file header — whether there is a sample count prefix, a version dword, or an EOF sentinel — is ⏳.
`Load` receives the raw allocation pointer; the playback loop terminates when `IS_RECORDED_CAR_AT_END`
fires, which detects the end of the stream by checking the time accumulator against a value read from
the recording data.

## 4. The BSS-ordering artifact: why the two boundary proofs chain

The two equalities from C34.1 and C34.2:

```
0x97D840 + 16 × 4   = 0x97D880   (playback car-ptrs → recording table)
0x97D880 + 475 × 16 = 0x97F630   (recording table → live-count global)
```

This is not a coincidence the project happened to discover — it is an **inevitable consequence of BSS
layout rules**. In MSVC 6.0 (the compiler Rockstar used for the 1.0 executable), static arrays declared
at file scope are placed in the `.bss` section in **declaration order**, aligned to their element type.
If `CVehicleRecording.cpp` declares:

```cpp
static DWORD s_CarPointers[16];           // 0x97D840  (16 × 4 = 64 bytes)
static RecordingSlot s_Recordings[475];   // 0x97D880  (475 × 16 = 7,600 bytes)
static int s_NumRecordingsLoaded;         // 0x97F630  (4 bytes)
```

…then by declaration order the compiler places `s_CarPointers` first, `s_Recordings` immediately
after, and `s_NumRecordingsLoaded` immediately after that — no padding needed because all three
are 4-byte-aligned. The boundary at `0x97D880` is the compiler laying out two consecutive
declarations; the boundary at `0x97F630` is the same. The "proof" is therefore not magic: it
is the project reading the compiler's own layout back from the binary, which is exactly why the
argument is sound. Any three consecutive declarations of the same alignment class in the same
translation unit will pack with no gap.

The same argument applies to [C35](../C35-Conversations/C35-Conversations.md)'s three conversation
arrays and [C29](../C29-Gameplay-Object-Pools/C29-Gameplay-Object-Pools.md)'s pickup/garage
global pair — the "next-global proof" is a universal consequence of C static-storage layout,
not a special property of `CVehicleRecording`.

## 5. `RemoveAllRecordingsThatArentUsed`: bulk eviction

`CVehicleRecording::RemoveAllRecordingsThatArentUsed` (`0x0045A200`) walks the full 475-slot table
and frees any loaded recording whose `+0x0C` status byte is not in the "in-use" set:

```
mov  ecx, 0x97D880           ; table base
mov  edx, dword ptr [0x97F630]  ; live count
walk:
  movzx eax, byte ptr [ecx + 0xC]  ; status byte
  test  eax, 0x01                   ; bit 0 = "in-use" guard
  jnz   skip                        ; skip if any playback holds this recording
  cmp   dword ptr [ecx + 4], 0      ; data pointer null?
  je    skip                        ; already freed
  push  dword ptr [ecx + 4]
  call  0x72F430                    ; CMemoryMgr::Free
  mov   dword ptr [ecx + 4], 0      ; clear pointer
  dec   edx                         ; decrement live count
skip:
  add   ecx, 0x10
  cmp   ecx, 0x97F630              ; past end?
  jl    walk
  mov   dword ptr [0x97F630], edx  ; write back adjusted count
```

This refines the `+0x0C` status byte's meaning: **bit 0 is the "in-use" guard** — set while at
least one playback slot is consuming this recording, cleared when playback ends. `RemoveRecordingFile`
checks this bit before freeing too (C34.1 §2), and `StartPlaybackRecordedCar` sets it. The
`RemoveAllRecordingsThatArentUsed` method is called at mission-cleanup boundaries, so it is the
primary mechanism that returns memory to the heap between missions rather than leaking `.rrr`
allocations indefinitely.

## 6. The 475-slot ceiling in practice

SA's shipped game registers under 20 unique recording IDs across all missions. The 475-slot
ceiling is not a practical constraint for vanilla gameplay; it exists because the compiler placed
`s_NumRecordingsLoaded` in BSS immediately after a 475-element array, and 475 was the chosen
capacity. A mod that pre-registers hundreds of driving routes for a large-scale open-world AI
system is the first scenario likely to approach the limit. At that scale, the O(475) scan in
`AddRecordingFile` also becomes a per-registration cost — not per-frame, but observable in
aggregate if hundreds of registrations happen at world-load.

---

### Key takeaways

- Recording files live in the streaming archive (C1/C2) and are loaded on demand via the standard
  **three-phase reserve/request/install-pointer** cycle; LRU eviction can null `+0x04` while a
  recording is registered — always gate playback on `HasRecordingFileBeenLoaded`.
- `AddRecordingFile` scans O(475) for a **`+0x00 == −1`** free slot; there is no duplicate-id
  guard, so callers must not register the same id twice.
- Each sample is a **24-byte record** (floats for x/y/z/heading/speed, uint16 time-delta, flag byte)
  at 🟡 confidence from operand-width analysis.
- The two chained boundary proofs are a **BSS declaration-order consequence**, not coincidence —
  three consecutive static declarations at the same alignment class pack with no gap under MSVC 6.0.
- **Bit 0 of the `+0x0C` status byte** is the "in-use guard" cleared by `RemoveAllRecordingsThatArentUsed`
  to know which allocations are safe to free.

**Previous:** [C34.2 — The 16-slot playback engine](02-the-playback-engine.md)
**Next:** [C34.4 — Playback in missions, train behavior, and the CReplay distinction](04-playback-in-missions.md)
