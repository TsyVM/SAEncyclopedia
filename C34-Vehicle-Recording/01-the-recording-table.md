# C34.1 — The Recording-File Table

> **The one-sentence version:** loaded vehicle-path recordings live in a **475-slot array of 16-byte
> records** at `0x97D880`, and the size needs no assumption because the array's end (`0x97D880 + 475×16`)
> lands exactly on the game's live-count global at `0x97F630` — the next-global proof — while
> `RemoveRecordingFile` reads out the record's three fields: an id, a data pointer, and a status byte.

**Subsystem category:** Vehicles — recording table
**Depends on:** [C34 hub](C34-Vehicle-Recording.md), [C2.1](../C2-CStreaming/01-the-id-space.md) (the
next-global sizing proof)
**RE status:** Documented
**Confidence:** ✅ for the size, stride and record layout · 🔷 for methods not disassembled here

---

## 1. Base, stride and the next-global proof

`CVehicleRecording::HasRecordingFileBeenLoaded` (`entry_va 0x0045A060`) searches the table by id, and its
loop names the base, the stride and the live count:

```
01560EA1  mov  edx, dword ptr [0x97F630]   ; live count of loaded recordings
...
01560EA9  mov  ecx, 0x97D880               ; table base
01560EAC  cmp  dword ptr [ecx], esi        ; record.id == target?
01560EB1  add  ecx, 0x10                    ; next record = 16 bytes
01560EB4  cmp  eax, edx                     ; < count?
```

Base `0x97D880`, stride `0x10` (16 bytes), live count in the global at `0x97F630`. The *capacity* — how many
slots the array physically has — is fixed by where the array ends, and it ends exactly at that count global:

```
0x97D880 + 475 × 0x10 = 0x97D880 + 0x1DB0 = 0x97F630
```

The array's last byte abuts the count global with no gap, so the capacity is **475 records** — the same
"an array is as long as the distance to the next referenced global" argument [C2.1](../C2-CStreaming/01-the-id-space.md)
used to size the streaming-info array. `RemoveRecordingFile` (`0x0045A0A0`) confirms the physical bound from
the other end, walking the array to `0x97F634` (`= 0x97D884 + 475×0x10`, the `+4` field of the slot one past
the end).

## 2. The 16-byte record

`RemoveRecordingFile` reads every field of a record while tearing one down (its `esi` starts at
`0x97D884` = base + 4):

```
0156410D  cmp  dword ptr [esi - 4], ebx     ; +0  recording id  (== target)
01564113  mov  eax, dword ptr [esi]         ; +4  loaded data pointer
0156411B  mov  cl, byte ptr [esi + 8]       ; +0xC status byte (in-use / don't-free guard)
...       call 0x72F430 (free the +4 pointer)
0156412A  mov  dword ptr [esi], 0           ; clear the data pointer
01564131  add  esi, 0x10                     ; next record
```

So each 16-byte record is `{ uint32 id (+0); void* data (+4); … ; uint8 status (+0xC) }`. The `+4` pointer is
the loaded `.rrr` recording data (allocated on request, freed here), the `+0` id is the recording number a
script references, and the `+0xC` status byte guards against freeing a recording still in use.
`CVehicleRecording::ShutDown` (`0x00459400`) walks the same 16-byte stride freeing every slot's `+4` pointer,
independently confirming both the stride and that `+4` is the owned allocation.

## 3. The rest of the loading side

The other loading methods (🔷) complete the request/lifecycle around this table: `RegisterRecordingFile`
(reserve a slot for an id), `RequestRecordingFile` (ask the streamer to load its `.rrr` data — tying to
[C1](../C1-Streaming/C1-Streaming.md)), `Load` (the on-load handler), and `RemoveAllRecordingsThatArentUsed`
(bulk cleanup, walking the same 475-slot table). Full list in
[`vehicle_recording.json`](../RE-Data/data/vehicle_recording.json).

---

### Key takeaways

- The recording-file table is **475 records × 16 bytes at `0x97D880`**, the capacity fixed by the array
  ending exactly on the live-count global at `0x97F630` (next-global proof).
- Each record is `{ id +0, data pointer +4, status byte +0xC }`; the `+4` pointer is the owned `.rrr` data,
  freed by `RemoveRecordingFile` and `ShutDown`.
- The live count of loaded recordings is the global at `0x97F630`, used as the search-loop bound.

**Next:** [C34.2 — The 16-slot playback engine](02-the-playback-engine.md).
