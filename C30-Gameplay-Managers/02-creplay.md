# C30.2 — CReplay and the 800 KB Buffer

> **The one-sentence version:** the replay recorder writes into a **ring of 8 blocks of exactly 100,000
> bytes** (`0x186A0`) at `0x97FB88` — three unrelated methods walk it with the identical base, stride and
> end, so the 800 KB total is proven three ways — and around it sit a 140-entry ped-pool conversion table
> and a fixed per-ped packet stride that show how the recorder maps the live ped pool into the buffer.

**Subsystem category:** Gameplay features — replay recording
**Depends on:** [C30 hub](C30-Gameplay-Managers.md), [C28](../C28-Class-Catalogue/C28-Class-Catalogue.md)
**RE status:** Documented
**Confidence:** ✅ for the buffer geometry and the conversion-table size · 🟡 for the packet-field meanings ·
🔷 for methods not disassembled here

---

## 1. The buffer: 8 × 100,000 bytes, walked three ways

`CReplay::StreamAllNecessaryCarsAndPeds` (`entry_va 0x0045D4B0`) iterates the replay blocks:

```
0156A8C6  mov  esi, 0x97FB88               ; buffer base
...       (process block)
0156A92D  add  esi, 0x186A0                ; next block = 100,000 bytes
0156A934  cmp  esi, 0xA43088               ; end of buffer
```

Base `0x97FB88`, stride `0x186A0`, end `0xA43088`. `0x186A0` is decimal **100,000** exactly, and the block
count closes:

```
(0xA43088 − 0x97FB88) / 0x186A0 = 0xC3500 / 0x186A0 = 8 blocks  →  800,000 bytes total
```

Two more methods walk the identical extent — `FindFirstFocusCoordinate` (`0x0045D6C0`) and
`IsThisPedUsedInRecording` (`0x0045DDE0`) both use base `0x97FB88`, stride `0x186A0`, end `0xA43088`. Three
independent witnesses to one buffer: the **8-block × 100,000-byte replay buffer** is ✅. (The round decimal
size — 100,000, not a power of two — is itself a small tell that this is an authored capacity constant, not
a computed one.)

## 2. The ped-pool conversion table

A replay must map the *live* ped pool (whose slots change frame to frame) onto stable indices in the
recording. `CReplay::InitialisePedPoolConversionTable` (`entry_va 0x0045EF20`) sets that up:

```
0156B3AE  mov  ecx, 0x8C                   ; 140 entries
0156B3B3  mov  edi, 0x97F838               ; conversion table base
0156B3B8  rep stosd                         ; clear it
...
0156B407  add  ebx, 0x7C4                   ; per-ped packet stride = 1988 bytes
```

The table is **140 dwords (`0x8C`) at `0x97F838`** — and 140 is exactly the game's `CPed` pool size, the
same limit [C28.1](../C28-Class-Catalogue/01-the-class-map-by-subsystem.md)'s ped subsystem hangs on, so the
conversion table has one slot per possible ped. The `add ebx, 0x7C4` steps a per-ped record of **1,988
bytes** through a parallel structure — the replay's per-ped working buffer. `DealWithNewPedPacket`
(`0x0045CEA0`) uses the same `imul …, 0x7C4` to address a ped's packet, confirming the stride.

The **1,988-byte (`0x7C4`) per-ped packet size is not arbitrary** — it matches `sizeof(CCopPed)`, the
largest ped subclass in the hierarchy. The replay system therefore allocates the maximum-possible ped snapshot
size for every slot unconditionally, regardless of whether the recorded ped was a `CPed`, `CPlayerPed`, or
`CCopPed`. This is a straightforward worst-case-size decision: one opaque 1988-byte block per conversion-table
entry, treated as `tReplayPedUpdateBlock` packets by the streaming side rather than direct field access.
The consequence for modders is that any ped subclass larger than 1988 bytes would corrupt replay memory —
a ceiling that is currently not reached by any vanilla class.

## 3. The rest of the class

The remaining methods (🔷) are the record/playback machinery on top of the buffer: the enable/disable gates
(`DisableReplays`, `EnableReplays`, `ShouldStandardCameraBeProcessed`), the per-entity deletion packets that
keep a recording consistent when the world changes (`RecordVehicleDeleted`, `RecordPedDeleted`), the memory
snapshot (`StoreStuffInMem`), and playback (`TriggerPlayback`, `PlayBackThisFrame`). Their packet-field
layouts are 🟡 — visible in the code but not fully traced here. Full list in
[`gameplay_managers.json`](../RE-Data/data/gameplay_managers.json).

---

### Key takeaways

- The replay buffer is **8 blocks × 100,000 bytes (`0x186A0`) at `0x97FB88`** — base, stride and end
  identical across three methods; `(end − base)/stride = 8`, 800 KB total.
- The **ped-pool conversion table is 140 entries at `0x97F838`** — one per `CPed` pool slot — and the
  per-ped replay packet is **1,988 bytes (`0x7C4`)**, the stride confirmed by two methods.
- The round-decimal 100,000-byte block size is an authored capacity constant, not a computed one.

**Next:** [C30.3 — CShopping and the 560-item ledger](03-cshopping.md).
