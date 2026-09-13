# C28.3 — CStreaming, and Re-deriving C2 from the Other End

> **The one-sentence version:** [C2](../C2-CStreaming/C2-CStreaming.md) measured the streaming-info array
> from the data side — base `0x8E4CC0`, 20-byte stride, 26,316 entries — and a single fill loop inside
> `CStreaming::ClearFlagForAll` reproduces every one of those numbers from the *code* side without knowing
> them, which is the strongest confirmation this project accepts; the same class also yields the load-status
> field offset, an 8-slot IMG archive table, and the intrusive doubly-linked render list.

**Subsystem category:** Streaming — in-memory structure
**Depends on:** [C28.1](01-the-class-map-by-subsystem.md), [C2](../C2-CStreaming/C2-CStreaming.md) (the ID
space and streaming-info record), [C1](../C1-Streaming/C1-Streaming.md) (IMG / `CdStream`)
**RE status:** Documented
**Confidence:** ✅ for every byte-fact below · 🔷 for the methods listed but not individually disassembled

---

## 1. The cross-check that matters

[C2.1](../C2-CStreaming/01-the-id-space.md) established the streaming-info array structurally: it begins at
`0x008E4CC0`, each record is 20 bytes, and there are **26,316** of them, because
`0x8E4CC0 + 26316 × 20 = 0x009654B0` lands exactly on the next referenced global. That derivation used the
array's *bounds* — where it starts and what sits after it.

`CStreaming::ClearFlagForAll` (`entry_va 0x00407A40`) clears one flag bit across every streaming-info
record, and its loop is built from the same three numbers, reached independently:

```
01566490  mov  cl, byte ptr [esp + 4]      ; flag mask arg
01566494  not  cl                          ; clear these bits
01566496  mov  eax, 0x8E4CC6               ; first record's flags byte  (base + 6)
015664A0  and  byte ptr [eax], cl          ; clear in this record
015664A2  add  eax, 0x14                   ; advance one record = 20 bytes
015664A5  cmp  eax, 0x9654B6               ; past the last record's flags?
015664AA  jl   0x15664A0
```

Read out the constants and the arithmetic closes on C2's numbers to the byte:

- The loop starts at `0x8E4CC6`, which is `0x8E4CC0 + 6` — so the **flags byte is at record offset +6**,
  exactly where [C2.2](../C2-CStreaming/02-streaming-info-record.md) placed it.
- It advances by `0x14` = **20 bytes** per iteration — C2's record stride.
- It stops at `0x9654B6`, and `0x9654B6 − 0x8E4CC6 = 0x807F0 = 526,320 = 26,316 × 20` — so the loop covers
  **26,316 records**, C2's exact count, and its end bound is C2's array end (`0x9654B0`) plus the same +6
  flags offset.

Three numbers — base, stride, count — a *data*-side derivation in C2 and a *code*-side derivation here,
with no shared intermediate. That is the same "two independent subsystems agree" standard the project uses
everywhere; here it is used to confirm a runtime data structure, and it confirms C2 completely.

## 2. The load-status field

`CStreaming::WeAreTryingToPhaseVehicleOut` (`entry_va 0x00407F80`) indexes the array by model id and reads
two fields:

```
01567B64  lea  eax, [eax + eax*4]          ; id * 5
01567B67  shl  eax, 2                       ; * 4  ->  id * 20  (the C2 stride again)
01567B6A  cmp  byte ptr [eax + 0x8E4CD0], 1 ; status byte, ==1 ?
...
01567B73  cmp  word ptr [eax + 0x8E4CC0], 0 ; nextIndex   (record + 0)
01567B7D  cmp  word ptr [eax + 0x8E4CC2], 0 ; prevIndex   (record + 2)
```

The `id * 20` addressing is C2's stride a third time. The status byte read at `0x8E4CD0 = base + 0x10`
gives the **load-status field at record offset +0x10**, and the `== 1` test is the "loaded" state — the
same field `HasVehicleUpgradeLoaded` (`0x00407820`) and `HasSpecialCharLoaded` (`0x00407F00`) test against
`1`. The two `word` reads at base+0 and base+2 are the `nextIndex` / `prevIndex` intrusive-list links C2
documented at those offsets, seen here from the streamer's own use of them.

## 3. The IMG archive table

`CStreaming::AddImageToList` (`entry_va 0x00407610`) opens by scanning the table of loaded IMG archives for
a free slot:

```
01567B93  mov  eax, 0x8E48D8               ; archive table base
01567B98  cmp  byte ptr [eax], 0           ; slot in use?
01567B9D  add  eax, 0x30                   ; next entry = 48 bytes
01567BA1  cmp  eax, 0x8E4A58               ; end of table
```

`(0x8E4A58 − 0x8E48D8) / 0x30 = 0x180 / 0x30 = 8` — the engine holds **8 IMG-archive slots, each 48 bytes**,
at `0x8E48D8`. This is the in-memory registry behind [C1](../C1-Streaming/C1-Streaming.md)'s IMG/`CdStream`
layer: eight archives (`gta3.img`, `gta_int.img`, `player.img`, and the streamed-script/cutscene archives)
can be mounted at once, each slot carrying its name and `CdStream` handle in 48 bytes.

## 4. The intrusive render list

`CStreaming::RenderEntity` (`0x004096D0`) and `RemoveEntity` (`0x00409710`) manipulate a doubly-linked list
whose head sentinel is a single static node:

```
0156D758  mov  ecx, dword ptr [0x8E48A0]   ; list head sentinel
...        mov  [entity+4] / [entity+8]     ; prev / next link surgery
```

Each entity carries its list links at **+4 (prev)** and **+8 (next)**, and `0x8E48A0` is the head. The
methods do textbook intrusive-list unlink-and-relink (splice the node out of its current position, splice
it in beside the head) — this is the per-frame "entities to render / rebuild" list the streamer maintains,
not a data table, so it has a sentinel and links rather than a base/stride/count.

## 5. The rest of the class

The other 26 methods (🔷, C27-inherited) fill in the streamer's request pipeline and its content-specific
loaders, and their names are coherent with the structure above: the request path (`RequestFile`,
`LoadRequestedModels`, `RemoveAllUnusedModels`, `RemoveLoadedZoneModel`, `IsVeryBusy` — which reads a
request-count global and compares it to 5), vehicle streaming (`StreamOneNewCar`,
`AddToLoadedVehiclesList`, `RemoveCarModel`, `PossiblyStreamCarOutAfterCreation`,
`RequestVehicleUpgrade`), ped streaming (`StreamPedsIntoRandomSlots`, `RemoveInappropriatePedModels`,
`LoadInitialPeds`), the emergency-service defaults (`GetDefaultCopModel`, `GetDefaultMedicModel`,
`GetDefaultFiremanModel`, `StreamCopModels`, `StreamFireEngineAndFireman`, `DisableCopBikes`), and
lifecycle (`ReInit`, `Shutdown`). Full list in [`class_catalogue.json`](../RE-Data/data/class_catalogue.json).

---

### Key takeaways

- `CStreaming::ClearFlagForAll`'s fill loop independently reproduces [C2](../C2-CStreaming/C2-CStreaming.md)'s
  streaming-info layout to the byte — base `0x8E4CC0`, flags at +6, 20-byte stride, **26,316** records —
  from code, with no knowledge of C2's data-side derivation. Strongest confirmation the project accepts.
- The record's **load-status byte is at +0x10** (`==1` = loaded), and the `nextIndex`/`prevIndex` links at
  +0/+2 appear in the streamer's own addressing.
- The engine mounts **8 IMG archives, 48 bytes each, at `0x8E48D8`** — the in-memory registry under C1's
  IMG layer.
- The render list is an intrusive doubly-linked list (links at entity +4/+8, head sentinel `0x8E48A0`),
  not a table.

**Next:** [C28.4 — CPathFind and the 28-byte node](04-cpathfind-and-the-28-byte-node.md).
